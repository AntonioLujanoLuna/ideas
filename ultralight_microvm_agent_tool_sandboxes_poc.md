# Ultra-light microVM Sandboxes for Agent Tools: Practical PoC

**Status:** Experiment proposal, not deployed or benchmarked.
**Updated:** 2026-10-22.
**Companion:** [Isolated and Distributed Agent Runtime](agent_runtime_isolation_and_remote_workers.md) and [Portal Tool Execution](portal_agent_tool_execution_architecture.md).

## 1. Decision

**Prototype Podman + crun/krun (libkrun) first; compare with Firecracker second.** Keep LangGraph and Copilot SDK agent orchestration outside the VM. Place only untrusted shell commands, file edits, generated Python and test processes in a hardware-isolated guest on a dedicated CPU/RAM worker host.

This is **tool isolation**, distinct from migrating the entire agent application into a per-session VM. A truly small guest needs a shell, selected tools and workspace, not LangGraph, Copilot CLI, FastAPI, an LLM client or persistent application state.

A small **OCI filesystem image can serve as the guest root filesystem** in krun. Unlike a normal container, the command executes inside a KVM guest kernel. Compressed image size does not represent resident/committed guest memory.

~~~text
Portal -> FastAPI + LangGraph/Copilot SDK
       -> server-owned Tool Router / authorization
       -> Agent Sandbox Controller on dedicated CPU/RAM host (:9080)
          -> session-bound krun microVM + workspace A
          -> session-bound krun microVM + workspace B
          -> Firecracker provider (experimental)
       -> enterprise LLM gateway -> existing vLLM GPU nodes

No direct guest access to GPU nodes, host credentials or corporate networks.
~~~

## 2. Runtime shortlist

| Runtime | Why | Caveats |
| --- | --- | --- |
| **crun/krun + libkrun + Podman** | Lowest integration cost if compatible krun-enabled packages and KVM are available. OCI workflows, vCPU/RAM annotations, workspace mounts, standard command execution. | Minimum effective krun RAM configuration is **128 MiB**. Default TSI guest networking uses the VMM's host networking context and must be deliberately contained. |
| **Firecracker** | Designed for minimal KVM guests, explicit process/lifecycle API, virtio-block and host-to-guest virtio-vsock, jailer, snapshots. | Kernel/rootfs creation, per-VM writable block filesystem, vsock tool bridge, network and VM lifecycle are your responsibility. **No native virtio-fs host directory sharing.** |
| **Cloud Hypervisor** | Also lightweight; virtio-fs, APIs, richer VM features. | More features and engineering than initial shell execution needs. |
| Rootless Podman | Baseline and temporary functional fallback if KVM blocked. | Shares host kernel, weaker security boundary. |
| gVisor / WASI | Optional comparison. | Neither is the same as a general-purpose KVM Linux microVM. |

Do not begin with Kubernetes/Kata or a full multi-node scheduler. Evaluate them after the single-worker PoC.

**Working hypotheses, not measurements:** one vCPU + **128 MiB** for krun + Alpine/BusyBox shell; **256–512 MiB** to try Python; **512 MiB to 1 GiB+** for substantial Git/tests/Node. The *whole* Copilot SDK/CLI or LangGraph application requires a separate larger profile. Avoid claiming specific guest count from small rootfs image sizes.

## 3. Prerequisites and host constraints

A dedicated Linux x86_64 or aarch64 host with accessible **/dev/kvm** is required. Rootless Podman does not grant KVM access. If the host is virtualized, investigate nested virtualization and the corporate security policy. Deploy under a dedicated OS identity with no sensitive host mounts.

~~~bash
set -euo pipefail
uname -m
test -c /dev/kvm || { echo "STOP: /dev/kvm is absent"; exit 1; }
test -r /dev/kvm && test -w /dev/kvm || {
  echo "STOP: user lacks read/write access to /dev/kvm"
  exit 1
}
command -v podman
command -v krun
podman info --format 'Rootless={{.Host.Security.Rootless}}'
krun --version
~~~

Provision a compatible **libkrun + libkrunfw + crun/krun + Podman** stack from tested, approved packages and register krun as an OCI runtime. Package availability varies by distribution. Verify krun support independently; merely having a default crun binary is insufficient. Note that upstream libkrun 2.0 main is still under development as of this document, so pin a compatible stable branch/release.

