# Cache-aware llm-d routing for the H100 and 2 × H200 vLLM nodes

**Status:** implementation proposal  
**Date:** 2026-10-01 (updated 2026-10-15)  
**Target:** existing gateway + one H100 host + one dual-H200 host, using Podman and Nginx without Kubernetes.

## 1. Objective and recommendation

Introduce llm-d's **Endpoint Picker (EPP)** and **Envoy** *behind* the existing API gateway, preserving the current public interface, authentication, quotas and other gateway logic. EPP should select a vLLM endpoint using **prefix-cache affinity and load**, not simple round robin. Each vLLM process retains its **own local KV cache**; llm-d's router tracks or estimates where useful prefixes reside, but does **not** automatically transfer or share KV tensors.

**Recommended rollout:** keep all three existing vLLM replicas independently reachable (H100, H200-A, H200-B); use Envoy + EPP with file-based discovery and **the newer optimized-baseline prefix-affinity filter + token-load scorer**; benchmark against the current policy; optionally enable precise cache indexing through vLLM KV events. Do not introduce Kubernetes, shared KV storage or prefill/decode disaggregation for this first deployment. See §10 for the observed 70/30/0 traffic skew and migration details.

**Important prerequisite:** cache-aware balancing across both hosts assumes that both endpoints serve the **same model identity, weights, tokenizer, chat template and compatible prefix-cache hash semantics**. If H100 serves 27B and H200 serves a larger model, use separate named model pools and model-aware gateway routing. Do not silently substitute different models for an API request.

## 2. Architecture

```text
Copilot / Claude Code / Cursor / portal agents
                    |
                    v
      existing gateway / Nginx / API policy
                    |
                    v
             Envoy listener
                    |
          ext_proc gRPC request
                    v
        llm-d Endpoint Picker (EPP)
          |                 |
          |     endpoint inventory from
          |     local endpoints.yaml
          |
          +---- chooses destination according to:
                 - model/endpoint eligibility and health
                 - prefix-cache affinity
                 - queued / running requests, load
                 - saturation rules and hardware capacity
                    |
        Envoy forwards HTTP/SSE to selected endpoint
                    |
          +---------+----------+
          |                    |
          v                    v
   H100 / vLLM:8000     2 × H200 / vLLM:8000
   local KV cache A     local KV cache B
```

On the gateway run:
- Existing gateway and Nginx: unchanged from the client's perspective; forward only the model/pool that needs balancing to Envoy.
- **Envoy:** the data-plane proxy. The llm-d integration uses Envoy's `ext_proc` extension and an original-destination upstream derived from the EPP decision. Use the reference configuration from the chosen llm-d release rather than an ordinary Nginx `upstream` as a substitute.
- **EPP:** the scheduling process, its configuration file, endpoint inventory and Prometheus metrics; it may also host the approximate prefix index.
- Optionally, for precise routing: the KV indexer/token producer configured in the EPP deployment, subscribing to each vLLM worker's KV events, plus exact render/tokenization support.

On the GPU hosts, keep the existing Podman-managed vLLM instances, with their own VRAM, KV blocks and (if enabled) per-host CPU KV offload. Set up private, firewall-restricted connectivity from Envoy/EPP to the nodes.

### Without Kubernetes

The official llm-d **no-Kubernetes** path runs the same Envoy/EPP routing components as ordinary processes or containers and uses a YAML endpoint file through its file-discovery plugin. `watchFile: true` permits inventory changes without restarting EPP when the file is rewritten atomically. EPP *policy/configuration* changes still require restarting EPP. Kubernetes-only CRDs, automatic service discovery and lifecycle management are not included.

Use the **same tagged llm-d release** for the no-Kubernetes guide, EPP binary, Envoy configuration and example YAML. Do not mix the `dev` configuration schema with a different stable release.

## 3. How cache-aware routing works

### Approximate prefix affinity (phase 1)

1. EPP sees the incoming request and approximates the prompt's token blocks.
2. It hashes its prefix blocks and consults an **in-memory historical index** of which endpoint previously received those blocks.
3. It scores eligible endpoints using both estimated prefix overlap and scheduling/load criteria.
4. Envoy forwards to the selected vLLM server.
5. EPP updates its *estimated* prefix placement after the routing decision.

This mode needs **no vLLM KV-event publishing, ZMQ port, sidecar or shared KV storage**. Its index can be wrong after evictions, restarts, long periods of load or differences in chat-template/tokenizer behaviour. It also cannot guarantee that a request produces a cache hit: only the vLLM engine determines actual block reuse.

For the **new optimized baseline**, configure `approx-prefix-cache-producer`, `inflight-load-producer`, `prefix-cache-affinity-filter`, and `token-load-scorer` rather than the former weighted `prefix-cache-scorer`/queue/KV/LRU blend. The reference scheduling profile includes only the filter and scorer: do **not** add a `max-score-picker` just because older profiles did. Start from the pinned release's optimized-baseline and no-Kubernetes samples; their current examples differ. Tune prefill throughput, the affinity release policy and any prefix-matching limit against long (up to 262k-token) agent conversations. If the index tracks only a short prefix, it cannot fully distinguish long sessions with identical beginnings. See §10.

**Load must be part of the decision:** on a saturated H200, it may be faster to recompute the prompt on H100 than to wait in the H200 queue. Conversely, preserving a large cached agent conversation can save significant prefill work when the preferred endpoint has capacity.

