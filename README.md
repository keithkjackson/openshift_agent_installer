# OpenShift Agent-Based Installer — Ansible Role

An Ansible role that automates **OpenShift agent-based installation**: it generates a bootable agent ISO and optionally mounts it to **Dell servers via iDRAC** using the **Redfish** API. It is designed for bare-metal and disconnected/air-gapped environments.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Variable Reference](#variable-reference)
- [Example Scenarios](#example-scenarios)
- [Execution and Tags](#execution-and-tags)
- [Cleanup](#cleanup)
- [Next Steps After Deployment](#next-steps-after-deployment)
- [Troubleshooting](#troubleshooting)

---

## Overview

The role performs two main phases:

1. **ISO generation** — Builds a bootable OpenShift agent-based installation ISO from `install-config.yaml` and `agent-config.yaml` (FIPS mode supported).
2. **ISO mounting** — Mounts the generated ISO to physical servers via Dell iDRAC virtual media (Redfish) and can set boot order and power state.

You can run both phases together or enable only one (e.g. generate ISO only, or mount an existing ISO).

---

## Features

- **Agent-based installer** — Uses `openshift-install agent create image` for ISO creation.
- **FIPS mode** — Configurable FIPS-enabled installs.
- **Bare metal & platforms** — Bare metal (with optional provisioning network), dual-stack, and vSphere-friendly install configs.
- **Disconnected / mirror** — Supports `imageContentSources` and `additionalTrustBundle` for air-gapped or mirror registries.
- **Flexible networking** — Single NIC, bonds, VLANs; NMState-style `networkConfig` in agent host definitions.
- **iDRAC integration** — Virtual media insert/eject, one-time or persistent boot from CD, optional power cycle via Redfish.
- **Validation** — Checks pull secret, SSH key, installer binary, Python, Ansible, collections, iDRAC connectivity, and ISO HTTP URL before running.
- **Example vars** — Sample configs for compact cluster, disconnected, workers, provisioning network, and dual-stack.

---

## Requirements

### Control node

- **Ansible** 2.9 or higher (ansible-core 2.12+ recommended).
- **Python** 3.8+ with modules: `requests`, `yaml` (PyYAML).
- **OpenShift installer** — `openshift-install` binary for the target OCP version (e.g. 4.14.x), typically at `/usr/local/bin/openshift-install`.

### Ansible collections

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install ansible.posix
```

### Target environment

- **ISO generation**: Run on **localhost**; writes to `work_dir` and `iso_output_dir`.
- **ISO serving**: An **HTTP server** must serve the ISO at `iso_http_url`; the URL must be reachable from the **iDRAC network** (and from the machines that will boot the agent).
- **iDRAC**: Dell servers with iDRAC and Redfish support (virtual media, boot control, power control).

### Optional

- **Ansible Vault** for secrets (`pull_secret`, `idrac_password`, etc.) when using `vars/vault.yml` or similar.

---

## Project Structure

```text
openshift_agent_installer/
├── deploy.yml              # Example playbook (compact cluster + role)
├── defaults/
│   └── main.yml            # Role defaults (cluster, ISO, iDRAC, agent, etc.)
├── vars/
│   ├── main.yml                  # Internal role variables
│   ├── ocp-compact-cluster.yml   # Example: 3 masters, bonded NICs
│   ├── ocp-disconnected.yml      # Example: disconnected + mirror
│   ├── ocp-dual-stack.yml        # Example: dual-stack
│   ├── ocp-with-provisioning.yml # Example: provisioning network
│   └── ocp-with-workers.yml      # Example: masters + workers
├── tasks/
│   ├── main.yml                  # Entry: validation, includes, summary
│   ├── prerequisites.yml         # Binary, Python, collections, iDRAC, URL checks
│   ├── generate_iso.yml          # install-config/agent-config + create image
│   ├── generate_oc_mirror_imageset_config.yml  # Renders oc-mirror ImageSetConfiguration (v2)
│   ├── mount_iso.yml             # Loop over idrac_servers
│   ├── mount_iso_single_server.yml # Per-server Redfish virtual media + boot
│   └── cleanup.yml               # Eject media, remove work_dir/iso_output_dir
├── handlers/
│   └── main.yml            # Handler: cleanup_work_dir
├── templates/
│   ├── install-config.yaml.j2        # Renders install-config for openshift-install
│   ├── agent-config.yaml.j2          # Renders agent-config (hosts, rendezvousIP)
│   └── ImageSetConfiguration-v2.yaml.j2  # Renders oc-mirror v2 ImageSetConfiguration
└── README.md / newreadme.md
```

---

## Installation

1. Install the role (e.g. in an Ansible project or roles path):

   ```bash
   # If using as a git submodule or clone
   git clone <repo-url> roles/openshift_agent_installer
   ```

2. Install collections and download `openshift-install`:

   ```bash
   ansible-galaxy collection install community.general ansible.posix
   # Download openshift-install for your OCP version from:
   # https://mirror.openshift.com/pub/openshift-v4/clients/ocp/
   ```

3. Prepare vars (and optionally vault) for your environment (see [Configuration](#configuration)).

---

## Quick Start

1. **Set variables** — Use one of the example vars files or your own. Required in all cases:
   - `cluster_name`, `base_domain`
   - `pull_secret`, `ssh_public_key`
   - For ISO generation: `rendezvous_ip`, `agent_hosts`
   - For ISO mounting: `iso_http_url`, `idrac_servers`, `idrac_password` (or per-server)

2. **Run the example playbook** (uses compact cluster vars and optional vault):

   ```bash
   ansible-playbook deploy.yml --ask-vault-pass
   ```

   Or run the role from your own playbook:

   ```yaml
   - hosts: localhost
     connection: local
     vars_files:
       - vars/ocp-compact-cluster.yml
       - vars/vault.yml   # if using vault
     roles:
       - role: openshift_agent_installer
   ```

3. **Ensure HTTP server is serving the ISO** at `iso_http_url` (from a host reachable by iDRAC and target nodes).

4. After the playbook runs, boot the servers from the mounted ISO (or use `power_cycle_servers: true`) and follow [Next Steps After Deployment](#next-steps-after-deployment).

---

## Configuration

- **Role defaults**: `defaults/main.yml` (versions, paths, cluster size, platform, networking, agent, iDRAC, etc.).
- **Overrides**: Pass extra vars, use `vars_files` in the playbook, or group/host vars. Sensitive data should use Ansible Vault (e.g. `vars/vault.yml`).
- **Example vars**: Copy and adapt one of the `vars/ocp-*.yml` files.

Important knobs:

- **ISO only (no mount)**  
  `iso_mounting_enabled: false`  
  Only generates the ISO; no iDRAC operations.

- **Mount only (ISO already built)**  
  `iso_generation_enabled: false`  
  Only mounts existing ISO from `iso_http_url` to `idrac_servers`.

- **Preserve artifacts**  
  `preserve_work_dir: true` — keep `work_dir` after run (e.g. for `wait-for bootstrap-complete` / `install-complete`).

---

## Variable Reference

### Required (always)

| Variable         | Description                          |
|------------------|--------------------------------------|
| `cluster_name`   | OpenShift cluster name               |
| `base_domain`    | Base DNS domain                      |
| `pull_secret`    | JSON pull secret                     |
| `ssh_public_key`  | SSH public key for node access       |

### ISO generation

| Variable               | Default              | Description                                |
|------------------------|----------------------|--------------------------------------------|
| `iso_generation_enabled` | `true`             | Run ISO generation phase                   |
| `ocp_version`          | `"4.14.3"`          | OpenShift version (documentation)          |
| `ocp_installer_path`   | `/usr/local/bin/openshift-install` | Path to `openshift-install` |
| `work_dir`             | `/tmp/openshift-agent-install` | Installer work directory           |
| `iso_output_dir`       | `/tmp/openshift-iso` | Where to write the ISO                     |
| `iso_name`             | `agent.x86_64.iso`   | Output ISO filename                        |
| `fips_enabled`         | `true`               | Enable FIPS in install-config              |
| `clean_work_dir`      | `true`               | Remove work_dir before generating          |
| `rendezvous_ip`       | `""`                 | Rendezvous host IP (required for ISO)      |
| `agent_hosts`         | `[]`                 | List of agent host definitions (required for ISO) |

### Cluster and platform

| Variable                        | Default           | Description                                  |
|---------------------------------|-------------------|----------------------------------------------|
| `control_plane_replicas`        | `3`               | Number of control plane nodes                |
| `compute_replicas`              | `0`               | Number of workers (0 = compact)              |
| `platform_type`                 | `"baremetal"`     | Platform: baremetal, none, vsphere, etc.     |
| `provisioning_network`          | `"Disabled"`      | Disabled, Managed, or Unmanaged              |
| `provisioning_network_interface`| `""`              | Required if Managed/Unmanaged                |
| `provisioning_network_cidr`     | `"172.22.0.0/24"` | For Managed provisioning network             |
| `api_vip` / `api_vips`          | `""` / `[]`       | API VIP(s) (single or dual-stack)            |
| `ingress_vip` / `ingress_vips`  | `""` / `[]`       | Ingress VIP(s)                               |

#### Provisioning network modes

**Disabled** (default for disconnected environments):

- No separate provisioning network
- All communication happens on the machine network
- Recommended for most disconnected/air-gapped installations
- Simplest configuration

**Managed**:

- OpenShift manages the provisioning network
- Requires a dedicated provisioning network interface and provisioning network CIDR
- Used when you want OpenShift to handle DHCP/PXE on the provisioning network

**Unmanaged**:

- Provisioning network exists but is managed externally
- Requires a dedicated provisioning network interface
- External DHCP server manages the provisioning network
- Used when you have existing provisioning infrastructure

**Example — standard bare metal (no provisioning network):**

```yaml
platform_type: "baremetal"
provisioning_network: "Disabled"
api_vip: "192.168.1.5"
ingress_vip: "192.168.1.6"
```

### Networking (install-config)

| Variable                    | Default              | Description                    |
|-----------------------------|----------------------|--------------------------------|
| `networking_type`           | `OVNKubernetes`      | Network type                   |
| `cluster_network_cidr`     | `10.128.0.0/14`      | Cluster network CIDR           |
| `cluster_network_host_prefix` | `23`              | Host prefix                    |
| `service_network`           | `172.30.0.0/16`     | Service network                |
| `machine_network_cidr`     | `192.168.1.0/24`     | Machine network (single)       |
| `machine_networks` / `cluster_networks` / `service_networks` | `[]` | Dual-stack / multi-network |

### Disconnected / mirror

| Variable                 | Default | Description                          |
|--------------------------|---------|--------------------------------------|
| `image_content_sources`  | `[]`    | Mirror definitions for install-config |
| `additional_trust_bundle`| `""`    | CA bundle for mirror (PEM)           |

**oc-mirror v2 — `ImageSetConfiguration`** (optional): set `oc_mirror_imageset_config_enabled: true` to render `mirror.openshift.io/v2alpha1` YAML (no `storageConfig`; use `oc-mirror … --v2`). The platform channel defaults to `stable-<major>.<minor>` derived from `ocp_version` (e.g. `4.18.30` → `stable-4.18`), with `minVersion` / `maxVersion` pinned to `ocp_version`. Operators are defined in `oc_mirror_operator_catalogs` (see `defaults/main.yml`).

| Variable                         | Default | Description |
|----------------------------------|---------|-------------|
| `oc_mirror_imageset_config_enabled` | `false` | Render ImageSetConfiguration |
| `oc_mirror_imageset_config_path` | `{{ work_dir }}/ImageSetConfiguration.yaml` | Output file |
| `oc_mirror_platform_channel`     | `""`    | Override channel name; if empty, derived from `ocp_version` |
| `oc_mirror_platform_channels`    | `[]`    | Full custom `mirror.platform.channels` list (replaces auto block) |
| `oc_mirror_platform_graph`       | `true`  | Set `mirror.platform.graph` |
| `oc_mirror_platform_architectures` | `[amd64]` | Platform architectures (omit if empty) |
| `oc_mirror_operator_catalogs`    | `[]`    | List of `{ catalog, packages: [...] }` for `mirror.operators` |

### ISO mounting (iDRAC)

| Variable                | Default   | Description                                  |
|-------------------------|-----------|----------------------------------------------|
| `iso_mounting_enabled`  | `true`    | Run ISO mount phase                          |
| `iso_http_url`          | `""`      | Full HTTP URL of the ISO (required for mount) |
| `idrac_servers`         | `[]`      | List of `{ name, idrac_ip, [idrac_username], [idrac_password], [role] }` |
| `idrac_username`        | `root`    | Default iDRAC user                           |
| `idrac_password`        | `""`      | Default iDRAC password (or per-server)       |
| `idrac_validate_certs`  | `false`   | Validate iDRAC TLS certificates              |
| `mount_timeout`         | `300`     | Redfish operation timeout (seconds)          |
| `mount_retries`         | `3`       | Retries for virtual media insert             |
| `boot_from_iso`         | `true`    | Set boot to virtual CD                       |
| `boot_once`             | `false`   | One-time boot vs persistent boot order       |
| `power_cycle_servers`   | `false`   | Reboot servers after mount                    |
| `power_state_after_mount` | `"On"`  | Power state after mount (e.g. On, ForceOff)  |

### Agent hosts (`agent_hosts`)

Each item can include:

- **hostname**, **role** (`master` or `worker`)
- **rootDeviceHints** (e.g. `deviceName: "/dev/sda"`)
- **interfaces**: list of `name`, `macAddress`
- **networkConfig**: NMState-style interfaces (ethernet, bond, vlan), DNS, routes

Use **`agent_hosts`** (not `hosts`) to avoid clashing with Ansible’s `hosts` keyword.

**Example — single-interface agent host:**

```yaml
agent_hosts:
  - hostname: "master-0"
    role: "master"
    rootDeviceHints:
      deviceName: "/dev/sda"
    interfaces:
      - name: "eno1"
        macAddress: "AA:BB:CC:DD:EE:01"
    networkConfig:
      interfaces:
        - name: "eno1"
          type: "ethernet"
          state: "up"
          ipv4:
            enabled: true
            address:
              - ip: "192.168.1.100"
                prefix-length: 24
            dhcp: false
      dns-resolver:
        config:
          server:
            - "192.168.1.1"
      routes:
        config:
          - destination: "0.0.0.0/0"
            next-hop-address: "192.168.1.1"
            next-hop-interface: "eno1"
```

For bonded interfaces and VLANs, see `vars/ocp-compact-cluster.yml` and `defaults/main.yml`.

---

## Example Scenarios

- **Compact cluster (3 masters, no workers)**  
  Use `vars/ocp-compact-cluster.yml` (or deploy.yml with that vars file). Defines bonded interfaces and rendezvous IP.

- **Disconnected / air-gapped**  
  Use `vars/ocp-disconnected.yml` (or equivalent) with `image_content_sources` and `additional_trust_bundle`; pull secret for mirror registry.

- **With workers**  
  Use `vars/ocp-with-workers.yml` (or similar): add worker entries to `agent_hosts` and set `compute_replicas` if needed.

- **With provisioning network**  
  Use `vars/ocp-with-provisioning.yml`: set `provisioning_network` to Managed or Unmanaged and fill `provisioning_network_interface` and optionally `provisioning_network_cidr`.

- **Dual-stack**  
  Use `vars/ocp-dual-stack.yml`: set `api_vips`, `ingress_vips`, `machine_networks`, `cluster_networks`, `service_networks` (and matching agent host networkConfig).

---

## Execution and Tags

Run the full role (default):

```bash
ansible-playbook deploy.yml --ask-vault-pass
```

Use tags to run only part of the role:

| Tag              | Description                    |
|------------------|--------------------------------|
| `prerequisites`  | Validation and prereq checks   |
| `generate_iso`, `iso` | ISO generation only       |
| `mount_iso`, `idrac` | ISO mounting only          |
| `oc_mirror`, `imageset_config` | ImageSetConfiguration for oc-mirror v2 only |

Examples:

```bash
ansible-playbook deploy.yml --tags prerequisites
ansible-playbook deploy.yml --tags generate_iso
ansible-playbook deploy.yml --tags mount_iso
ansible-playbook deploy.yml --tags oc_mirror
```

---

## Cleanup

The role includes a **cleanup** task list that:

- Ejects virtual media from all `idrac_servers`
- Optionally removes `work_dir` and (if set) `iso_output_dir`

Run cleanup via a playbook that includes the role’s cleanup tasks, for example:

```yaml
# cleanup.yml (example)
- hosts: localhost
  connection: local
  vars_files:
    - vars/ocp-compact-cluster.yml
    - vars/vault.yml
  tasks:
    - name: Run cleanup tasks
      ansible.builtin.include_role:
        name: openshift_agent_installer
        tasks_from: cleanup.yml
```

If your playbook is structured so that the same vars are loaded, you can run:

```bash
ansible-playbook cleanup.yml --ask-vault-pass
```

Behavior:

- **preserve_work_dir: true** — Keeps `work_dir` (needed for `wait-for bootstrap-complete` / `install-complete`).
- **clean_iso_output_dir: true** — Removes `iso_output_dir` during cleanup (default false).

---

## Next Steps After Deployment

1. **Verify ISO is reachable**  
   `curl -I <iso_http_url>`

2. **Wait for bootstrap**  
   ```bash
   openshift-install agent wait-for bootstrap-complete --dir=<work_dir> --log-level=info
   ```

3. **Wait for install complete**  
   ```bash
   openshift-install agent wait-for install-complete --dir=<work_dir> --log-level=info
   ```

4. **Use the cluster**  
   ```bash
   export KUBECONFIG=<work_dir>/auth/kubeconfig
   oc get nodes
   ```

5. **Cleanup**  
   After installation is complete, run the cleanup playbook so the role ejects virtual media and optionally removes directories.

---

## Troubleshooting

| Issue | Check |
|-------|--------|
| ISO generation fails | `ocp_installer_path` exists and is executable; `work_dir`/`iso_output_dir` writable; `pull_secret` valid JSON; `ssh_public_key` format; `rendezvous_ip` and `agent_hosts` set. |
| Mount fails | `iso_http_url` returns 200 from the host running Ansible and is reachable from iDRAC network; `idrac_servers` have correct IPs and credentials; `community.general` installed; firewall allows HTTPS to iDRAC. |
| Nodes don’t boot from ISO | In iDRAC, confirm virtual media is inserted and boot order (or one-time boot) is CD; use `power_cycle_servers: true` to reboot after mount. |
| Prereq failures | Run with `--tags prerequisites` and fix reported errors (Python, Ansible version, collections, installer binary, pull secret, SSH key, iDRAC connectivity). |

For more detail on provisioning network modes (Disabled vs Managed vs Unmanaged), see the comments and examples in `defaults/main.yml` and the existing `README.md`.