If KVM/krun is unavailable, ordinary rootless Podman can test the tool API only; it must not silently downgrade a production high-risk sandbox's isolation class.

## 4. First functional smoke test

**This is a development smoke test, not a production-hardened sandbox.** The OCI image supplies Alpine userspace, while krun launches a real guest. Use a new empty dedicated path for the workspace.

~~~bash
set -euo pipefail

WORKSPACE="$HOME/agent-microvm-poc/workspace"
mkdir -p "$WORKSPACE"
chmod 700 "$HOME/agent-microvm-poc" "$WORKSPACE"

podman run --rm \
  --runtime=krun \
  --annotation=krun.cpus=1 \
  --annotation=krun.ram_mib=128 \
  --network=none \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --pids-limit=64 \
  --volume="$WORKSPACE:/workspace:Z" \
  --workdir=/workspace \
  docker.io/library/alpine:3.22 \
  sh -ceu 'printf "hello from VM\n" > output.txt; uname -a; cat output.txt'

test "$(cat "$WORKSPACE/output.txt")" = "hello from VM"
printf 'Workspace output: %s\n' "$WORKSPACE/output.txt"
~~~

Compare the guest and host kernel from uname. krun's manual explicitly suggests this verification. Validate the installed runtime, mount semantics, SELinux relabeling and exit-code behavior. Never use volume relabeling on a sensitive shared directory.

**Critical security caveat:** krun's default **TSI networking** proxies guest sockets through the VMM's host network context. The --network=none flag above is **not evidence of default-deny egress** under this VMM. Test the exact installed networking implementation and block unapproved egress **outside** the guest, at an independently enforced host/VMM network boundary. Block metadata endpoints, host services, other tenant sessions, management subnets and Podman sockets. If the corporate environment cannot provide a robust deny rule for krun, treat it as lab-only and test Firecracker **without a guest NIC** instead.

For repeated coding commands, start one session-scoped VM, use tested Podman exec semantics repeatedly, then destroy it after idle TTL; avoid a new boot on every command unless tools are sparse or isolation requirements demand it. Establish actual installed runtime support for exec, cancellation and shutdown first.

## 5. Minimal SDK-independent sandbox contract

Implement a server-owned **SandboxExecutionProvider** callable by existing LangGraph tools and Copilot SDK callback handlers. Conceptual interface (not an existing library API):

~~~text
create(session_id, profile, limits) -> sandbox_id
exec(sandbox_id, argv, cwd, stdin, timeout, max_output_bytes)
  -> exit_code, stdout, stderr
write_file(sandbox_id, workspace_relative_path, bytes) -> digest
read_file(sandbox_id, workspace_relative_path, max_bytes) -> bytes
list_files(sandbox_id, workspace_relative_path) -> entries
terminate(sandbox_id) -> terminal_status
~~~

Prefer argv arrays over implicit shell string interpolation. Offer shell mode only to capabilities explicitly authorized for it. File paths must be relative to /workspace, normalized, checked for symlink traversal and constrained to regular files, with size and disk quotas.

The private Sandbox Controller could implement:

~~~text
POST   /v1/sandboxes
POST   /v1/sandboxes/{id}/exec
PUT    /v1/sandboxes/{id}/files/{relative-path}
GET    /v1/sandboxes/{id}/files/{relative-path}
GET    /v1/sandboxes/{id}/files
DELETE /v1/sandboxes/{id}
~~~

These endpoints are **proposed**, not implemented. Authenticate the backend and authorize each operation using short-lived, server-issued session claims. The model cannot select images, arbitrary host mounts, network privileges or tenant scope. The controller chooses the provider based on security policy: **KrunPodmanSandboxProvider** first and **FirecrackerSandboxProvider** later.

Use one workspace per authorized session, persisted as explicitly scoped artifacts/patches in durable storage independently of guest RAM. Set a **trial 5–15 minute idle TTL** and a hard maximum lifetime; never reuse a dirty VM across tenants. Enforce resource quotas, output caps, command deadlines, cancellation including process-tree/VM teardown, streaming backpressure, filesystem ownership and revocation.