### Precise prefix affinity (optional phase 2)

For more accurate decisions, vLLM emits events for newly stored, removed and cleared KV blocks. EPP subscribes to the per-worker ZMQ event streams and maintains an index mapping cached blocks to their current owner. A token producer must reproduce the model's prompt rendering and tokenization, including chat templates; the precise prefix producer combines those tokens with the cache index. The `prefix-cache-scorer` then uses actual rather than inferred residency.

**Precise does not mean free or infallible:** processing events and tokenization add resources and network traffic; the event stream/index can lag and requires reconnect/resync handling. Precise routing is especially relevant when cache evictions are frequent, prompts are long, offload tiers complicate residency, or the model uses a more complex attention/cache topology. Check compatibility with the selected vLLM, model and llm-d release before enabling it.

### Not KV sharing

Routing to a node because it owns a prefix avoids recomputation **on that node**. Sending the same conversation to another node normally requires a new prefill there. CPU offload on one server remains local unless an independent distributed-transfer mechanism is configured. llm-d has additional P2P/disaggregated-cache projects, but those are **not part of this proposal**.

## 4. What changes in vLLM?

| Item | Phase 1: approximate | Phase 2: precise |
| --- | --- | --- |
| Automatic prefix caching (APC) | **Enable / verify** | **Enable / verify** |
| Existing per-node GPU KV cache / FP8 KV | Keep | Keep |
| CPU KV offload, if currently configured | Keep, verify metrics and compatibility | Keep, verify index's tier semantics |
| `--max-model-len`, `--max-num-seqs`, `--gpu-memory-utilization` | Tune per machine | Tune per machine |
| vLLM OpenAI-compatible HTTP + streaming | Keep | Keep |
| vLLM health and Prometheus metrics | Expose on private network | Expose on private network |
| KV events / ZMQ publisher | **Not required** | **Required** |
| Exact render/tokenization support | Not required | Required |
| Shared KV store / KV migration | Not required | Not required |

### 4.1 Automatic prefix caching

Verify APC is enabled on **both** replicas. If your vLLM release does not have it enabled by default, add:

```bash
--enable-prefix-caching
```

Confirm the exact CLI option and defaults with `vllm serve --help` for your pinned image. APC is vLLM's own cache-reuse mechanism; llm-d does not enable APC on its behalf. If your configuration changes the prefix hash algorithm, model architecture, cache block sizing, tokenizer, LoRA or template, do not assume cross-replica block-key equivalence.

Continue to use your FP8 KV-cache setting if already tested for the deployed model. FP8 KV reduces local cache-memory demand but does not make cache contents accessible to another server.

### 4.2 The H200 allocation is a separate decision

- **Tensor parallelism 2:** one vLLM endpoint spanning both H200s; together with H100 there are **two** schedulable endpoints.
- **One replica per H200:** two separately addressable H200 vLLM endpoints; together with H100 there are **three** schedulable endpoints. Each H200 has its own KV cache and its own request scheduler.
- The dual-H200 host has greater aggregate memory and different throughput/latency from H100. EPP should not treat both machines as equal-capacity servers. Measure and adjust each replica's concurrency and routing thresholds.
- If the GPUs run different models, route by explicit model/pool instead of putting them into one interchangeable pool.

### 4.3 Enabling KV events, only for precise routing

Use vLLM's **version-matched** KV-events configuration. Recent releases expose `KVEventsConfig` with `enable_kv_cache_events`, `publisher: "zmq"`, `endpoint` (for example `tcp://*:5557`), `replay_endpoint` where supported, `topic` and queue limits. The exact supported JSON structure and CLI/config-file spelling **must be checked against your deployed vLLM version** before applying a command. Do not add a ZMQ publisher for the initial approximate deployment.

For precise mode:
1. Turn on KV events and the ZMQ publisher separately on **each** vLLM instance.
2. Bind and advertise distinct, reachable private IP:port endpoints. If multiple vLLM replicas share one host, do not attempt to bind both to the same host port.
3. Restrict ZMQ event/replay ports to the EPP gateway, not the public internet.
4. Configure the EPP precise-prefix-cache producer, KV indexer, topic/subscriptions and the token producer that uses *exact* vLLM rendering; make the scorer reference the precise producer explicitly.
5. Verify the index populates after prefills, loses evicted blocks, recovers from a worker restart and remains correct after a model reload.

Follow the linked llm-d precise-routing deployment guide for the selected release's complete, compatible event publisher and EPP configuration.

## 5. Deployment plan

### Phase 0: baseline and prerequisites

- Pin current vLLM image digest/version, model revision, tokenizer/template, existing Podman flags, 262k model context (if deployed), FP8 KV setting, CPU-offload setting and effective concurrency limits.
- Record current baseline under representative 20–30-user agent traffic: requests/s, generated tokens/s, first-token latency (p50/p95/p99), time per output token, queue times, prefix-cache hits and GPU/CPU memory.
- Confirm both GPU hosts can be reached from the gateway over private networking and that the gateway has CPU/RAM capacity for Envoy + EPP.
- Check that long-lived SSE/OpenAI-compatible streaming, timeouts, request bodies, authorization, `/v1/chat/completions` and error responses survive each proxy hop.
- Verify models are genuinely equivalent if both will share a pool.

### Phase 1: deploy the router without changing public traffic

