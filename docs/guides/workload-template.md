# Write and publish a vLLM workload template

A workload template is a versioned source package. Its `build(ctx)` function
describes one workload. A deployment supplies the workload's binding name,
inputs, placement, and replica count.

## Publish your first template

Run `obito accelerator list` and choose a GPU accelerator supported in your
target region. The example uses an H100 SXM 80 GB catalog name. Adjust the
accelerator, GPU count, and memory for your model and region. Save this file as
`vllm/main.star`.

```starlark
load("obito", "resource", "expose", "probe", "telemetry", "autoscale")

gpu = resource.accelerator(name="gpu-nvidia-h100-sxm-80gb")

def build(ctx):
    queued = telemetry.metric(
        name="queued_requests",
        expr="sum(vllm:num_requests_waiting)",
        interval="30s",
    )
    routes = []
    if ctx.inputs.public:
        routes.append(expose.http(name="api", port="http"))
    return workload(
        labels={"app.kubernetes.io/name": "vllm"},
        containers=[container(
            name="vllm",
            image=ctx.inputs.image,
            command=["vllm", "serve"],
            args=[
                ctx.inputs.model,
                "--host", "0.0.0.0",
                "--port", 8000,
                "--max-model-len", ctx.inputs.max_model_len,
                "--tensor-parallel-size", ctx.inputs.gpus,
            ],
            resources={
                "accelerator": gpu.request(count=ctx.inputs.gpus),
                "cpu_millis": 4000,
                "memory_bytes": 32 * 1024 * 1024 * 1024,
            },
            ports={"http": 8000},
            readiness=probe.http(port="http", path="/health"),
            scrape=telemetry.prometheus(port="http"),
        )],
        exposures=routes,
        scaling=[autoscale.target(metric=queued, per_replica=4)],
    )

define_template(
    inputs={
        "image": param.string(resource="image", required=True,
                              description="Obito image ID for an upstream vLLM image"),
        "model": param.string(default="Qwen/Qwen2.5-0.5B-Instruct"),
        "gpus": param.int(default=1, min=1, max=8),
        "max_model_len": param.int(default=4096, min=128),
        "public": param.bool(default=True),
    },
    build=build,
)
```

Publish and inspect the template using your Obito profile.

```sh
obito workload-template create ./vllm --name vllm
obito workload-template get vllm --version 1
```

Use the returned version if the example name already has published versions.
The image ID is a deployment input. Use an account-accessible, ready image
built from upstream vLLM. See `obito deployment guide` for deployment commands.

## Source packages and versioning

- Put the entry point in `main.star`. Call
  `define_template(build=build, inputs={}, assets=[])` exactly once.
  The builder takes one context and returns one `workload(...)`.
- `load("obito", ...)` exports `resource`, `config`, `expose`, `probe`,
  `telemetry`, and `autoscale`. The constructors `define_template`, `workload`,
  `container`, `peer_group`, and `param` are predeclared. Use
  `load("helpers/flags.star", "serve_flags")` for a package-local module.
  Load paths are relative to the package root. Import cycles, imports of
  `main.star`, and loads outside the package fail.
- The upload includes `.star` files and assets named in `assets`.
  Use canonical relative paths and regular files. Symlinks and path traversal
  are rejected. The package limit is 128 files and 1 MiB total. Declaration
  and build share a Starlark execution budget of 1,000,000 steps. Evaluation
  has no ambient filesystem, network, or environment access.
- `obito workload-template list` and `get <name-or-id> [--version N]` inspect
  versions. Omitting the version selects the latest ready version. Versions
  are immutable. Download to a new directory, edit locally, and publish
  against the version you started from.

```sh
obito workload-template download vllm ./vllm-edit --version 1
obito workload-template update vllm ./vllm-edit --base-version 1
obito workload-template edit vllm
```

`edit` opens an editor and publishes changes as a new version. If publication
reports a version conflict, download the latest source and reconcile your
changes before retrying. Creating a template validates its declarations;
deployment creation evaluates the builder with the actual inputs and resources.

