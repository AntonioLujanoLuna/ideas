# Reproducing an Nginx + Rootless Podman + vLLM Deployment with Ansible

**Status:** Infrastructure proposal  
**Scenario:** An existing single-H100 inference server runs Nginx and vLLM in rootless Podman. A new server has two H200 GPUs and may run a different model. The aim is to reproduce the operational configuration without forcing both servers to have identical GPU or model settings.

## 1. Objective

Convert the configuration of the existing H100 server into a **declarative, version-controlled Ansible deployment**, then use it to provision and maintain the H200 server. The same playbook should support future GPU nodes.

Ansible does not automatically clone a running Linux installation. The first migration requires inspecting the existing server, identifying which settings are intentional, and encoding those settings as files, templates, variables, and tasks. Once that is done, subsequent deployments are reproducible and mostly idempotent.

The initial migration should **not modify the working H100 deployment**. Use it as the reference, deploy to the H200 node, verify the result, and only then consider bringing the H100 under Ansible management.

## 2. Proposed architecture

~~~text
                         Ansible control node
                     (laptop or management host)
                       playbooks + Git repository
                                  |
                         SSH to both nodes
                    +-------------+-------------+
                    |                           |
                H100 node                  H200 node
                 1 x H100                  2 x H200
                    |                           |
                  Nginx                       Nginx
                    |                           |
             Rootless Podman             Rootless Podman
                    |                           |
                 vLLM (TP=1)               vLLM (TP=2*)
                    |                           |
             Existing model              Selected model
~~~

*TP=2 is appropriate if one model instance is distributed across the two H200s. Two independent model instances, one per GPU, would generally use TP=1 each instead. Ansible should support either layout through host-specific configuration.

Ansible is **agentless**: only the control node needs Ansible. Managed Linux nodes ordinarily need an SSH service, a suitable Python interpreter for Python-based modules, and the permissions required by the tasks. No Kubernetes or OpenShift deployment is needed.

**Scope:** configure each node independently. A single public endpoint, health-aware balancing, request routing and cross-node KV-cache reuse are separate architectural concerns. Merely duplicating Nginx does not implement these capabilities.

## 3. What can be reused, and what must differ

| Component | Reuse across machines | Node-specific considerations |
|---|---|---|
| Nginx | Reverse-proxy template, headers, timeouts, API routing, validation | Hostnames, certificates, addresses, port conflicts |
| Podman | Rootless conventions, Quadlet/systemd service, image pull policy, mounts | Service account, UID/GID mappings, storage paths |
| vLLM | Container definition, logging, health checks, startup conventions | Model, image version, GPU count, TP, context length, memory settings |
| NVIDIA | Required-tooling checks, CDI workflow | Driver compatibility, devices and locally generated CDI specification |
| Storage | Directory structure and permissions | Available disk space, model files, cache location |
| Networking | General policy and tests | Corporate proxy, firewall, DNS, certificates, reachable ports |
| Security | Secret-management pattern and least-privilege approach | Individual credentials and access policies |

Pin the image versions used in each environment. Do not assume an H100-tested vLLM image, NVIDIA configuration or model will behave identically on H200 hardware.

For a long-context deployment, settings such as `--max-model-len`, `--kv-cache-dtype`, GPU-memory utilization, CPU offload, prefix caching and maximum concurrency should be **explicit host variables**, not buried in a copied shell command.

## 4. Does Ansible require sudo?

### 4.1 Installing Ansible on the control node

**No sudo is necessary** if Python and virtual environments are already available:

~~~bash
python3 -m venv ~/.venvs/ansible
source ~/.venvs/ansible/bin/activate
python -m pip install --upgrade pip
python -m pip install ansible
ansible --version
~~~

Only the control node needs this installation. It can be a laptop, the existing H100 node or a dedicated management server, provided it can reach the targets over SSH.

### 4.2 Permissions on the managed nodes

Installing Ansible and **using Ansible to change a machine** are different operations.

| Operation | Normally requires sudo? |
|---|---|
| Read files accessible to the SSH account | No |
| Write files in that account's home | No |
| Create or update its rootless Podman containers | No |
| Manage its user-level systemd services | No |
| Modify /etc/nginx or other protected files | Yes, unless explicitly delegated |
| Reload the system Nginx service | Usually yes |
| Install packages through the system package manager | Yes |
| Install NVIDIA drivers or NVIDIA Container Toolkit | Yes |
| Generate CDI configuration in /etc/cdi | Yes |
| Set system-wide firewall rules | Yes |
| Enable lingering for a service user | Usually administrator action |

