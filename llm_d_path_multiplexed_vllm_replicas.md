# llm-d with path-multiplexed vLLM replicas behind one Nginx port

**Status:** implementation note  
**Date:** 2026-10-05  
**Target:** one H100 vLLM endpoint plus two independent vLLM replicas on a dual-H200 host, where the H200 host can expose only TCP port `9080` and Nginx distinguishes replicas by URL prefix.

Companion to [`llm_d_cache_aware_vllm_routing.md`](llm_d_cache_aware_vllm_routing.md).

## 1. Problem

Assume the dual-H200 machine runs one vLLM process per GPU but externally exposes both through the same Nginx listener:

```text
H200_HOST:9080/modelA/v1/chat/completions
H200_HOST:9080/modelB/v1/chat/completions

H200_HOST:9080/modelA/v1/metrics
H200_HOST:9080/modelB/v1/metrics
```

This works for normal HTTP clients because Nginx can route on the URL path.

The problem is that llm-d's Endpoint Picker ultimately selects a serving destination as an endpoint such as an address and port. The two H200 replicas therefore collapse to the same serving destination from llm-d's point of view:

```text
H200_HOST:9080
H200_HOST:9080
```

The path prefix is not an independent schedulable endpoint.

Therefore, simply exposing different metrics URLs does **not** make the replicas independently routable. Even if metrics for `modelA` and `modelB` are distinguishable, the selected HTTP destination still needs to identify which replica should receive the request.

## 2. Recommended solution

Create two small proxy/shim listeners on the machine that runs Envoy/EPP, or on another machine visible to them.

Each shim has a unique socket and forwards to one path on the H200 host:

```text
                          dual-H200 server
                       only :9080 reachable
                  +----------------------------+
                  | Nginx :9080                |
                  |                            |
                  | /modelA/* -> vLLM GPU 0    |
                  | /modelB/* -> vLLM GPU 1    |
                  +-------------^--------------+
                                |
                                |
        +-----------------------+-----------------------+
        |                                               |
127.0.0.1:18001                                  127.0.0.1:18002
H200 replica A shim                              H200 replica B shim
        |                                               |
        +-----------------------+-----------------------+
                                |
                         Envoy + llm-d EPP
```

llm-d then sees two genuinely distinct endpoints:

```text
127.0.0.1:18001
127.0.0.1:18002
```

while the shims translate them to:

```text
H200_HOST:9080/modelA/...
H200_HOST:9080/modelB/...
```

This preserves the network restriction on the H200 host. No extra externally reachable H200 ports are required.

## 3. Shim Nginx configuration

A minimal example on the llm-d/Envoy gateway:

```nginx
server {
    listen 127.0.0.1:18001;

    location = /metrics {
        proxy_pass http://H200_HOST:9080/modelA/v1/metrics;
    }

    location / {
        proxy_pass http://H200_HOST:9080/modelA/;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header Connection "";

        proxy_buffering off;
        proxy_request_buffering off;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}

server {
    listen 127.0.0.1:18002;

    location = /metrics {
        proxy_pass http://H200_HOST:9080/modelB/v1/metrics;
    }

    location / {
        proxy_pass http://H200_HOST:9080/modelB/;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header Connection "";

        proxy_buffering off;
        proxy_request_buffering off;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

With this arrangement, Envoy/EPP can address the replicas using ordinary OpenAI-compatible paths:

```text
127.0.0.1:18001/v1/chat/completions
127.0.0.1:18002/v1/chat/completions

127.0.0.1:18001/metrics
127.0.0.1:18002/metrics
```

The shims then forward those requests to:

```text
H200_HOST:9080/modelA/v1/chat/completions
H200_HOST:9080/modelB/v1/chat/completions

H200_HOST:9080/modelA/v1/metrics
H200_HOST:9080/modelB/v1/metrics
```

### Container networking note

If Envoy and EPP run inside Podman, `127.0.0.1` inside those containers is not automatically the host loopback.

Use one of the following:

1. Run the relevant router containers with host networking.
2. Bind the shim listeners to a private gateway address reachable from the containers.
3. Put the shim and router containers on the same Podman network and address the shim by container/service name.

Do not expose the shim ports publicly.

## 4. llm-d endpoint inventory

Conceptually, the file-discovery configuration should expose the two shims as separate endpoints.

The exact schema must match the pinned llm-d release, but the endpoint identity should look like:

```yaml
endpoints:
  - name: h200-gpu0
    address: "GATEWAY_IP"
    port: "18001"
    labels:
      model: qwen

  - name: h200-gpu1
    address: "GATEWAY_IP"
    port: "18002"
    labels:
      model: qwen
```

The key property is not the label. It is that the replicas have **different address:port identities** from llm-d's perspective.

## 5. Complete H100 + 2 x H200 topology

Assume the H100 machine already exposes one directly addressable vLLM endpoint:

```text
H100_HOST:9080
```

Then the llm-d pool can contain three independent schedulable replicas:

```text
                         +------------------+
client -> gateway -----> | Envoy + EPP      |
                         +--------+---------+
                                  |
              +-------------------+-------------------+
              |                   |                   |
              v                   v                   v
      H100_HOST:9080       GATEWAY_IP:18001     GATEWAY_IP:18002
              |                   |                   |
              |                   v                   v
              |          H200_HOST:9080/      H200_HOST:9080/
              |             modelA/...           modelB/...
              |                   |                   |
              v                   v                   v
          H100 GPU            H200 GPU 0          H200 GPU 1