1. Obtain the official **no-Kubernetes** example for a *pinned llm-d release*. Use its reference EPP configuration, Envoy `ext_proc` settings and file-discovery schema.
2. Run EPP and Envoy in separate, host-managed Podman containers on the gateway; use Quadlet/systemd for startup and restart policies.
3. Create `endpoints.yaml` listing the actual private addresses, ports, model grouping and any labels/metadata supported by the pinned release. Enable file watching if required.
4. Configure EPP's optimized-baseline approximate prefix producer, in-flight load producer, prefix-affinity filter and token-load scorer as in §10; retain working file discovery and metrics extraction.
5. Expose Envoy on a **private test port**. Do not immediately replace the production Nginx upstream.
6. Probe two backends, force one unavailable, test draining and restart behaviour, and check Envoy/EPP/vLLM metrics.

### Phase 2: controlled traffic shift

- Route a small subset of real workloads via the gateway to Envoy; retain the existing direct vLLM route for instant rollback.
- Compare round robin, simple least-load and cache-affinity + load-aware policies using the same workload where possible.
- Run a realistic multi-turn coding-agent experiment: successive long prompts in the same conversation should tend to return to the endpoint that cached the earlier turns *unless* it is saturated.
- Measure actual **vLLM APC hit/miss metrics**, not just EPP's estimated affinity. Watch for prefix evictions caused by many unrelated conversations.
- Simulate H100 outage, H200 outage and EPP/Envoy restart; document what happens to requests already in progress. Routing to a new endpoint does not automatically migrate an active stream or its cache.
- Adjust the gateway's eligible endpoint pool based on model identity and endpoint capacities.

### Phase 3: precise KV index if justified

Only proceed if measurements show approximate routing is materially misrouting or wasting prefill. Enable KV events, indexer and token-producer components, validate eviction correctness and compare TTFT, aggregate throughput, cache hit rate, extra router CPU/RAM, event volume and operational complexity against Phase 2.

## 6. Security, availability and operation

- Preserve one public gateway and its existing authentication/authorization layer. Keep Envoy, EPP metrics, vLLM endpoints, render/tokenization endpoints and ZMQ private.
- Configure TLS or a properly secured internal segment for gateway-to-worker traffic, as required by organizational policy. Do not assume llm-d adds authentication for you.
- Use minimum permissions for Podman/systemd; host-level package installation, firewall changes and system-managed units may require administrators, while rootless Podman containers often do not.
- Manage service inventory and container configs with Ansible, reusing `ansible_vllm_gpu_node_deployment.md` in this repository. Never commit API keys, passwords or sensitive endpoint data.
- One EPP/Envoy gateway is a **single point of failure** even if both GPU nodes survive. For production HA consider redundant gateway/router instances and a separate front-door failover strategy.
- Watch for vLLM telemetry changes, config drift between models, missing KV events (precise mode), routing to stale endpoints and uneven H100/H200 utilization.
- Updating `endpoints.yaml` may be hot-reloaded by file discovery; modifying EPP scoring configuration requires an EPP restart.
- Rollback: switch the existing gateway/Nginx upstream back to the known-good direct vLLM endpoint(s). Keep this path until the routed setup is proven.

## 7. Acceptance criteria

1. Existing clients see the same OpenAI-compatible URLs, model names, response formats and SSE behaviour.
2. Requests distribute only among endpoints that serve the **requested** model.
3. Repeated prefixes show **measurable actual** APC reuse on the selected backend; cache locality is not treated as guaranteed.
4. Under high load, saturation overrides cache affinity when doing so improves service objectives.
5. On an endpoint failure, *new* requests can use remaining eligible replicas; in-flight request semantics and retries are explicitly defined.
6. Monitoring exposes per-endpoint traffic, routing decisions (as available), queue depth, TTFT, tokens/s, actual vLLM prefix-cache behaviour and backend health.
7. The direct gateway-to-vLLM route remains available as a tested rollback.
8. With realistic long-context concurrent workloads, the routed setup demonstrably improves at least one chosen objective (for example p95 TTFT or successful throughput) without unacceptable regression elsewhere.

## 8. Open questions to settle before implementing

- Current deployment: one Qwen3.8-27B-FP8 replica on H100 and one per H200 (three endpoints in one model pool). If model identity changes, split pools accordingly.
- Current deployment uses two TP=1 H200 replicas. Reassess TP=2 separately only after obtaining relevant throughput/concurrency measurements.
- Exact vLLM image and llm-d release, especially for KV-events and tokenizer compatibility.
- Current gateway implementation and whether its upstream can be changed to a private Envoy listener without modifying its application logic.
- Are nodes in different buildings or datacentres? Check RTT and bandwidth, particularly before considering precise event streams or future KV transfer.
- Does CPU KV offload affect the reported/usable cache residency and precise scorer for the chosen release?
- What failure behavior and p95 latency target matter most to the users?

## 9. References