## Inputs and resources

| Declaration | Accepted arguments |
| --- | --- |
| `param.string` | `description`, `required`, `default`, `resource` |
| `param.int` | `description`, `required`, `default`, `min`, `max` |
| `param.bool` | `description`, `required`, `default` |

All arguments above are optional. Inputs are optional by default. Declare up
to 1024 inputs with names made of letters, digits, and underscores, starting
with a letter or underscore. Supply required inputs at deployment time.
`required=True` cannot be combined with a default. Unknown inputs are rejected.
An omitted optional input without a default is `None` in `ctx.inputs`.
Integer bounds are inclusive and apply to defaults and overrides. Access
values as `ctx.inputs.<name>`, and use a conditional to turn a boolean into
a flag or string. A plain string is passed through. A string declared with
`resource="image"` accepts a canonical Obito image ID and becomes an image
handle in the builder.

Declare exactly one `resource.accelerator(id=...)` or
`resource.accelerator(name=...)` before the builder runs. This selects the
hardware mapping, including for CPU-only templates. A declaration alone
reserves no devices. In `build(ctx)`, use
`resources={"accelerator": gpu.request(count=1)}` to request devices.
The request count is a nonnegative integer and is also available as
`.count` on the request object. `cpu_millis` and `memory_bytes` are
nonnegative integers. For example, 4000 millicores requests four CPU cores.

A container image can be a nonempty registry reference string, a catalog
handle from `resource.image(id=...)` or `resource.image(name=...)`
declared before build, or a handle from an image input. Use exactly one
selector argument per declaration. Catalog handles expose the resolved
`.id`. Accelerator handles also expose `.type`, `.name`, `.vendor`, `.model`,
and `.memory_gb` when the catalog supplies memory. Image handles expose `.ref`
and `.digest`. Read these fields inside the builder. Catalog image handles pin
a resolved digest. Image IDs and accelerator names must exist in the target
environment.

## Workloads, containers, and groups

`workload(containers, init_containers=[], labels={}, peer_group=..., exposures=[], scaling=[])`
describes the leader pod, optional peers, routing, and metric targets.
Supply at least one regular container. Kubernetes label keys and values
apply. Obito's `aries.dev/` and LeaderWorkerSet ownership labels are reserved.

`container(name, image, command=[], args=[], env={}, resources={}, ports={}, files={}, readiness=..., scrape=...)`
describes one container. Use a DNS label for the name and unique names within
each pod, including its init containers. Commands are argv lists. Args accept
an argv list or a flag map, whose keys start with `-`. Each map entry emits a
flag followed by its value, and a `None` value emits the flag alone. Use a
list for positional values, repeated flags, or values whose order matters.
Argv and environment values support strings, integers, and runtime references.
Use explicit string values for booleans passed to a process. Commands,
normalized args, env, ports, and files each have a 1024-entry limit.
`ports` maps names to integers from 1 to 65535. Names referenced by a Service,
probe, or scrape must match a port on the applicable container.

Use `init_containers` for tasks that must finish before a pod starts serving.
For example, an init container can check files baked into the vLLM image.
Init containers accept the same basic container fields. Readiness and scrape
configuration belong on regular containers.

`peer_group(count, containers, init_containers=[])` adds 1–1024 peer pods
to each replica, alongside the leader. A replica of a group with two peers
has three pods. Each pod's accelerator demand is the larger of the sum of
its regular container requests and its largest init container request.
Deployment demand sums leader and peer pod demand and multiplies by the
replica count.

### Run vLLM across two pods

Save this as a separate `vllm-distributed/main.star`. It runs one leader
and one headless peer per replica, with one GPU per pod and two pipeline
stages. Use an upstream vLLM image that supports multi-node multiprocessing
and a model compatible with pipeline parallelism. The pods need to reach
each other over the worker network. No assets are required.

