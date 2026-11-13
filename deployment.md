# Deploy vLLM on Obito

A deployment runs one or more named workload bindings in a single cluster.
Each binding uses an immutable workload template version and its own inputs,
placement, and replica settings.

## Choose a region and prepare the template

Sign in to your Obito profile. Run `obito region list` and
`obito accelerator list`. Choose a region with deployment support for the
accelerators required by all bindings. A region can include several clusters;
one eligible cluster must support the full combination. Catalog support
describes configuration and does not guarantee capacity. Use either
`--region-name` or `--region` with an exact ID.

Read `obito workload-template guide` and publish its complete `vllm/main.star`
as template `vllm`. It needs no assets. Find the ID of a ready catalog image
built from upstream vLLM with `obito image list`. The image must support the
model and accelerator you selected. In the commands below, set `REGION_NAME`,
`VLLM_IMAGE_ID`, and later `LMCACHE_IMAGE_ID` and `VLLM_LMCACHE_IMAGE_ID`
to values from your account. The examples assume newly published templates
at version 1. Check with `obito workload-template get` and adjust the pins
if your published versions differ.

## Create a fixed-size deployment

```sh
obito deployment create vllm-fixed --region-name "$REGION_NAME" \
  --workload 'from=vllm@v1,name=engine,replicas=2' \
  --param "engine.image=$VLLM_IMAGE_ID" \
  --param 'engine.model=Qwen/Qwen2.5-0.5B-Instruct'
```

- Repeat `--workload` for 1–256 bindings. Each `name` is unique in the
  deployment and starts with a lowercase letter. Use lowercase letters,
  digits, and hyphens, with no trailing hyphen and at most 63 characters.
- `from` accepts a template name or canonical `workloadtemplate-…` ID.
  Use `@vN` to pin a specific version. Omitting it selects the latest ready
  version at creation and pins that result. A bare source token inside
  the workload argument also works, for example `vllm@v1,name=engine`.
- Pass `--param binding.input=value` after declaring that binding with
  `--workload`. Quote arguments containing shell characters or spaces.
  Inspect inputs with `obito workload-template get vllm --version 1`.
  Required inputs must be supplied. Optional inputs use the template's
  defaults. Integer and boolean inputs are parsed according to the
  schema. Use `true` or `false` for a boolean.
- Placement defaults to replicated, with one replica if unspecified.
  Fixed counts allow 0–1,000,000. Zero keeps the binding and its network
  identity while it runs no groups. An explicit `placement` field cannot
  be combined with `replicas`, even when set to `replicated`.
  The `scaling_policy` and `create_once` fields are reserved and inactive.

Save the returned deployment ID as `DEPLOYMENT_ID`. Use
`obito deployment get "$DEPLOYMENT_ID"` to track per-binding readiness and
desired/applied revisions. Creation accepts desired configuration and
starts reconciliation. Wait for `RUNNING` and check the exposed URL.

For an inference request, use the API exposure URL from `deployment get`
as `INFERENCE_URL`. Set `OBITO_API_KEY` to an account API key, created with
`obito auth api-key create inference` if you need one.

```sh
curl --fail-with-body "${INFERENCE_URL%/}/v1/chat/completions" \
  -H "Authorization: Bearer $OBITO_API_KEY" \
  -H 'Content-Type: application/json' \
  --data '{"model":"Qwen/Qwen2.5-0.5B-Instruct","messages":[{"role":"user","content":"Say hello."}],"max_tokens":32}'
```

## Compose multiple replicated workloads

Reuse the same source package to run two model bindings. This example
exposes one endpoint. For two public endpoints in a deployment, declare
distinct exposure names and path prefixes in the templates.

```sh
obito deployment create vllm-pair --region-name "$REGION_NAME" \
  --workload 'from=vllm@v1,name=primary,replicas=2' \
  --param "primary.image=$VLLM_IMAGE_ID" \
  --workload 'from=vllm@v1,name=secondary,replicas=1' \
  --param "secondary.image=$VLLM_IMAGE_ID" \
  --param 'secondary.model=Qwen/Qwen2.5-1.5B-Instruct' \
  --param 'secondary.public=false'
```

## Pair replicated vLLM with node-local LMCache

