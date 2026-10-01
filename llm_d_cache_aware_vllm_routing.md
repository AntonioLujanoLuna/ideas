# Cache-aware llm-d routing for the H100 and 2 × H200 vLLM nodes

**Status:** implementation proposal  
**Date:** 2026-10-01  
**Target:** existing gateway + one H100 host + one dual-H200 host, using Podman and Nginx without Kubernetes.

## 1. Objective and recommendation

Introduce llm-d's **Endpoint Picker (EPP)** and **Envoy** *behind* the existing API gateway, preserving the current public interface, authentication, quotas and other gateway logic. EPP should select a vLLM endpoint using **prefix-cache affinity and load**, not simple round robin. Each vLLM process retains its **own local KV cache**; llm-d's router tracks or estimates where useful prefixes reside, but does **not** automatically transfer or share KV tensors.

**Recommended rollout:** keep both existing vLLM deployments independently reachable; introduce Envoy + EPP with static/file-based endpoint discovery; start with approximate prefix awareness; benchmark against round robin; optionally enable precise cache indexing through vLLM KV events. Do not introduce Kubernetes, shared KV storage or prefill/decode disaggregation for this first deployment.

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

Configure the `approx-prefix-cache-producer`, `prefix-cache-scorer` and appropriate load/saturation scorers in the EPP configuration. Start from the chosen release's optimized-baseline/no-Kubernetes sample, not an unvalidated configuration copied from another EPP version. Tune weights and any maximum prefix-matching limit against our long (up to 262k-token) agent conversations. If the configured index tracks only a short prefix, it cannot fully distinguish two long sessions with an identical beginning.

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
4. Configure EPP's approximate prefix producer, prefix scorer, load-awareness, health filtering and saturation rules.
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

- Same 27B model across H100/H200 or a different, larger H200 model? This determines single-pool versus multi-model routing.
- One TP=2 H200 endpoint or two TP=1 replicas? Benchmark memory headroom, model fit and concurrency.
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