Ansible uses **become: true** for privileged tasks (typically through sudo). This can be applied to individual tasks or roles; the entire playbook does not need to run as root. A sudo password may be requested with the Ansible option **--ask-become-pass**, provided the remote account is authorized to use sudo.

A recommended corporate arrangement is for infrastructure administrators to perform the one-time machine bootstrap and give the application deployment account control of its own files, rootless containers and user services. Privileged Nginx changes can remain in a separate, administrator-run playbook.

**Important:** logging in over SSH does not grant sudo. Ansible cannot bypass corporate permissions. If rootless Podman is not installed or GPU access has not been configured, an administrator may be required before the unprivileged deployment can run.

## 5. Inspect and record the existing H100 configuration

Run these commands on the H100 node. Execute Podman commands **as the user that owns the existing rootless container**, not through sudo.

~~~bash
# Inventory of containers, images, networking and mounts
podman ps -a
podman images
podman info
podman inspect vllm

# User-level and system-level service definitions
systemctl --user list-unit-files
systemctl --user status vllm.service
systemctl status nginx.service

# Nginx's effective configuration, including included files
sudo nginx -T

# GPU / driver information
nvidia-smi
nvidia-ctk cdi list
~~~

Adjust the container and service names to the actual deployment. Some commands may fail if the corresponding utility is not installed or if the service uses a different name; that is useful discovery information, not necessarily a problem.

Also inspect:

- The command or script that originally starts vLLM, including its image entrypoint and command.
- Podman Quadlet files, systemd units, environment files, mounted volumes, user namespaces and restart behavior.
- Nginx virtual hosts, upstreams, TLS handling, forwarded headers, streaming behavior and extended inference timeouts.
- The location of model weights and Hugging Face caches, including ownership and SELinux labeling if applicable.
- GPU access through NVIDIA Container Toolkit/CDI, plus any corporate proxy, internal image registry and CA configuration.
- Boot behavior: whether rootless systemd uses lingering and whether the service starts after a restart.

**Treat command output as sensitive.** Podman inspection can expose tokens through environment variables; Nginx output can reveal internal addresses and paths. Redact credentials before checking files into Git. Do not commit private keys, API tokens, model-access tokens or the complete unreviewed inspection output.

Before changing the old machine, retain a recoverable copy of its current configuration and record its image digest and working model settings.

## 6. Suggested Ansible repository

~~~text
ansible-vllm/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   ├── group_vars/
│   │   └── gpu_nodes.yml
│   └── host_vars/
│       ├── h100.yml
│       └── h200.yml
├── roles/
│   ├── nginx/
│   │   ├── tasks/main.yml
│   │   └── templates/vllm.conf.j2
│   ├── podman/
│   │   └── tasks/main.yml
│   ├── nvidia/
│   │   └── tasks/main.yml
│   └── vllm/
│       ├── tasks/main.yml
│       └── templates/vllm.container.j2
├── bootstrap.yml
└── deploy.yml
~~~

The **bootstrap playbook** covers authorized, privileged host preparation. The **deployment playbook** manages the application with the unprivileged service user, apart from any explicitly authorized Nginx changes.

An illustrative inventory:

~~~yaml
# inventory/hosts.yml
all:
  children:
    gpu_nodes:
      hosts:
        h100:
          ansible_host: h100-node.example.internal
          ansible_user: gpuuser
        h200:
          ansible_host: h200-node.example.internal
          ansible_user: gpuuser
~~~

These are example hostnames and account names, not discovered values. Substitute the actual SSH addresses and approved deployment account.

Keep shared parameters in **group_vars/gpu_nodes.yml**:

~~~yaml
vllm_port: 8000
vllm_max_model_len: 262144
vllm_kv_cache_dtype: fp8
vllm_gpu_memory_utilization: 0.90
~~~

And machine-dependent parameters in the corresponding **host_vars**:

~~~yaml
# inventory/host_vars/h100.yml
gpu_count: 1
vllm_tensor_parallel_size: 1
vllm_model: /srv/models/h100-model
vllm_image: registry.example.internal/vllm:approved-h100
~~~

~~~yaml
# inventory/host_vars/h200.yml
gpu_count: 2
vllm_tensor_parallel_size: 2
vllm_model: /srv/models/h200-model
vllm_image: registry.example.internal/vllm:approved-h200
~~~

Model paths and image addresses above are **illustrative configuration values**. Fill them using the existing installation and the actual selected H200 model/image. Do not choose versions merely because they appear in an example.

For two independent instances on the H200 node, model the two containers explicitly, with separate names, host ports and GPU assignments, rather than setting TP=2.

