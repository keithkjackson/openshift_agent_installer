# OpenShift Agent-Based Installer Role

This Ansible role automates the generation of an OpenShift agent-based installation ISO and mounts it to Dell servers via iDRAC using the Redfish protocol. Designed for disconnected/air-gapped environments.

## Description

The role performs two primary functions:
1. **ISO Generation**: Creates a bootable OpenShift installation ISO using agent-based installer with FIPS mode enabled
2. **ISO Mounting**: Mounts the generated ISO to physical servers via Dell iDRAC using Redfish API

## Requirements

### Ansible Version
- Ansible 2.9 or higher
- ansible-core 2.12+ recommended

### Collections
```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install ansible.posix

### Platform and Provisioning Network Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `platform_type` | `"baremetal"` | Platform type (baremetal, none, vsphere, etc.) |
| `provisioning_network` | `"Disabled"` | Provisioning network mode: Disabled, Managed, or Unmanaged |
| `provisioning_network_interface` | `""` | Network interface for provisioning (required for Managed/Unmanaged) |
| `provisioning_network_cidr` | `"172.22.0.0/24"` | CIDR for provisioning network (required for Managed) |
| `api_vip` | `""` | API VIP address (required for baremetal) |
| `ingress_vip` | `""` | Ingress VIP address (required for baremetal) |
| `api_vips` | `[]` | List of API VIPs (for dual-stack) |
| `ingress_vips` | `[]` | List of Ingress VIPs (for dual-stack) |

#### Provisioning Network Modes

**Disabled** (Default for disconnected environments):
- No separate provisioning network
- All communication happens on the machine network
- Recommended for most disconnected/air-gapped installations
- Simplest configuration

**Managed**:
- OpenShift manages the provisioning network
- Requires a dedicated provisioning network interface
- Requires provisioning network CIDR
- Used when you want OpenShift to handle DHCP/PXE on provisioning network

**Unmanaged**:
- Provisioning network exists but is managed externally
- Requires a dedicated provisioning network interface
- External DHCP server manages the provisioning network
- Used when you have existing provisioning infrastructure

#### Example Configurations

**Standard Baremetal (No Provisioning Network)**:
```yaml
platform_type: "baremetal"
provisioning_network: "Disabled"
api_vip: "192.168.1.5"
ingress_vip: "192.168.1.6"

### Agent Configuration Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `rendezvous_ip` | `""` | IP of the rendezvous host (required) |
| `agent_hosts` | `[]` | List of host configurations with MAC addresses |

**Important Note**: The variable is named `agent_hosts` (not `hosts`) to avoid conflicts with Ansible's reserved `hosts` keyword.

#### Agent Hosts Configuration

Example `agent_hosts` configuration:
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
