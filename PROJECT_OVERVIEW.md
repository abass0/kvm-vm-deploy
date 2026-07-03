# Red Hat Automation Platform - KVM VM Provisioning

## Project Summary

This is a foundational Ansible project for Red Hat Automation Platform that provisions RHEL 9 virtual machines from a golden QCOW2 image on a KVM/libvirt hypervisor.

## Project Structure

```
ansible/
├── ansible.cfg                          # Ansible configuration
├── PROJECT_OVERVIEW.md                  # This file
├── QUICKSTART.md                        # Quick start guide
│
├── inventory/
│   ├── hosts.ini                        # Hypervisor inventory
│   └── host_vars/
│       └── kvm-host-01.yml              # Example host variables
│
├── playbooks/
│   ├── provision_vm.yml                 # Single VM provisioning playbook
│   └── provision_multiple_vms.yml       # Multiple VM provisioning playbook
│
└── roles/
    └── kvm_vm/                          # VM provisioning role
        ├── README.md                    # Complete role documentation
        ├── defaults/
        │   └── main.yml                 # Default variables
        ├── tasks/
        │   ├── main.yml                 # Main task orchestration
        │   ├── validate.yml             # Variable validation
        │   ├── prepare_disk.yml         # Clone QCOW2 image
        │   ├── cloud_init.yml           # Generate cloud-init files
        │   ├── define_vm.yml            # Define VM in libvirt
        │   └── start_vm.yml             # Start VM and wait
        ├── templates/
        │   ├── user-data.j2             # Cloud-init user-data
        │   ├── meta-data.j2             # Cloud-init meta-data
        │   ├── network-config.j2        # Cloud-init network config
        │   └── vm.xml.j2                # Libvirt domain XML
        ├── handlers/
        │   └── main.yml                 # Handlers (empty)
        └── meta/
            └── main.yml                 # Role metadata
```

## Role: kvm_vm

### Purpose

Provision a RHEL 9 virtual machine from a golden QCOW2 image. Nothing more, nothing less.

### Workflow

```
Golden Image → Clone → Cloud-init → Define VM → Power On → Done
```

### What It Does

- ✅ Validates required variables
- ✅ Clones golden QCOW2 image using backing file
- ✅ Generates cloud-init configuration (user-data, meta-data, network-config)
- ✅ Creates libvirt domain XML from template
- ✅ Defines VM in libvirt
- ✅ Starts the VM
- ✅ Waits until VM is running
- ✅ Supports both DHCP and static IP configuration

### What It Does NOT Do

- ❌ Install packages
- ❌ Register to Satellite or Insights
- ❌ Wait for cloud-init completion
- ❌ Wait for SSH availability
- ❌ Configure operating system
- ❌ Post-provisioning tasks
- ❌ Storage pool management
- ❌ Network management
- ❌ VM lifecycle management (destroy, snapshot, etc.)

### Key Features

- **Idempotent**: Safe to run multiple times
- **Clean code**: Well-documented, follows Red Hat best practices
- **Modular**: Tasks split into logical files
- **Configurable**: All variables in defaults/main.yml
- **Production-ready**: Includes validation, error handling, proper permissions

## Usage Examples

### Provision Single VM with DHCP

```bash
ansible-playbook playbooks/provision_vm.yml \
  -e vm_name=web-server-01 \
  -e vm_memory_mb=4096 \
  -e vm_vcpus=2 \
  -e vm_ssh_public_key="$(cat ~/.ssh/id_rsa.pub)"
```

### Provision VM with Static IP

```bash
ansible-playbook playbooks/provision_vm.yml \
  -e vm_name=db-server-01 \
  -e vm_memory_mb=8192 \
  -e vm_vcpus=4 \
  -e use_dhcp=false \
  -e vm_ip=192.168.122.100 \
  -e vm_gateway=192.168.122.1 \
  -e vm_ssh_public_key="$(cat ~/.ssh/id_rsa.pub)"
```

### Provision Multiple VMs from Inventory

Define VMs in `inventory/host_vars/kvm-host-01.yml`, then:

```bash
ansible-playbook playbooks/provision_multiple_vms.yml
```

## Automation Controller Integration

This project is designed for Red Hat Automation Controller:

1. **Project**: Sync this Git repository
2. **Job Template**: Use `playbooks/provision_vm.yml`
3. **Survey**: Enable survey for self-service VM provisioning
4. **Credentials**: SSH credential with sudo access to hypervisor

See `roles/kvm_vm/README.md` for complete survey configuration JSON.

## Requirements

### Hypervisor

- Fedora Linux (or RHEL/CentOS)
- KVM/libvirt installed and configured
- Required packages: `qemu-img`, `virsh`, `cloud-localds`
- Default libvirt network configured
- SSH access from Automation Controller
- Sudo privileges

### Required Directories

```bash
/var/lib/libvirt/images/templates/    # Golden images
/var/lib/libvirt/images/instances/    # VM disks
/var/lib/libvirt/cloud-init/          # Cloud-init ISOs
```

### Golden Image

- RHEL 9 QCOW2 image
- Cloud-init installed and enabled
- Located at: `/var/lib/libvirt/images/templates/rhel9-golden.qcow2`

## Variables

### Required

- `vm_name`: Unique VM identifier
- `vm_ssh_public_key`: SSH public key for VM access

### Optional (with defaults)

- `vm_memory_mb`: 2048
- `vm_vcpus`: 2
- `vm_disk_size`: 20G
- `vm_hostname`: (defaults to vm_name)
- `vm_domain`: lab.local
- `vm_network`: default
- `use_dhcp`: true
- `autostart`: false

### Static IP Configuration

When `use_dhcp: false`:
- `vm_ip`: IP address
- `vm_prefix`: Network prefix (e.g., 24)
- `vm_gateway`: Default gateway
- `vm_dns_servers`: List of DNS servers

See `roles/kvm_vm/defaults/main.yml` for complete variable list.

## Documentation

- **QUICKSTART.md**: Quick start guide with examples
- **roles/kvm_vm/README.md**: Complete role documentation
  - Requirements
  - Variables
  - Examples
  - Automation Controller survey configuration
  - Troubleshooting

## Design Principles

### Simplicity First

This is an MVP implementation focused on one task: provisioning VMs. It intentionally avoids:
- Over-engineering
- Unnecessary abstractions
- Feature creep
- Premature optimization

### Red Hat Best Practices

- Follows Ansible best practices
- Uses FQCN for modules
- Proper use of `changed_when` and `failed_when`
- Idempotent operations
- Clear task names
- Modular task organization

### Clean Code

- Readable and maintainable
- Well-documented
- Consistent formatting
- Descriptive variable names
- Comments only where necessary

## Success Criteria

The role succeeds when:
1. ✅ VM disk created from golden image
2. ✅ Cloud-init ISO generated
3. ✅ VM defined in libvirt
4. ✅ VM state is "running"

That's it. No waiting for SSH, no OS configuration, no post-provisioning.

## Next Steps

After provisioning a VM with this role:

1. Wait for cloud-init to complete (~1-2 minutes)
2. SSH to the VM: `ssh ansible@VM_IP`
3. Apply additional configuration using other roles
4. Register to Satellite (separate role/playbook)
5. Install and configure applications (separate roles)

## Contributing

This is a home lab project. Keep it simple, focused, and maintainable.

When adding features:
- Follow existing patterns
- Update documentation
- Test for idempotency
- Don't break the "one responsibility" principle

## License

MIT

## Support

This is a foundational role for home lab environments. For issues or questions, refer to the README.md files.
