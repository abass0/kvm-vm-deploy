# Ansible Role: kvm_vm

A foundational Ansible role for provisioning RHEL 9 virtual machines from a golden QCOW2 image on a KVM/libvirt hypervisor.

## Purpose

This role performs **one task only**: provision a VM from an existing golden image and power it on.

It does **NOT**:
- Install packages
- Register to Satellite or Insights
- Wait for cloud-init completion
- Configure the operating system
- Perform post-provisioning tasks
- Manage storage pools or networks
- Handle VM lifecycle beyond initial creation

## Workflow

```
Golden QCOW2 Image
      ↓
Clone using backing file
      ↓
Generate cloud-init files
      ↓
Define VM in libvirt
      ↓
Power on VM
      ↓
Done
```

## Requirements

### Hypervisor

- Fedora Linux (or RHEL/CentOS)
- KVM/libvirt installed and configured
- `qemu-img`, `virsh`, `cloud-localds` available
- Default libvirt network configured
- SSH access from Automation Controller
- `become` privileges

### Directories

The following directories must exist on the hypervisor:

```
/var/lib/libvirt/images/templates/   # Golden images
/var/lib/libvirt/images/instances/   # VM disks
/var/lib/libvirt/cloud-init/         # Cloud-init ISOs
```

### Golden Image

A RHEL 9 QCOW2 image with cloud-init installed, located at:

```
/var/lib/libvirt/images/templates/rhel9-golden.qcow2
```

## Role Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `vm_name` | Unique VM identifier | `web-server-01` |
| `vm_ssh_public_key` | SSH public key for VM user | `ssh-rsa AAAAB3...` |

### VM Resources

| Variable | Default | Description |
|----------|---------|-------------|
| `vm_memory_mb` | `2048` | Memory in megabytes |
| `vm_vcpus` | `2` | Number of virtual CPUs |
| `vm_disk_size` | `20G` | Disk size (10G, 50G, etc.) |

### VM Identity

| Variable | Default | Description |
|----------|---------|-------------|
| `vm_hostname` | `{{ vm_name }}` | VM hostname |
| `vm_domain` | `lab.local` | DNS domain |
| `vm_user` | `ansible` | Primary user account |

### Source Image

| Variable | Default | Description |
|----------|---------|-------------|
| `vm_image` | `/var/lib/libvirt/images/templates/rhel9-golden.qcow2` | Path to golden image |

### Network Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `vm_network` | `default` | Libvirt network name |
| `use_dhcp` | `true` | Use DHCP for IP addressing |

#### Static IP Configuration

Set `use_dhcp: false` and configure:

| Variable | Required When | Description |
|----------|---------------|-------------|
| `vm_ip` | Static IP | IP address (e.g., `192.168.122.100`) |
| `vm_prefix` | Static IP | Network prefix (e.g., `24`) |
| `vm_gateway` | Static IP | Default gateway IP |
| `vm_dns_servers` | Static IP | List of DNS servers |

### VM Behavior

| Variable | Default | Description |
|----------|---------|-------------|
| `autostart` | `false` | Start VM automatically on host boot |

### Internal Variables

Do not override these unless necessary:

| Variable | Default | Description |
|----------|---------|-------------|
| `vm_disk_path` | `/var/lib/libvirt/images/instances/{{ vm_name }}.qcow2` | VM disk path |
| `vm_cloud_init_dir` | `/var/lib/libvirt/cloud-init/{{ vm_name }}` | Cloud-init config directory |
| `vm_cloud_init_iso` | `/var/lib/libvirt/cloud-init/{{ vm_name }}.iso` | Cloud-init ISO path |

## Directory Structure

```
roles/kvm_vm/
├── defaults/
│   └── main.yml              # Default variables
├── tasks/
│   ├── main.yml              # Main task orchestration
│   ├── validate.yml          # Variable validation
│   ├── prepare_disk.yml      # Clone QCOW2 image
│   ├── cloud_init.yml        # Generate cloud-init files
│   ├── define_vm.yml         # Define VM in libvirt
│   └── start_vm.yml          # Start VM and wait
├── templates/
│   ├── user-data.j2          # Cloud-init user-data
│   ├── meta-data.j2          # Cloud-init meta-data
│   ├── network-config.j2     # Cloud-init network config
│   └── vm.xml.j2             # Libvirt domain XML
├── handlers/
│   └── main.yml              # Handlers (unused)
├── meta/
│   └── main.yml              # Role metadata
└── README.md                 # This file
```

## Dependencies

None.