Agent/tool authority remains in the backend. Running in a VM does not authorize sensitive external actions such as Jira modification, Git push, internal search or deployments. Those still need normal policy, credentials and approvals. Do not place gateway tokens, SSH agents, corporate proxy tokens or Podman sockets into guests.

## 6. Firecracker follow-up PoC

Once the tool API works with krun:

1. Boot the official Firecracker minimal example in a lab with KVM, then build a minimal kernel + BusyBox/Alpine **read-only base ext4 rootfs**.
2. Provide **per-VM writable disk copy/overlay** for /workspace; never concurrently attach one mutable ext4 image to multiple guests.
3. Run a minimal command supervisor in the guest and send tool commands/results over **virtio-vsock**. Firecracker bridges guest AF_VSOCK to a host Unix domain socket. Do not expose guest SSH or direct internet networking for the first profile.
4. Firecracker **does not support virtio-fs**. Move files via bounded vsock RPC or per-VM block volumes and controlled artifact imports/exports. Do not promise transparent host directory mounts.
5. Apply the **Firecracker jailer**, independent nonprivileged VMM credentials, cgroups, seccomp and host network controls before any production deployment.
6. Benchmark cold boot versus warm pool and clean snapshots. Snapshots can carry session memory/credentials, have compatibility constraints and require vsock reconnection testing. Only share **clean pre-user snapshots**.

This is a second provider under the same interface, not a rewrite of portal tools.

## 7. Measure, validate, decide

Compare ordinary rootless Podman, Podman+krun and Firecracker on identical shell/python workloads:

- Cold first-command latency, warm command latency, p50/p95/p99 over at least 100 launches and 100 warm commands.
- Actual **host RSS/PSS and committed guest memory**, CPU and disk overhead, not merely VM-configured RAM; idle density at 1, 10, 50 and 100 sandboxes.
- File writes/reads, process spawning, Git operations, cancellation, workspace/artifact restore and host restart/reconciliation.
- Session locality, tool-heavy agent turn latency, queue time, I/O saturation and fairness across users.
- Negative security tests: cross-session file access, symlink escapes, host filesystem or KVM/Podman socket access, blocked outbound traffic, host process access, oversized logs and fork bombs.
- Auditing, trace propagation, hard termination, orphan cleanup, authorization revocation, artifact digests and idempotency.

Record CPU generation, Linux kernel, KVM/nested settings, distro, SELinux/AppArmor, rootless status, exact software/image versions and image digest. **No performance measurements have been executed yet.** Do not choose capacity targets before collecting real results.

**Go/no-go:** prefer the simplest provider satisfying the actual isolation threat model, verified egress denial, workspace segregation, resource limits, cancellation and measured agent latency. Never fall back silently to a weaker sandbox when KVM capacity is exhausted.

## 8. Concrete sequence

1. Check a dedicated CPU/RAM host has /dev/kvm permission and approved pinned krun-enabled packages.
2. Run the krun file-writing smoke test and verify a **different guest kernel**.
3. Test **denied networking** with security/infra approval. If TSI cannot be controlled robustly, stop krun production evaluation.
4. Build the small SandboxExecutionProvider interface and a mock/baseline driver.
5. Connect one LangGraph tool and one Copilot SDK tool callback to the same remote sandbox interface.
6. Test 10 concurrent real agent sessions, then 50/100 shell-tool-only sessions; profile guest/host memory and cold/warm latency.
7. Repeat with a Firecracker guest + vsock command supervisor, no guest NIC and per-VM filesystem.
8. Pick a provider, then add admission control, authenticated controller access through :9080, audit and operator metrics.

## 9. Primary references

- crun krun manual (configuration, memory minimum, network modes): https://github.com/containers/crun/blob/main/krun.1.md
- libkrun (stable branch warning): https://github.com/libkrun/libkrun
- krunvm alternative: https://github.com/libkrun/krunvm
- Firecracker KVM and jailer guide: https://github.com/firecracker-microvm/firecracker/blob/main/docs/getting-started.md
- Firecracker vsock: https://github.com/firecracker-microvm/firecracker/blob/main/docs/vsock.md
- Firecracker snapshotting: https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md
- Kata Firecracker versus Cloud Hypervisor: https://github.com/kata-containers/kata-containers/blob/main/docs/design/virtualization.md
- Cloud Hypervisor: https://github.com/cloud-hypervisor/cloud-hypervisor