```starlark
load("obito", "resource", "probe")

gpu = resource.accelerator(name="gpu-nvidia-h100-sxm-80gb")

def build(ctx):
    image = ctx.inputs.image
    check = container(name="check-vllm", image=image, command=["vllm", "--version"])
    common_args = [
        ctx.inputs.model,
        "--distributed-executor-backend", "mp",
        "--pipeline-parallel-size", 2,
        "--nnodes", ctx.group.size,
        "--node-rank", ctx.group.index,
        "--master-addr", ctx.group.leader.host,
        "--max-model-len", 4096,
    ]
    resources = {"accelerator": gpu.request(count=1), "cpu_millis": 4000,
                 "memory_bytes": 32 * 1024 * 1024 * 1024}
    env = {"VLLM_HOST_IP": ctx.local.ip}
    return workload(
        init_containers=[check],
        containers=[container(
            name="leader", image=image, command=["vllm", "serve"],
            args=common_args + ["--host", "0.0.0.0", "--port", 8000],
            env=env, resources=resources, ports={"http": 8000},
            readiness=probe.http(port="http", path="/health"),
        )],
        peer_group=peer_group(count=1, init_containers=[check], containers=[container(
            name="worker", image=image, command=["vllm", "serve"],
            args=common_args + ["--headless"], env=env, resources=resources,
        )]),
    )

define_template(
    inputs={
        "image": param.string(resource="image", required=True),
        "model": param.string(default="Qwen/Qwen2.5-0.5B-Instruct"),
    },
    build=build,
)
```