## Example Inventory

### Static Inventory (DHCP)

```ini
[hypervisors]
kvm-host-01 ansible_host=192.168.1.10 ansible_user=ansible

[hypervisors:vars]
ansible_python_interpreter=/usr/bin/python3
```

### Host Variables (DHCP Example)

`host_vars/kvm-host-01.yml`:

```yaml
---
vms_to_provision:
  - name: web-server-01
    memory_mb: 4096
    vcpus: 2
    disk_size: 50G
    ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC..."
  
  - name: db-server-01
    memory_mb: 8192
    vcpus: 4
    disk_size: 100G
    ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC..."
```

### Host Variables (Static IP Example)

`host_vars/kvm-host-01.yml`:

```yaml
---
vms_to_provision:
  - name: web-server-01
    memory_mb: 4096
    vcpus: 2
    disk_size: 50G
    use_dhcp: false
    ip: 192.168.122.100
    prefix: 24
    gateway: 192.168.122.1
    dns_servers:
      - 192.168.122.1
      - 8.8.8.8
    ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC..."
```

## Example Playbooks

### Single VM with DHCP

```yaml
---
- name: Provision VM with DHCP
  hosts: hypervisors
  become: true

  roles:
    - role: kvm_vm
      vars:
        vm_name: web-server-01
        vm_memory_mb: 4096
        vm_vcpus: 2
        vm_disk_size: 50G
        vm_ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
```

### Single VM with Static IP

```yaml
---
- name: Provision VM with static IP
  hosts: hypervisors
  become: true

  roles:
    - role: kvm_vm
      vars:
        vm_name: db-server-01
        vm_memory_mb: 8192
        vm_vcpus: 4
        vm_disk_size: 100G
        use_dhcp: false
        vm_ip: 192.168.122.100
        vm_prefix: 24
        vm_gateway: 192.168.122.1
        vm_dns_servers:
          - 192.168.122.1
          - 8.8.8.8
        vm_ssh_public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        autostart: true
```

### Multiple VMs from Inventory

```yaml
---
- name: Provision multiple VMs
  hosts: hypervisors
  become: true

  tasks:
    - name: Provision each VM
      ansible.builtin.include_role:
        name: kvm_vm
      vars:
        vm_name: "{{ item.name }}"
        vm_memory_mb: "{{ item.memory_mb }}"
        vm_vcpus: "{{ item.vcpus }}"
        vm_disk_size: "{{ item.disk_size }}"
        use_dhcp: "{{ item.use_dhcp | default(true) }}"
        vm_ip: "{{ item.ip | default('') }}"
        vm_prefix: "{{ item.prefix | default(24) }}"
        vm_gateway: "{{ item.gateway | default('') }}"
        vm_dns_servers: "{{ item.dns_servers | default(['8.8.8.8', '8.8.4.4']) }}"
        vm_ssh_public_key: "{{ item.ssh_key }}"
      loop: "{{ vms_to_provision }}"
      loop_control:
        label: "{{ item.name }}"
```

## Example Automation Controller Survey

Use this survey configuration in Red Hat Automation Controller to allow users to provision VMs via a self-service form.

### Survey Configuration (JSON)

