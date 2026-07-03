# Quick Start Guide

This guide shows how to provision a VM using the `kvm_vm` role.

## Prerequisites

1. **Hypervisor Setup**
   - Fedora Linux with KVM/libvirt installed
   - Required directories exist:
     ```bash
     sudo mkdir -p /var/lib/libvirt/images/{templates,instances}
     sudo mkdir -p /var/lib/libvirt/cloud-init
     ```

2. **Golden Image**
   - RHEL 9 QCOW2 image with cloud-init installed
   - Located at: `/var/lib/libvirt/images/templates/rhel9-golden.qcow2`

3. **Automation Controller**
   - SSH access to hypervisor configured
   - Become privileges available
   - This Ansible project synced

## Provision a Single VM (DHCP)

1. **Update inventory**
   
   Edit `inventory/hosts.ini` with your hypervisor IP:
   ```ini
   [hypervisors]
   kvm-host-01 ansible_host=YOUR_HYPERVISOR_IP
   ```

2. **Run the playbook**
   ```bash
   ansible-playbook playbooks/provision_vm.yml \
     -e vm_name=test-vm-01 \
     -e vm_ssh_public_key="$(cat ~/.ssh/id_rsa.pub)"
   ```

3. **Verify VM is running**
   ```bash
   ansible hypervisors -m shell -a "virsh list --all"
   ```

## Provision a VM with Static IP

```bash
ansible-playbook playbooks/provision_vm.yml \
  -e vm_name=static-vm-01 \
  -e use_dhcp=false \
  -e vm_ip=192.168.122.100 \
  -e vm_gateway=192.168.122.1 \
  -e vm_ssh_public_key="$(cat ~/.ssh/id_rsa.pub)"
```

## Provision Multiple VMs

1. **Define VMs in inventory**
   
   Edit `inventory/host_vars/kvm-host-01.yml` and add your VMs to `vms_to_provision`

2. **Run the playbook**
   ```bash
   ansible-playbook playbooks/provision_multiple_vms.yml
   ```

## Using Automation Controller

1. **Create a Project**
   - SCM Type: Git
   - SCM URL: Your repository URL
   - SCM Branch: main

2. **Create Job Template**
   - Name: Provision KVM VM
   - Job Type: Run
   - Inventory: Your hypervisor inventory
   - Project: Your project
   - Playbook: playbooks/provision_vm.yml
   - Credentials: SSH credential for hypervisor
   - Privilege Escalation: ✓ Enabled
   - Survey: ✓ Enabled

3. **Add Survey** (see README.md for complete survey JSON)
   
   Required fields:
   - VM Name
   - Memory (MB)
   - vCPUs
   - Disk Size
   - SSH Public Key

4. **Launch Job Template**
   
   Fill out the survey and launch. The VM will be provisioned and started.

## What Gets Created

When you run the role, it creates:

```
/var/lib/libvirt/images/instances/
└── vm-name.qcow2                    # VM disk (backing file)

/var/lib/libvirt/cloud-init/
├── vm-name/
│   ├── user-data                    # Cloud-init user config
│   ├── meta-data                    # Cloud-init metadata
│   └── network-config               # Network config (if static)
└── vm-name.iso                      # Cloud-init ISO

Libvirt:
└── VM defined and running
```

## Accessing the VM

After provisioning completes:

1. **Get IP address** (DHCP VMs)
   ```bash
   virsh domifaddr vm-name
   ```

2. **SSH to the VM**
   ```bash
   ssh ansible@VM_IP
   ```

3. **Check cloud-init status** (inside VM)
   ```bash
   cloud-init status
   ```

## Common Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `vm_name` | - | **Required** - VM name |
| `vm_memory_mb` | 2048 | Memory in MB |
| `vm_vcpus` | 2 | Number of CPUs |
| `vm_disk_size` | 20G | Disk size |
| `use_dhcp` | true | Use DHCP vs static IP |
| `vm_ssh_public_key` | - | **Required** - SSH public key |
| `autostart` | false | Start VM on host boot |

## Troubleshooting

**VM not starting?**
```bash
virsh dominfo vm-name
journalctl -u libvirtd -f
```

**Can't SSH to VM?**
```bash
virsh console vm-name  # Press Enter a few times
# Check IP: ip addr show
# Check cloud-init: cloud-init status --long
```

**Need to delete a VM?**
```bash
virsh destroy vm-name    # Stop VM
virsh undefine vm-name   # Remove definition
rm /var/lib/libvirt/images/instances/vm-name.qcow2
rm -rf /var/lib/libvirt/cloud-init/vm-name*
```

## Next Steps

After the VM is provisioned and running:

1. Wait for cloud-init to complete (~1-2 minutes)
2. SSH to the VM using the configured user and SSH key
3. Apply additional configuration using other Ansible roles
4. Register to Satellite (if needed)
5. Install applications

Remember: This role only provisions the VM. Post-provisioning is done separately.