Each group reserves two GPUs, one for the leader and one for the peer.
Use replicated placement and a chart that supports `ctx.local.ip`.
[vLLM parallelism and scaling](https://docs.vllm.ai/en/stable/serving/parallelism_scaling/)

## Build context and networking

| Expression | Value |
| --- | --- |
| `ctx.inputs.<name>` | Typed input or `None`, as declared by the template |
| `ctx.local.ip` | Current pod IP |
| `ctx.group.index` | Member index, with zero for the leader |
| `ctx.group.size` | Total group size, including the leader |
| `ctx.group.leader.host` | Group leader hostname |
| `ctx.workloads(name="engine")` | A binding in the same deployment |
| Binding `.id`, `.name`, `.template_id`, `.template_name`, `.version` | Workload and template identity |
| Binding `.replicas`, `.inputs.<name>` | Build-time replica metadata and normalized scalar inputs |
| Binding `.endpoint("http").host`, `.port`, `.address` | Stable Service hostname, port, and host:port |
| `spec.endpoint("http")` | Same endpoint handle on the `workload(...)` object returned by the current builder |

Use runtime references directly in argv or env, or concatenate them with
`+`. For example, vLLM can use `env={"VLLM_HOST_IP": ctx.local.ip}`
when configured for distributed serving. Use a chart mapping that supports
local IP references (workload-lws 0.3.1 or later). Runtime references cannot
be converted with `str()` or used as build-time numbers. Group references
are unavailable for node placement.

All binding metadata is available before any builder runs. Forward references,
cycles, and references to bindings with zero replicas are valid. Missing binding
names or named ports fail deployment creation. Service addresses give no
startup ordering. Sibling processes must handle startup and restart gaps.
Replica-only scaling leaves build-time replica metadata in the compiled
configuration at its last build value. A node binding's stored replica metadata
does not represent a count of nodes.

Use `expose.http(name, port, path_prefix="/")` in `workload(exposures=[...])`
to route a public HTTPS path to a named regular-container port. The deployment
supports up to 16 exposures. Use unique exposure names and normalized,
unique path prefixes per deployment. The target region needs a configured
public gateway. Clients send an account API key as a Bearer token. Use the
internal endpoint when communication stays inside the deployment.

## Config files

Construct both file handles inside `build(ctx)`. `config.file(path)` mounts a
declared UTF-8 asset verbatim.
`config.template(path, vars={...})` renders a declared asset using only the
explicit variable map and the supported template operations. In
`container(files={...})`, map absolute, normalized file paths to the
resulting handles. Overlapping mounts are rejected. Use `{{ var "key" }}`
to read a value and `{{ var "key" | json }}` to quote it for JSON or YAML.
The renderer rejects missing variables, loops, definitions, and arbitrary
functions. Sibling endpoint references can be rendered. Local IP and group
references are only available as process arguments and environment values.
Rendered assets are bounded to 1 MiB per binding.

For a file-backed vLLM configuration, add `serve.yaml` to `assets`, mount it
with `files={"/etc/vllm/serve.yaml": config.file("serve.yaml")}`, and use
`args=["--config", "/etc/vllm/serve.yaml"]` with `command=["vllm", "serve"]`.
Save this asset next to `main.star` and match the declared port and GPU count.

```yaml
model: Qwen/Qwen2.5-0.5B-Instruct
host: 0.0.0.0
port: 8000
max-model-len: 4096
tensor-parallel-size: 1
```

For an input-driven asset, replace the model line with
`model: {{ var "model" | json }}` and use
`config.template("serve.yaml", vars={"model": ctx.inputs.model})`.

## Readiness probes

Set one `readiness` probe per regular container. A failed probe keeps the
container out of normal Service traffic and the group unready. It does not
restart the process.

| Builtin | Arguments and behavior |
| --- | --- |
| `probe.http` | `port`, optional `path` (default `/`), `scheme` (`HTTP` by default or `HTTPS`), `headers` (string pairs), `timing`. Use `path="/health"` for the vLLM HTTP server. |
| `probe.tcp` | `port`, optional `timing`. Checks that a TCP listener accepts connections. |
| `probe.grpc` | `port`, optional `service` and `timing`. Requires a server implementing the standard gRPC health service. An HTTP-only vLLM endpoint cannot answer it. |
| `probe.exec` | `command` (nonempty string argv), optional `timing`. Runs inside the container and checks its exit status. |

Network probes use a named port on the same container. `probe.timing(...)`
accepts `initial_delay_seconds=0`, `period_seconds=10`, `timeout_seconds=1`,
`success_threshold=1`, and `failure_threshold=3` by default. All are positive
32-bit integers except the initial delay, which can be zero.

## Metrics and autoscaling

- Set `scrape=telemetry.prometheus(port="http", path="/metrics")` on a
  regular container. The path defaults to `/metrics`. Obito scrapes every
  15 seconds. The named port must exist on that container.
- Inside `build`, use `telemetry.metric(name, expr, interval="60s", sources=...)`.
  Use a unique name of up to 128 characters, starting with a letter or
  underscore and continuing with letters, digits, or underscores.
  `expr` is PromQL or a composition of PromQL strings and local metric
  handles using `+`. Raw series default to the current binding. Pass
  `sources=[ctx.workloads(name="other-engine")]` to scope raw series to
  explicit bindings in the same deployment. The sources list accepts 1–256
  handles. Obito applies ownership selectors. Expressions cannot supply
  those selectors themselves.
- Use `interval` from `1s` to `24h`. Records on the same interval run in
  dependency order. Different intervals follow Prometheus lookback semantics.
  The template supports up to 128 attached records and 64 KiB of literal
  PromQL per expression. Only records referenced by scaling targets and
  their dependencies are installed for autoscaled bindings.
- Use `autoscale.target(metric, per_replica=...)` or
  `autoscale.target(metric, value=...)` in `workload(scaling=[...])`. Pass a
  local metric handle and exactly one positive integer or decimal string
  target. `per_replica` compares an average per replica. `value` compares
  the total metric directly. Up to 16 targets are supported. The autoscaler
  takes the largest recommendation across targets.
- Produce exactly one finite, nonnegative result per scaling target.
  For vLLM, `sum(vllm:num_requests_waiting)` aggregates waiting requests
  across the selected replicas. A missing, stale, or multidimensional result
  is unavailable for scaling. Declaring a target leaves scaling disabled
  until the deployment enables it and supplies bounds.

See `obito deployment guide` to create fixed, node-local, and autoscaled
vLLM deployments.