```json
{
  "name": "VM Provisioning Survey",
  "description": "Provision a RHEL 9 VM from golden image",
  "spec": [
    {
      "question_name": "VM Name",
      "question_description": "Unique name for the virtual machine",
      "required": true,
      "type": "text",
      "variable": "vm_name",
      "min": 1,
      "max": 64,
      "default": "",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "Hostname",
      "question_description": "VM hostname (defaults to VM name)",
      "required": false,
      "type": "text",
      "variable": "vm_hostname",
      "min": 0,
      "max": 64,
      "default": "",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "Memory (MB)",
      "question_description": "Memory in megabytes",
      "required": true,
      "type": "integer",
      "variable": "vm_memory_mb",
      "min": 1024,
      "max": 32768,
      "default": "2048",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "vCPUs",
      "question_description": "Number of virtual CPUs",
      "required": true,
      "type": "integer",
      "variable": "vm_vcpus",
      "min": 1,
      "max": 16,
      "default": "2",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "Disk Size",
      "question_description": "Disk size (e.g., 20G, 50G, 100G)",
      "required": true,
      "type": "text",
      "variable": "vm_disk_size",
      "min": 0,
      "max": 10,
      "default": "20G",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "Network Type",
      "question_description": "IP addressing method",
      "required": true,
      "type": "multiplechoice",
      "variable": "use_dhcp",
      "min": 0,
      "max": 0,
      "default": "true",
      "choices": "true\nfalse",
      "new_question": true
    },
    {
      "question_name": "Static IP Address",
      "question_description": "IP address (only if DHCP is false)",
      "required": false,
      "type": "text",
      "variable": "vm_ip",
      "min": 0,
      "max": 15,
      "default": "",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "Network Prefix",
      "question_description": "Network prefix length (only if DHCP is false)",
      "required": false,
      "type": "integer",
      "variable": "vm_prefix",
      "min": 1,
      "max": 32,
      "default": "24",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "Default Gateway",
      "question_description": "Default gateway IP (only if DHCP is false)",
      "required": false,
      "type": "text",
      "variable": "vm_gateway",
      "min": 0,
      "max": 15,
      "default": "",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "SSH Public Key",
      "question_description": "SSH public key for VM access",
      "required": true,
      "type": "textarea",
      "variable": "vm_ssh_public_key",
      "min": 100,
      "max": 2000,
      "default": "",
      "choices": "",
      "new_question": true
    },
    {
      "question_name": "Autostart",
      "question_description": "Start VM automatically on host boot",
      "required": true,
      "type": "multiplechoice",
      "variable": "autostart",
      "min": 0,
      "max": 0,
      "default": "false",
      "choices": "true\nfalse",
      "new_question": true
    }
  ]
}
```

### Survey Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| VM Name | Text | Yes | - | Unique VM identifier |
| Hostname | Text | No | (vm_name) | VM hostname |
| Memory (MB) | Integer | Yes | 2048 | Memory allocation |
| vCPUs | Integer | Yes | 2 | CPU count |
| Disk Size | Text | Yes | 20G | Disk size |
| Network Type | Choice | Yes | true | DHCP or static |
| Static IP Address | Text | No* | - | IP address |
| Network Prefix | Integer | No* | 24 | CIDR prefix |
| Default Gateway | Text | No* | - | Gateway IP |
| SSH Public Key | Textarea | Yes | - | User's SSH public key |
| Autostart | Choice | Yes | false | Enable autostart |

*Required when Network Type is `false` (static IP)

### Job Template Configuration

In Automation Controller, create a Job Template with:

- **Name**: Provision KVM VM
- **Job Type**: Run
- **Inventory**: Your hypervisor inventory
- **Project**: Your Ansible project
- **Playbook**: `playbooks/provision_vm.yml`
- **Credentials**: SSH credential for hypervisor
- **Privilege Escalation**: Enabled
- **Survey**: Enabled (use survey above)
- **Prompt on Launch**: Variables (optional)

## Idempotency

This role is idempotent and safe to run multiple times:

- Skips disk creation if disk already exists
- Skips VM definition if VM is already defined
- Only starts VM if not already running
- Uses `changed_when` and `failed_when` appropriately

## Cloud-init Configuration

The role configures cloud-init with:

- Hostname and FQDN
- User account with SSH key
- Passwordless sudo access
- SSH key-based authentication only
- Root login disabled
- Password authentication disabled
- Filesystem growth to disk size
- UTC timezone

Cloud-init does **NOT**:

- Install packages
- Update packages
- Run custom scripts
- Register to Satellite
- Configure applications

## Libvirt Configuration

The generated VM uses:

- **Machine type**: Q35
- **CPU mode**: host-passthrough
- **Disk bus**: VirtIO (for performance)
- **Network**: VirtIO
- **Storage format**: QCOW2 with backing file
- **Console**: Serial console available
- **Graphics**: VNC (localhost only)

## Success Criteria

The role succeeds when:

1. VM disk is created from golden image
2. Cloud-init ISO is generated
3. VM is defined in libvirt
4. VM state is "running"

The role does **NOT** wait for:

- SSH to become available
- Cloud-init to complete
- Network configuration to finish
- Applications to start

## Troubleshooting

### VM fails to start

Check libvirt logs:
```bash
journalctl -u libvirtd
virsh dominfo <vm_name>
```

### Cloud-init not applying

Verify cloud-init ISO is attached:
```bash
virsh dumpxml <vm_name> | grep -A 5 cdrom
```

### Network not configured

For static IP, verify network-config was generated:
```bash
cat /var/lib/libvirt/cloud-init/<vm_name>/network-config
```

### SSH key not working

Verify key in user-data:
```bash
cat /var/lib/libvirt/cloud-init/<vm_name>/user-data
```

## License

MIT

## Author Information

Created for Red Hat Automation Platform home lab environment.