- [llm-d: No-Kubernetes deployment](https://llm-d.ai/docs/infrastructure/no-kubernetes-deployment)
- [llm-d: running without Kubernetes, motivation and file discovery](https://llm-d.ai/blog/running-llm-d-without-kubernetes)
- [llm-d: EPP architecture and Envoy ext_proc](https://llm-d.ai/docs/dev/architecture/core/router/epp)
- [llm-d: prefix-cache-aware routing, approximate and precise](https://llm-d.ai/docs/dev/architecture/advanced/kv-management/prefix-cache-aware-routing)
- [llm-d: KV-cache indexer and token producer](https://llm-d.ai/docs/dev/architecture/advanced/kv-management/kv-indexer)
- [llm-d: precise routing deployment guide, versioned example](https://llm-d.ai/docs/0.8/well-lit-paths/foundations/precise-prefix-cache-routing)
- [llm-d: EPP scheduling/scorers](https://llm-d.ai/docs/architecture/core/router/epp/scheduling)
- [vLLM: KV events configuration API](https://docs.vllm.ai/en/latest/api/vllm/config/kv_events/)
- Existing repository companion: [Ansible deployment and configuration](ansible_vllm_gpu_node_deployment.md)

> **Implementation note:** This is an architecture and rollout specification, not a promise that a single EPP YAML, CLI flag set or container image will work across all llm-d/vLLM versions. Generate the concrete manifests and commands from the **pinned release's official no-Kubernetes examples**, then test them on the gateway before production cutover.


## 10. Optimized baseline migration for the observed 70/30/0 split

**Observation (2026-10-15):** EPP-selected traffic is approximately **70% H100 / 30% H200-A / 0% H200-B** even though the H100 is generally the lower-capacity worker. The aim is **not** a fixed 20/40/40 ratio. The aim is higher total useful tokens/s, lower tail TTFT/TPOT, and safe use of all healthy capacity while retaining worthwhile KV affinity.

**First separate two failure classes:**
- **Discovery/path problem:** if the third endpoint is missing, unhealthy, or both H200 shim paths reach the same vLLM process, weighting cannot fix it.
- **Scheduling problem:** if all three appear as independent, selectable backends and H200-B is healthy, strong prefix affinity or the older max-score weighted policy may be reinforcing the H100 cache and starving a replica.

### 10.1 What actually changed upstream

The official [optimized-baseline values](https://github.com/llm-d/llm-d/blob/main/guides/optimized-baseline/router/optimized-baseline.values.yaml) now define **four plugins**, and only **two scheduling entries**:

\`\`\`yaml
plugins:
  - type: approx-prefix-cache-producer
  - type: inflight-load-producer
  - type: prefix-cache-affinity-filter
  - type: token-load-scorer
schedulingProfiles:
  - name: default
    plugins:
      - pluginRef: prefix-cache-affinity-filter
      - pluginRef: token-load-scorer
\`\`\`

**Do not carry over** \`prefix-cache-scorer\` + \`queue-scorer\` + \`kv-cache-utilization-scorer\` + \`no-hit-lru-scorer\` weights, or add \`max-score-picker\`, when reproducing this precise reference profile. The [upstream no-Kubernetes example configuration](https://github.com/llm-d/llm-d/blob/main/guides/no-kubernetes-deployment/router/epp/config.yaml) currently still uses the **older weighted profile**, despite its comment claiming it mirrors the optimized baseline. Explicitly combine its *file-discovery/dataLayer wiring* with the *newer optimized-baseline scheduling plugins* rather than copy either file verbatim.

### 10.2 Standalone EPP config for an initial smoke test

Use the following combined YAML **only on a router release verified to support these plugins and the \`llm-d.ai/v1\` config schema**. It is a candidate \`/etc/epp/config.yaml\`; preserve the existing working container mounts, Envoy \`ext_proc\` setup, and file inventory. Initial plugin defaults are **not calibrated** for this heterogeneous fleet.

\`\`\`yaml
apiVersion: llm-d.ai/v1
kind: EndpointPickerConfig

plugins:
  - name: file-discovery
    type: file-discovery
    parameters:
      path: /etc/epp/endpoints.yaml
      watchFile: true

  - name: metrics-source
    type: metrics-data-source
  - name: metrics-extractor
    type: core-metrics-extractor

  - type: approx-prefix-cache-producer
  - type: inflight-load-producer
  - type: prefix-cache-affinity-filter
  - type: token-load-scorer

schedulingProfiles:
  - name: default
    plugins:
      - pluginRef: prefix-cache-affinity-filter
      - pluginRef: token-load-scorer

dataLayer:
  injectDefaults: false
  discovery:
    endpoints:
      pluginRef: file-discovery
  sources:
    - pluginRef: metrics-source
      extractors:
        - pluginRef: metrics-extractor
\`\`\`

The \`dataLayer.discovery.endpoints.pluginRef\` shape and **nested** \`sources[].extractors\` follow the current no-Kubernetes upstream sample. Older EPP versions may have different syntax. Validate configuration loading through EPP startup logs **before directing user traffic**; never assume this is compatible with an unknown pinned router version.

### 10.3 Endpoints and gateway-local Nginx shims

Preserve the [path-multiplexing companion design](llm_d_path_multiplexed_vllm_replicas.md). Since both remote H200 processes are exposed by the same \`H200_HOST:9080\`, they need separate gateway-local socket identities. The file-discovery plugin requires **literal IPv4 addresses** and numeric port **strings**, not hostnames or paths.

\`\`\`yaml
# /etc/epp/endpoints.yaml (illustrative; substitute reachable IPv4s)
endpoints:
  - name: h100
    address: "192.0.2.10"
    port: "9080"
    labels:
      model: qwen
  - name: h200-a
    address: "127.0.0.1"
    port: "18001"
    labels:
      model: qwen
  - name: h200-b
    address: "127.0.0.1"
    port: "18002"
    labels:
      model: qwen
\`\`\`

Here \`192.0.2.10\` is a documentation-only stand-in. Gateway-local \`127.0.0.1\` shims work **only if Envoy and the scraper share that network namespace** (for example host networking); otherwise use private gateway addresses reachable from their containers. For every endpoint, verify \`/health\`, \`/metrics\`, \`/v1/models\` and one known-safe streamed request. Check **different metric counters/processes** at ports \`18001\` and \`18002\`; a successful HTTP response alone does not prove separate backends. File discovery does not by itself guarantee active health-based exclusion; verify the actual EPP/Envoy failure semantics and detection delay.

### 10.4 Diagnosis and Prometheus

Check that \`llm_d_epp_ready_endpoints\` corresponds to **three** eligible endpoints, that EPP selections really occur on three distinct endpoint names, and that selected traffic matches the **actual** per-vLLM request counters. Scrape EPP \`/metrics\` (typically port \`9090\`) and vLLM \`/metrics\` on **each independent shim**. Check EPP logs and \`file-discovery\` startup inventory. The following are actual documented EPP metric names:

\`\`\`promql
# Selection rate per replica, including failure/success status:
sum by (endpoint_name, status) (
  rate(llm_d_epp_scheduler_attempts_total[5m])
)

# Estimated in-flight token work per endpoint:
sum by (endpoint_name) (
  llm_d_epp_inflight_tokens
)

# Estimated concurrent requests per endpoint:
sum by (endpoint_name) (
  llm_d_epp_inflight_requests
)

# p95 routing overhead:
histogram_quantile(
  0.95,
  sum by (le) (
    rate(llm_d_epp_scheduler_e2e_duration_seconds_bucket[5m])
  )
)
\`\`\`

The per-endpoint load gauges only materialize when the corresponding producer is configured and exposed. \`llm_d_epp_ready_endpoints\` is **pool-scoped**, not a per-endpoint health table, so use selection labels, worker metrics and direct probes as well. Inspect the exact label sets and metric availability of the pinned router release.

Expected diagnostic decision:
1. **Only two endpoints loaded:** repair inventory/shims/container networking or health filtering, not scorer weights.
2. **Three loaded, zero EPP selections for H200-B:** instrument filter candidate sets and scorer outcomes, compare with a temporary minimal load-only/no-affinity control.
3. **EPP selects H200-B but vLLM shows zero traffic:** fix Envoy original-destination routing, gateway shim paths or Nginx proxying.
4. **All healthy but H100 has persistently worse p95 TTFT and higher queues:** reduce cache stickiness and test hardware-aware routing.

### 10.5 Calibrate rather than assume equal worker capacity

The upstream \`prefix-cache-affinity-filter\` default \`peakPrefillThroughput: 15928\` tokens/s was measured for **Qwen3-32B on H100 80GB with TP=2**, not our Qwen3.8-27B-FP8 **TP=1** on H100 and H200. Follow [upstream calibration methodology](https://github.com/llm-d/llm-d/blob/main/guides/recipes/router/calibration/README.md), but run equivalent uncached prefill measurements separately against each real worker. The recipe's estimate is:

\`\`\`text
peakPrefillThroughput = CHUNK_SIZE / median(TTFT)
\`\`\`

Match \`CHUNK_SIZE\` to vLLM \`--max-num-batched-tokens\`, keep FP8 KV, model revision, speculative decoding, CPU KV offloading and sampling parameters representative of production, and use cold/unique prompt prefixes to avoid cached-token contamination. Repeat with long-context multi-turn coding agent loads, sweeping concurrency, and measure input/output tokens/s, p50/p95/p99 TTFT, TPOT, queue wait, APC hits, KV utilization and failures. Consider comparing \`token-load-scorer\` against \`active-request-scorer\` for decode-heavy traffic as a **separate** A/B policy, not by arbitrarily blending weights.

**Crucial limitation:** the stock optimized baseline does not automatically normalize token-load scores by each worker's measured H100/H200 throughput. Its shared \`peakPrefillThroughput\` parameter is not a per-replica capacity table. Do not claim that configuring one number produces optimal heterogeneous scheduling. If performance remains unbalanced, evaluate (a) a version-supported per-endpoint hardware preference scorer using verified labels, or (b) a custom capacity-normalized scorer using **measured** per-replica prefill/decode rates, queue load, and KV headroom. A static ratio such as 20/40/40 is merely a test hypothesis, not a target SLO.

### 10.6 Rollout, comparison and rollback

1. Save current EPP YAML, image tag/digest, discovery file, Envoy config and gateway route. Record 70/30/0 and baseline throughput/latencies across representative periods. Ensure there is a direct rollback route.
2. Probe H100, H200-A and H200-B independently, including metrics counter attribution. Abort migration if an H200 cannot be independently reached/identified.
3. Stage the combined EPP configuration on a private test listener using the pinned router image/version; inspect startup logs for plugin loading or schema errors. No vLLM restart or ZMQ KV publisher is required for **approximate** affinity.
4. Run matched A/B workloads: cold single-turn, repeated cached prefixes, diverse coding-agent conversations, 262k-capable long prompts, and concurrency sweeps. Observe per-replica **chosen requests**, not just percentages.
5. Start a restricted canary from the corporate gateway while preserving the old upstream route. Monitor timeouts, token throughput, error rate, p95/p99 TTFT/TPOT, cache hits, queueing, memory and CPU offload pressure.
6. Only then tune affinity saturation thresholds and hardware preference; validate them against the **p95 latency and total throughput**, not traffic balance alone.
7. Roll back by restoring the saved EPP config/previous container and gateway direct route if model identity is wrong, one endpoint remains unroutable, or latency/errors worsen materially.

**Acceptance:** all three healthy replicas demonstrably receive traffic on sufficiently varied workloads; no wrong-model or duplicate-shim routing; consistent SSE streaming; reduced H100 overload; improved aggregate useful throughput and/or p95 TTFT without unacceptable APC regression. Keep long-lived streaming request failure behavior explicit: a new routing decision does not migrate an existing stream or a KV cache.

**Source alignment as of 2026-10-15:**
- [Optimized baseline upstream reference](https://github.com/llm-d/llm-d/blob/main/guides/optimized-baseline/router/optimized-baseline.values.yaml)
- [Standalone EPP file discovery and dataLayer reference](https://github.com/llm-d/llm-d/blob/main/guides/no-kubernetes-deployment/router/epp/config.yaml)
- [Endpoint discovery schema, IPv4 constraints and network requirements](https://github.com/llm-d/llm-d-router/blob/main/docs/discovery.md)
- [EPP metric definitions](https://github.com/llm-d/llm-d-router/blob/main/docs/metrics.md)
- [Prefill throughput calibration](https://github.com/llm-d/llm-d/blob/main/guides/recipes/router/calibration/README.md)


## 11. Precise prefix cache, CPU offloading, and hybrid Qwen: feasibility review

**Reviewed:** 2026-10-15. **Status:** design option / conditional next phase, **not** a validated drop-in implementation. This section reviews a proposed precise-prefix architecture for TP=1 Qwen3.8-27B-FP8 on H100 + 2×H200, 262k max context, FP8 KV, MTP speculative decoding and long-running coding agents. The intended future scale is about 300 developers, not 300 simultaneously active requests.

### 11.1 Decision

**Do not replace the optimized-baseline rollout in §10 with precise prefix routing immediately.** First fix 70/30/0, verify three independent replica identities, benchmark approximate affinity and H100/H200 load. Next validate native CPU KV offloading **on one replica** with actual Qwen hybrid-state reuse. Only if the observed routing errors are material should precise indexing and extra ZMQ infrastructure be deployed.

| Capability | Supported / qualified finding |
| --- | --- |
| Approximate prefix routing | Learns cache placement from past routing, not real vLLM residency; works with existing HTTP-only network |
| Precise KV routing | Real event-driven block index and exact token rendering; requires per-replica ZMQ event transport plus replay/recovery |
| CPU KV offload | Native vLLM \`OffloadingConnector\` can extend a local replica's usable prefix cache; works independently of precise routing |
| Precise CPU event indexing | \`self_describing_kv_events\` is opt-in for native offloading; emitted metadata has documented limitations, including independently unretrievable CPU sub-chunks |
| Tier-aware precise scoring | Architecture docs describe media and defaults \`gpu=1.0\`, \`cpu=0.8\`; **do not assume the proposed \`mediumWeights\` YAML field is supported** or that end-to-end hybrid-aware cache scoring is production-ready |
| H100/H200 heterogeneity | Precise locality alone does not normalize by accelerator throughput or queue capacity |
| Two active-active EPP replicas | **Not safe by default** with current reference token-load accounting and index replay semantics |
| P2P remote CPU KV sharing | Separate experimental design path, not enabled by KV events, exact rendering or local OffloadingConnector alone |

### 11.2 Why the original proposed precise EPP YAML is not deployable

The actual upstream **precise-routing** guide builds on the optimized baseline; its plugin chain uses \`token-producer\`, \`endpoint-notification-source\`, \`precise-prefix-cache-producer\`, \`inflight-load-producer\`, \`prefix-cache-affinity-filter\`, and \`token-load-scorer\`. The deployed scheduling profile contains **only** the affinity filter and token-load scorer, not \`precise-prefix-cache-scorer\`, \`load-aware-scorer\`, \`max-score-picker\`, \`single-profile-handler\` or arbitrary weighted scoring from the draft.

Reference excerpt **only** (not a complete EPP YAML; the standalone file-discovery adaptation remains implementation work):

\`\`\`yaml
plugins:
  - type: token-producer
    parameters:
      modelName: Qwen/Qwen3-32B
      vllm:
        endpointDiscovery: {}
        messagesRenderMode: auto
        responsesRenderMode: auto
  - type: endpoint-notification-source
  - type: precise-prefix-cache-producer
    parameters:
      tokenProcessorConfig:
        blockSizeTokens: 64
        hashAlgorithm: xxh64
      speculativeIndexing: true
      indexerConfig:
        kvBlockIndexConfig:
          enableMetrics: true
      kvEventsConfig:
        engineType: vllm
        topicFilter: "kv@"
        concurrency: 8
        discoverPods: true
        podDiscoveryConfig:
          socketPort: 5556
          replaySocketPort: 5559
  - type: inflight-load-producer
    parameters:
      prefixMatchInfoProducerName: precise-prefix-cache-producer
  - type: prefix-cache-affinity-filter
    parameters:
      prefixMatchInfoProducerName: precise-prefix-cache-producer
      peakPrefillThroughput: 15926
  - type: token-load-scorer

schedulingProfiles:
  - name: default
    plugins:
      - pluginRef: prefix-cache-affinity-filter
      - pluginRef: token-load-scorer
\`\`\`

**All model IDs and the 15,926 tokens/s calibration above are upstream reference values for Qwen3-32B, H100 TP=2, and must be replaced/calibrated for this fleet.** \`discoverPods: true\` / \`endpointDiscovery: {}\` assume the documented Kubernetes model-server discovery integration, so **do not copy them into the present standalone Podman/file-discovery EPP as if they worked**. Establish version-supported event-source discovery and render endpoint selection; retain the standalone data-layer wiring from §10 only after validating plugin dependencies. Upstream uses \`--enable-scale-out\` to expose \`/v1/*/render\` on supported vLLM model-server builds. These render requests can traverse HTTP shims, provided the rewrite and model identity are correct.

**Hash caution:** internal router \`xxh64\` block keys need not equal vLLM's own APC hash algorithm \`sha256_cbor\`. What matters is exact rendered tokens, block semantics, coherent event chains, correct model identity and cache-group handling. Do not impose \`PYTHONHASHSEED\` equality as a generic requirement for all precise-index configurations; it is relevant to specific vLLM hashing/transfer setups. Use release-matched official examples, not guessed plugin or flag names.

References: [precise router values](https://github.com/llm-d/llm-d/blob/main/guides/precise-prefix-cache-routing/router/precise-prefix-cache-routing.values.yaml), [model-server KV events/render flags](https://github.com/llm-d/llm-d/blob/main/guides/precise-prefix-cache-routing/modelserver/gpu/vllm/base/patch-vllm.yaml).

### 11.3 Native CPU KV offloading: local usefulness versus router knowledge

Sample **connector JSON**, adapted from the current vLLM offloading documentation. Confirm support in the **pinned** deployed vLLM image before setting this:

\`\`\`json
{
  "kv_connector": "OffloadingConnector",
  "kv_role": "kv_both",
  "kv_connector_extra_config": {
    "spec_name": "CPUOffloadingSpec",
    "cpu_bytes_to_use": 100000000000,
    "self_describing_kv_events": true
  }
}
\`\`\`

- Without event indexing, vLLM can still restore locally offloaded cache when subsequent requests reach the **same** replica. Do not confuse this with \`--cpu-offload-gb\` for model-weight offloading.
- \`self_describing_kv_events\` defaults to \`false\`; setting it to \`true\` is useful **only with KV events enabled separately**, and is not a guarantee that every event represents a reusable unit. Default/fallback and tiering cases can still yield incomplete metadata.
- **Correctness blocker to investigate:** [vLLM issue #57159](https://github.com/vllm-project/vllm/issues/57159) was open on 2026-10-15. Under \`blocks_per_chunk > 1\`, the connector may announce GPU-size sub-block hashes even when the CPU store retrieves only the full, larger chunk. A prefix that ends inside that CPU chunk may therefore be *advertised as a hit but fail retrieval*. Test partial-prefix divergence at every offload chunk boundary.
- Published event \`medium\` and group metadata do not imply the current llm-d scoring path has verified tier-aware **hybrid** hit semantics. llm-d's [KV-indexer docs](https://github.com/llm-d/llm-d/blob/main/docs/architecture/kv-management/kv-indexer.md) describe tier weights (\`gpu=1.0\`, \`cpu=0.8\`) and characterize hybrid-attention-aware scoring as work in progress. Inspect the **specific EPP/indexer version's source and config schema** before trying to override weights.
- For a first CPU-tier proof, leave event-based precise routing off; prove actual CPU store/load and restoration with one directly addressed vLLM instance, then benchmark two independent H200 replicas with isolated tiers.

Reference: [vLLM KV offloading usage](https://github.com/vllm-project/vllm/blob/main/docs/features/kv_offloading_usage.md).

### 11.4 Hybrid attention and realistic transfer costs

The Qwen model combines full-attention KV with Gated DeltaNet / linear-attention state. A block-level event about **full-attention KV alone** does not establish that the recurrent state necessary to resume at a particular prefix boundary is restored. Prefix caching/offloading requires correct cache groups and valid state checkpoints across all relevant layers.

The preliminary ~3 GB estimate for a 95k-token full-attention prefix is **not** a measured host↔device copy size for the entire hybrid cache. The often-quoted 60–80 ms CPU reload must **not** be adopted as an SLO or benchmark result: PCIe transfers, pinned-memory pressure, DMA overlap, CPU NUMA locality, offload-chunk granularity, scheduling, and recurrent-state restoration all contribute. Validate with GPU and CPU tiers independently and report p50/p95 TTFT and total CPU restore latency.

Critical regression cases using the *same pinned Qwen3.8 FP8 weights, 262k context, FP8 KV and MTP settings*:

1. Repeated prompt fully resident in GPU cache: verify actual APC hits and correct generated continuation.
2. Long prompt after forcing GPU eviction but while CPU cache retains its prefix: verify **external-cache hit/load counters**, prompt tokens recomputed and latency.
3. Prefix that diverges midway through an offload chunk: ensure no false reusable-prefix claims.
4. Recurrent-state/hybrid-group boundary changes, evictions and restarts: check output equivalence and whether warmup is required.
5. Mix short and 50k–200k agentic prompts at increasing concurrency; compare offloading on/off and routing strategies.

Do not assume fixed \`--max-num-seqs 32\`, \`--block-size 64\`, or CPU offload size are universally valid for the model and vLLM release. Use observed max concurrency and group-specific block requirements.

### 11.5 Network and EPP high availability

**Today's connectivity limitation:** the H200 server exposes **one external HTTP listener**, \`H200_HOST:9080\`, with paths \`/modelA/\` and \`/modelB/\`; gateway-local shims (\`18001\` and \`18002\`) preserve independent HTTP endpoints. **Ordinary Nginx HTTP location rewriting cannot forward raw ZMQ streams by URL path.**

A precise deployment must obtain separately reachable, identity-stable **KV-event** and **replay** TCP sockets per vLLM process, e.g. a pair of restricted private ports per replica, TCP \`stream\` proxies with independently bound gateway listeners, or an approved tunnel/relay. These transports require security approval, socket-level health/reconnect handling and topic/index identity mapping. Merely setting \`endpoint: tcp://*:5557\` and a model topic is insufficient. The pinned upstream guide uses **5556 (events), 5559 (replay)** and event topics incorporating each worker's inference endpoint. Ensure the router can discover and subscribe to both H200 processes **despite their shared :9080 HTTP entrypoint**.

**Start with one EPP:** upstream explicitly fixes \`router.epp.replicas: 1\` for its precise guide. In-flight token accounting is process-local, and a newly started index may have a replay window too short to reconstruct all warm cache. Two routers subscribing to the same events does not automatically yield consistent index history, complete recovery, or accurate global load. Active-active would require shared or reconciled in-flight state and verifiable full-index recovery (current snapshot support is experimental), plus an HA plan for Envoy/gateway.

### 11.6 Memory and hardware-aware load balance

Proposing **100 GB of CPU KV on each H200** implies 200 GB of requested offload tier on their **shared host**, before vLLM process RSS, pinned buffers, \`/dev/shm\`, any other workload, and OS headroom. A machine with 200–250 GB host RAM may be at significant OOM/swap risk. Start with a smaller tier per replica; measure \`MemAvailable\`, RSS, page faults, NUMA locality and connector store/load counters. Host memory allocation and CPU tier sizing must be considered **per host**, not per GPU in isolation.

Precise routing by itself may **reinforce the existing 70/30/0 H100 preference** if that worker genuinely holds the longest reused prefixes. Proper heterogeneous scheduling should approximate:

\`\`\`text
estimated_request_cost(endpoint) =
    queue_wait(endpoint)
  + cache_restore_time(endpoint, GPU/CPU/miss)
  + uncached_prefill_tokens / measured_prefill_rate(endpoint)
  + decode_congestion_penalty(endpoint)
\`\`\`

This is a *proposed cost function*, not an available built-in EPP plugin. The stock precise filter + token-load scorer is not proof of per-GPU performance normalization.

### 11.7 Staged evaluation and acceptance gates

| Stage | Change | Continue only if |
| --- | --- | --- |
| A | Complete §10 optimized baseline and correct the zero-traffic H200 | All three independent eligible endpoints; better aggregate tokens/s and tail latency without excessive cache-loss |
| B | Run native CPU offload on one H200 directly, without precise routing | Verified real CPU-tier store **and** load, correct hybrid state, useful TTFT improvement and acceptable memory overhead |
| C | Add tier-aware **approximate** policy to account for local CPU cache | Better TTFT or actual reuse on repeated multi-turn sessions versus B |
| D | Pilot precise GPU-prefix event indexing with exact render and ZMQ/replay connectivity | Correct event identity/replay and lower wasted uncached prefill versus A |
| E | Pilot CPU self-describing events and tier-aware precise routing, **only after issue and hybrid checks** | No false partial-chunk hits, correct state restoration, robust restart/recovery, useful improvement versus C |
| F | Evaluate P2P cache transfer and active-active EPP separately | Transfer beats recompute under measured local network RTT/bandwidth, and state synchronization/HA is proven |

For each experiment, record per-endpoint request distribution, **actual** cached tokens, in-flight work, queue wait, prefill tokens/s, decode tokens/s, p95/p99 TTFT/TPOT, cache store/load bytes, CPU/GPU memory pressure, errors, and EPP overhead. Replay the *same* multi-turn tool-using traces across variants, plus cold sessions and concurrency sweeps.

**Production decision:** if the gap between approximate routing with CPU offload and precise routing with CPU offload is small, keep the simpler HTTP-only approximate design. Do not implement remote ZMQ, precise index recovery, or cross-node KV transfer without measured evidence that they improve the required service objectives.

**Related upstream references:**
- [Precise routing guide](https://github.com/llm-d/llm-d/blob/main/guides/precise-prefix-cache-routing/README.md)
- [Precise EPP plugins](https://github.com/llm-d/llm-d/blob/main/guides/precise-prefix-cache-routing/router/precise-prefix-cache-routing.values.yaml)
- [Tiered-prefix-cache guide](https://github.com/llm-d/llm-d/blob/main/guides/tiered-prefix-cache/README.md)
- [Tiered CPU producer/scorer config](https://github.com/llm-d/llm-d/blob/main/guides/tiered-prefix-cache/router/single-host/tiered-prefix-cache-cpu.values.yaml)
- [vLLM offloading connector](https://github.com/vllm-project/vllm/blob/main/docs/features/kv_offloading_usage.md)
- [vLLM offload event correctness issue #57159](https://github.com/vllm-project/vllm/issues/57159)
- [KV-indexer architecture and hybrid warning](https://github.com/llm-d/llm-d/blob/main/docs/architecture/kv-management/kv-indexer.md)