## 7. Rootless Podman: prefer a declarative Quadlet service

If the current deployment uses a manually executed Podman command, consider translating it into a [Quadlet](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) file. Quadlet lets systemd generate and manage the container service from a declarative definition.

For rootless Podman, a typical location is:

~~~text
~/.config/containers/systemd/vllm.container
~~~

A **conceptual** Jinja2 template, subject to the actual container image's entrypoint and the options confirmed on the existing node:

~~~ini
# roles/vllm/templates/vllm.container.j2
[Unit]
Description=vLLM inference server
Wants=network-online.target
After=network-online.target

[Container]
Image={{ vllm_image }}
ContainerName=vllm
PublishPort=127.0.0.1:{{ vllm_port }}:8000
AddDevice=nvidia.com/gpu=all
Volume=%h/.cache/huggingface:/data/huggingface
Environment=HF_HOME=/data/huggingface
Exec=vllm serve {{ vllm_model }} --host 0.0.0.0 --port 8000 --tensor-parallel-size {{ vllm_tensor_parallel_size }} --max-model-len {{ vllm_max_model_len }} --kv-cache-dtype {{ vllm_kv_cache_dtype }} --gpu-memory-utilization {{ vllm_gpu_memory_utilization }}

[Service]
Restart=on-failure
TimeoutStartSec=900

[Install]
WantedBy=default.target
~~~

**Do not deploy this example blindly.** Whether the Exec line should contain **vllm serve** or only its arguments depends on the image's ENTRYPOINT. Retain the current container's validated startup convention and translate its other flags, mounts, environment variables and network requirements. If using image-bundled entrypoints, test the generated unit before relying on it.

The example also assumes NVIDIA CDI devices are configured, all available GPUs are intended for the container, the model path exists **inside the container**, and the service user has an accessible Hugging Face cache. If the model is on a host-only path, add an appropriate mount. Use explicit per-device CDI assignments if multiple containers must be isolated to individual GPUs.

After Ansible installs or changes the Quadlet file, reload user-level systemd and start or restart the generated **vllm.service**. The deployment user's user manager must be available. If the service must start at boot without an interactive login, an administrator may need to enable lingering for that user.

Avoid putting Hugging Face tokens directly in Quadlet files or environment variables committed to Git. Use an approved secrets mechanism such as Ansible Vault, Podman secrets or systemd credentials, and scope access appropriately.

## 8. Nginx: preserve the existing proxy behavior

If Nginx runs as a **system service**, writing its configuration and reloading it usually requires sudo, even if rootless Podman and vLLM do not.

Capture the existing effective configuration and convert only the relevant, application-specific parts into templates. A local vLLM upstream commonly points to **127.0.0.1:8000** on each server, but existing TLS termination, path routing, authentication and gateway headers must be retained.

For inference, explicitly review:

- Long upstream read/send timeouts for large prompts and slow prefills.
- Streaming responses and any proxy-buffering behavior that interferes with them.
- Forwarded IP/host/protocol headers and API-key handling.
- Request-body size, connection limits and client disconnect behavior.
- Whether Nginx binds publicly or only to an internal load balancer.
- Whether the same certificates can legally and technically be installed on both machines.

Validate the generated configuration with **nginx -t** before reloading the service. Ideally use a handler so that a successful configuration change triggers a reload, and a failed validation leaves the running configuration untouched.

A fully unprivileged deployment can exclude the Nginx role if infrastructure administrators deploy and manage its configuration separately.

## 9. One-time bootstrap versus routine deployment

### Bootstrap: infrastructure team or authorized sudo account

- Install/verify Python, Podman, Nginx, NVIDIA drivers and NVIDIA Container Toolkit.
- Configure NVIDIA CDI on the **destination machine**, not by copying the H100's device specification.
- Create/provision the rootless service user and its subordinate UID/GID mappings if required.
- Prepare model/cache directories and filesystem permissions.
- Enable lingering if unattended rootless services must survive logout and boot.
- Configure system-level Nginx, firewall rules, proxy trust and TLS certificates.
- Confirm registry access, SSH and any outbound access required to retrieve model assets.

The NVIDIA device specification should be generated for each machine and regenerated when required by changes to GPUs, drivers or the toolkit.

### Routine deployment: unprivileged service account

- Render and install the Quadlet definition and permitted application files.
- Pull or reference an approved container image.
- Ensure application-owned cache and model locations exist.
- Reload the user systemd manager and start/restart the vLLM service when necessary.
- Verify the local inference API and report failures.