```

Conceptually:

```yaml
endpoints:
  - name: h100
    address: "H100_HOST"
    port: "9080"
    labels:
      model: qwen

  - name: h200-0
    address: "GATEWAY_IP"
    port: "18001"
    labels:
      model: qwen

  - name: h200-1
    address: "GATEWAY_IP"
    port: "18002"
    labels:
      model: qwen
```

From llm-d's perspective these are three independent backends, even though both H200 shim endpoints eventually traverse the same remote TCP port `9080`.

## 6. Existing H200 Nginx

The H200-side reverse proxy can continue multiplexing the replicas by path.

For example:

```nginx
server {
    listen 9080;

    location /modelA/ {
        proxy_pass http://127.0.0.1:8000/;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header Connection "";

        proxy_buffering off;
        proxy_request_buffering off;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }

    location /modelB/ {
        proxy_pass http://127.0.0.1:8001/;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header Connection "";

        proxy_buffering off;
        proxy_request_buffering off;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

The internal vLLM ports can remain local to that host. The only remotely reachable port remains `9080`.

## 7. Why metrics paths alone are insufficient

Suppose EPP can read:

```text
H200_HOST:9080/modelA/v1/metrics
H200_HOST:9080/modelB/v1/metrics
```

That provides two metric streams, but it does not solve serving destination selection.

After deciding that GPU 0 is preferable, the data plane still needs a destination that unambiguously means GPU 0.

If both candidates are represented as:

```text
H200_HOST:9080
```

the distinction is lost.

The shim listeners make the identity explicit:

```text
GATEWAY_IP:18001 -> H200 GPU 0
GATEWAY_IP:18002 -> H200 GPU 1
```

This is the important part of the design.

## 8. Alternative: multiple IP addresses

If infrastructure allows assigning two private IP addresses to the H200 host, an even cleaner design is:

```text
10.10.20.21:9080 -> modelA
10.10.20.22:9080 -> modelB
```

Nginx can bind both addresses on the same port and route each socket to a different local vLLM instance.

Then llm-d can use the two H200 replicas directly:

```yaml
endpoints:
  - name: h200-gpu0
    address: "10.10.20.21"
    port: "9080"

  - name: h200-gpu1
    address: "10.10.20.22"
    port: "9080"
```

This removes the gateway-side shim layer.

However, if the H200 machine is constrained to one externally visible address and one externally visible port, the two shim endpoints are the simplest solution.

## 9. KV-event routing is a separate issue

The HTTP path multiplexing solution covers:

- OpenAI-compatible inference traffic.
- Health/metrics traffic.
- Independent llm-d backend identities.
- Approximate prefix-cache-aware routing.

It does **not** automatically solve vLLM precise KV-event routing.

vLLM's KV-event publisher uses ZMQ/TCP. It is not an HTTP endpoint that can simply be exposed as:

```text
/modelA/kv-events
/modelB/kv-events
```

behind ordinary HTTP Nginx path routing.

If precise llm-d cache indexing is enabled later, each vLLM process must expose a distinct KV-event publisher identity reachable by the component consuming those events. Options include:

- Separate reachable ZMQ ports.
- Separate private IP addresses using the same ZMQ port.
- A TCP-level proxy with distinct listener sockets on the gateway or H200 host.

For the initial deployment, approximate cache-aware routing avoids this extra complexity and requires no KV-event publisher.

## 10. Recommended deployment order

1. Keep the two existing H200 vLLM processes unchanged.
2. Keep H200 Nginx on `:9080` with `/modelA/` and `/modelB/`.
3. Add two private shim listeners on the llm-d/Envoy gateway.
4. Verify direct calls through each shim independently.
5. Verify `/metrics` through each shim.
6. Add the H100 endpoint plus both shim endpoints to llm-d file discovery.
7. Run EPP with load-aware routing first.
8. Add approximate prefix-cache affinity.
9. Benchmark routing, TTFT, queueing, APC hit rate and throughput.
10. Only consider precise KV-event routing if approximate routing is insufficient.

## 11. Validation checklist

The deployment is correct when all of the following hold:

- `GATEWAY_IP:18001/v1/models` reaches only H200 replica A.
- `GATEWAY_IP:18002/v1/models` reaches only H200 replica B.
- `GATEWAY_IP:18001/metrics` reports replica A's metrics.
- `GATEWAY_IP:18002/metrics` reports replica B's metrics.
- Streaming responses work through both proxy layers.
- Envoy/EPP reports three distinct eligible backends: H100, H200 GPU 0 and H200 GPU 1.
- Taking one H200 vLLM process down removes or penalizes only that replica.
- Requests routed to H200 GPU 0 never accidentally land on GPU 1, and vice versa.
- Existing clients still use the same public gateway URL.
- H200 still exposes only TCP port `9080` to the network.

## 12. References

- [llm-d router repository](https://github.com/llm-d/llm-d-router)
- [llm-d endpoint discovery documentation](https://github.com/llm-d/llm-d-router/blob/main/docs/discovery.md)
- [llm-d metrics documentation](https://github.com/llm-d/llm-d-router/blob/main/docs/metrics.md)
- [llm-d no-Kubernetes deployment](https://llm-d.ai/docs/infrastructure/no-kubernetes-deployment)
- [vLLM KV events configuration](https://docs.vllm.ai/en/latest/api/vllm/config/kv_events/)

> **Implementation note:** Pin llm-d and vLLM versions before producing the final `endpoints.yaml`, EPP policy and Envoy configuration. The stable architectural requirement here is that each schedulable vLLM replica must have a distinct destination identity from the router's perspective. The exact configuration schema can change between llm-d releases.