Use the same accelerator selector in both packages so the mapping selects
nodes for both components. Use a pair of digest-backed catalog images with
the same compatible LMCache version, at least 0.5.6, and `cuda-python`. The
client image also needs upstream vLLM and the LMCache multiprocess connector.
Match client and server versions and enable isolated IPC on both ends.
The node mapping must support DaemonSets and NVIDIA management CDI/NRI
access for the workload namespace. Use ordinary CUDA allocations with
this connector.

Save as `lmcache/main.star`. No assets are required.

```starlark
load("obito", "resource")

gpu = resource.accelerator(name="gpu-nvidia-h100-sxm-80gb")

def build(ctx):
    return workload(containers=[container(
        name="lmcache-server",
        image=ctx.inputs.image,
        command=["lmcache", "server"],
        args=[
            "--host", "0.0.0.0", "--port", 6555,
            "--supported-transfer-mode", "lmcache_driven",
            "--isolated-ipc", "--l1-size-gb", ctx.inputs.cache_gb,
            "--eviction-policy", "LRU",
        ],
        ports={"cache": 6555},
        resources={
            "cpu_millis": 4000,
            "memory_bytes": (ctx.inputs.cache_gb + 8) * 1024 * 1024 * 1024,
        },
    )])

define_template(
    inputs={
        "image": param.string(resource="image", required=True),
        "cache_gb": param.int(default=16, min=1, max=4096),
    },
    build=build,
)
```

Save as `vllm-lmcache/main.star`. No assets are required.

```starlark
load("obito", "resource", "expose", "probe")

gpu = resource.accelerator(name="gpu-nvidia-h100-sxm-80gb")

def build(ctx):
    cache = ctx.workloads(name="cache").endpoint("cache")
    transfer = (
        '{"kv_connector":"LMCacheMPConnector",'
        + '"kv_connector_module_path":"lmcache.integration.vllm.lmcache_mp_connector",'
        + '"kv_role":"kv_both","kv_connector_extra_config":{'
        + '"lmcache.mp.host":"tcp://' + cache.host + '",'
        + '"lmcache.mp.port":' + cache.port + ','
        + '"lmcache.mp.mp_transfer_mode":"lmcache_driven",'
        + '"lmcache.mp.isolated_ipc":true}}'
    )
    return workload(
        containers=[container(
            name="vllm",
            image=ctx.inputs.image,
            command=["vllm", "serve"],
            args=[
                ctx.inputs.model, "--host", "0.0.0.0", "--port", 8000,
                "--max-model-len", 4096, "--kv-transfer-config", transfer,
            ],
            ports={"http": 8000},
            readiness=probe.http(port="http", path="/health"),
            resources={
                "accelerator": gpu.request(count=1),
                "cpu_millis": 4000,
                "memory_bytes": 32 * 1024 * 1024 * 1024,
            },
        )],
        exposures=[expose.http(name="api", port="http")],
    )

define_template(
    inputs={
        "image": param.string(resource="image", required=True),
        "model": param.string(default="Qwen/Qwen2.5-0.5B-Instruct"),
    },
    build=build,
)
```

```sh
obito workload-template create ./lmcache --name lmcache
obito workload-template create ./vllm-lmcache --name vllm-lmcache
obito deployment create vllm-node-cache --region-name "$REGION_NAME" \
  --workload 'from=lmcache@v1,name=cache,placement=node' \
  --param "cache.image=$LMCACHE_IMAGE_ID" \
  --param 'cache.cache_gb=16' \
  --workload 'from=vllm-lmcache@v1,name=engine,replicas=2' \
  --param "engine.image=$VLLM_LMCACHE_IMAGE_ID"
```

Node placement runs one cache pod on every eligible node, including nodes
without vLLM pods. The cache uses the accelerator mapping for node selection
and requests CPU and memory. It reserves zero accelerators. The cache Service
routes only to a cache pod on the client node. Confirm eligible cache nodes
cover all vLLM nodes. A stable Service name gives no startup ordering, and
local traffic has no remote fallback during cache startup or restart. The
binding name `cache` and the named port `cache` are used by the vLLM source.
Adjust memory sizing to the node and verify both workloads converge before
testing inference.

Node placement is create-only. A node binding cannot use replica counts,
autoscaling, peer groups, group expressions, or positive accelerator
requests. The node workload stays active until deployment deletion. Public
CLI output shows workload state. Admin output can also show nodes ready.