Ansible roles should be **idempotent**: repeated runs should not restart healthy services or overwrite unchanged files unnecessarily. Apply **become: true** only to tasks that genuinely require it.

## 10. Migration and validation procedure

1. **Inventory the H100 node.** Save sanitized Nginx, Podman, vLLM, NVIDIA and systemd configuration, plus a known-good deployment reference.
2. **Encode its intended state.** Create shared role templates and H100-specific variables. Keep the production service unchanged.
3. **Ask infrastructure to bootstrap the H200 node.** Ensure the approved deployment account has access to Podman, the appropriate GPUs, model storage and user services.
4. **Set H200 variables.** Select the model, image, TP=2 versus two TP=1 instances, context length, cache settings, mounts and ports.
5. **Check SSH and scope changes to H200.** Use Ansible's inventory targeting and check/diff modes where the relevant modules support them.
6. **Deploy only to H200.** Test NVIDIA CDI/container access and validate the Quadlet-generated service.
7. **Validate Nginx separately if necessary.** Test its configuration, reload it through an authorized account, then verify upstream behavior.
8. **Run functional and load tests.** Check streaming/non-streaming requests, model identity, long-context prompts, GPU utilization, memory consumption, concurrent users, logs and restart behavior.
9. **Test rollback.** Retain the last working image, host variables and configuration commit so that a failed model upgrade can be reverted.
10. **Only after H200 is stable, optionally reconcile H100 with Ansible.**

Typical control-node commands:

~~~bash
# Activate the local Ansible environment
source ~/.venvs/ansible/bin/activate

# Confirm SSH access to the target group
ansible -i inventory/hosts.yml gpu_nodes -m ansible.builtin.ping

# Examine changes to the new node (check mode has module-dependent limits)
ansible-playbook -i inventory/hosts.yml deploy.yml --limit h200 --check --diff

# Deploy to the new node only
ansible-playbook -i inventory/hosts.yml deploy.yml --limit h200
~~~

These commands assume the playbook and inventory have been implemented. A bootstrap playbook requiring administrative privileges can be run separately by an authorized operator, potentially using **--ask-become-pass**.

### Acceptance criteria

- **GPU access:** the container detects the intended GPU(s); TP matches the chosen deployment layout.
- **vLLM:** the correct model starts and answers requests through the OpenAI-compatible API.
- **Networking:** Nginx reaches the container and preserves streaming, headers and timeouts.
- **Operations:** the service restarts on failure and behaves correctly after a host reboot.
- **Security:** credentials are not committed to Git; the service runs with the intended privileges.
- **Reproducibility:** a second Ansible run produces no unexpected changes.
- **Isolation:** targeting H200 leaves the H100 production workload untouched.

## 11. Future: load balancing and cache-aware routing

Once both GPU nodes are working independently, an upstream proxy or router can distribute inference traffic. Its placement depends on the network topology and which machines can reach which services.

At that stage, decide whether the nodes expose the **same model and compatible API behavior**. A generic load balancer must not treat different models as interchangeable without an explicit routing policy. Simple Nginx balancing also has no inherent awareness of vLLM KV-cache residency. Prefix-aware routing, shared KV-cache storage or a more advanced inference scheduler would require additional design.

If the H200 server runs two independent vLLM containers, provide separate host ports (for example, 8000 and 8001) and configure the local or upstream router accordingly. If it runs a single TP=2 instance, there is only one vLLM API endpoint on that node.

The Ansible repository should make these topologies configurable, but **configuration management and inference request scheduling are separate responsibilities**.

## 12. Recommended implementation sequence

1. Start with an **application-only, non-sudo Ansible deployment** for the H200 node, assuming the infrastructure team can supply a correctly bootstrapped machine.
2. Introduce Quadlet if it simplifies the current Podman lifecycle, but preserve the known-working startup command and container settings.
3. Keep Nginx and NVIDIA bootstrap tasks separate from routine vLLM deployment and use privileged execution only when authorized.
4. Validate that the H200 configuration is reproducible before migrating the existing H100 into the same management workflow.
5. Add multi-node routing only after each node is independently reliable.

This achieves the immediate goal, **reproducing the H100 deployment on the H200 without requiring universal sudo access**, while leaving room for different models and a future multi-node inference architecture.

## References

- [Ansible installation guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [Ansible privilege escalation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)
- [Podman Quadlet and systemd units](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
- [NVIDIA Container Toolkit CDI support](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/cdi-support.html)
- [vLLM serving documentation](https://docs.vllm.ai/en/latest/serving/openai_compatible_server/)