## Create a deployment with queue-driven autoscaling

The first guide's `vllm` template scrapes vLLM's `/metrics`, records
`sum(vllm:num_requests_waiting)`, and targets four waiting requests per
replica. The sum reduces per-replica series to one queue-depth value.
Use a compatible vLLM image that emits this metric and a region configured
for readiness, Prometheus scraping and recording, and the autoscaling
metrics adapter. The mapping needs workload-lws 0.3.0 or compatible support
for these features.

```sh
obito deployment create vllm-autoscaled --region-name "$REGION_NAME" \
  --workload 'from=vllm@v1,name=engine,enable-autoscale=true,min-replicas=2,max-replicas=8' \
  --param "engine.image=$VLLM_IMAGE_ID"
```

- Autoscaling defaults off. Enabling it requires at least one target in the
  template and both bounds, each within 1–1,000,000 with minimum at most
  maximum. Omit `replicas` for an autoscaled binding. The maximum does not
  reserve capacity.
- A `per_replica` target uses the metric as a total and compares its average
  per replica to the threshold. For example, a queue of 24 requests and a
  target of 4 per replica imply approximately six replicas, subject to
  bounds and HPA behavior. A `value` target compares the measured metric
  directly to a single threshold. The highest recommendation wins across
  targets. The controller applies its normal tolerance and stabilization.
- Raw queries default to the binding's own scraped series. Explicit
  `sources` select sibling bindings and still require a finite, nonnegative
  result with one series for each target. Missing or stale samples are
  unavailable to the autoscaler. A real zero can drive scale-down toward
  the minimum.
- To diagnose a workload stuck at its minimum, check its readiness and
  configured bounds with `deployment get`. Verify the image serves
  `/metrics` on the declared port, reports the expected metric under load,
  and that the PromQL reduces it to one series. Allow time for both
  scraping and recording. If the metric is healthy and the requested
  replicas remain unready, check capacity, quota, and placement.
  [vLLM production metrics](https://docs.vllm.ai/en/stable/usage/metrics/)

## Update or scale

An update changes versions and replaces input overrides for the named
bindings. Supply required inputs again. Omitted optional inputs take
defaults, and omitted versions keep the current pin. Unlisted bindings keep
their pins and inputs. Obito rebuilds the deployment against the resulting
configuration. Binding membership, placement, and hardware shape are fixed.
For a revision conflict, re-read the deployment before retrying.

After publishing version 2 of the same template, update the fixed example:

```sh
obito deployment update "$DEPLOYMENT_ID" \
  --workload 'name=engine,from=vllm@v2' \
  --param "engine.image=$VLLM_IMAGE_ID"
obito deployment scale "$DEPLOYMENT_ID" --replicas engine=3
obito deployment get "$DEPLOYMENT_ID"
```

Scaling sets an absolute group count, and repeatable `--replicas` flags can
change several fixed bindings together. Set a fixed binding to zero to stop
it while retaining its identity. For an autoscaled deployment ID saved as
`AUTOSCALED_ID`, change only the settings you want to replace:

```sh
obito deployment scale "$AUTOSCALED_ID" --workload 'name=engine,min-replicas=3'
obito deployment scale "$AUTOSCALED_ID" \
  --workload 'name=engine,enable-autoscale=false,replicas=2'
```

Disabling autoscaling without a count keeps the current target as the fixed
count. Disable it before updating to a template version with no targets.
Scale requests need an applied launch. Use `deployment get` to observe
convergence. Delete with `obito deployment delete "$DEPLOYMENT_ID"` when
you are done.

## Common creation failures

- **No eligible cluster.** Check that one cluster in the selected region
  supports every binding's accelerator mapping and public routing if used.
  Check quota and available capacity for the requested GPU groups.
- **Missing or invalid input.** Inspect the exact template version. Check
  the binding name in each `--param`, the order of flags, and whether the
  supplied image is a ready catalog image accessible to your account.
- **Missing sibling or port.** Match the binding names in `ctx.workloads`
  and declare the named port on a regular container.
- **Unready or unavailable endpoint.** Check the workload state and
  revisions, confirm the server listens on the port and passes its
  readiness probe, and use the published HTTPS URL with a valid account
  API key. For a node-local dependency, check that both pods can run on
  the same node.
