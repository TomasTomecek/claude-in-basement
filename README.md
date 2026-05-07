# Claude living in my basement

This is a simple project to lock down Claude Code in a VM so it cannot disrupt anything on the host system.
Let Claude Code happily work autonomously in the basement.

<img src="Gemini_Generated_pic.png" width="600">

## What it does

- Provisions a **Fedora 43 KVM VM** via libvirt using a cloud image
- **Network-isolates** the VM: internet access works, host LAN is blocked
- Installs **Claude Code** and **Google Cloud CLI** inside the VM
- Configures **Vertex AI** credentials so Claude can use Anthropic's models
- Auto-generates an **SSH key pair** (stored in `keys/`, never committed)
- Prints an **SSH command** at the end so you can connect immediately

## Requirements

Host packages (Fedora/RHEL):

```bash
sudo dnf install virt-install libvirt qemu-kvm python3-libvirt cloud-utils virtiofsd
sudo systemctl enable --now libvirtd
```

Ansible and collections:

```bash
pip install ansible
ansible-galaxy collection install -r requirements.yml
```

## Configuration

Edit `vars/config.yml` before running:

```yaml
# Required: your Vertex AI GCP project ID
google_project_id: "itpc-gcp-global-eng-claude"

# Optional: Vertex AI region (default: global)
google_cloud_location: "global"

# Optional: path to your gcloud ADC credentials on the host
# Run 'gcloud auth application-default login' first if this doesn't exist
adc_credentials_path: "~/.config/gcloud/application_default_credentials.json"

# Optional: share directories from host into the VM
shared_mounts: []
# Example:
# shared_mounts:
#   - host_path: /home/user/projects
#     guest_path: /home/fedora/projects
```

## Usage

```bash
ansible-playbook playbook.yml --ask-become-pass
```

The playbook will:

1. Generate an SSH key pair at `keys/id_rsa` (gitignored)
2. Download the Fedora 43 Cloud Base image
3. Create the VM disk, cloud-init ISO, and libvirt network
4. Start the VM and wait for SSH to be available
5. Install gcloud, Node.js, and Claude Code inside the VM
6. Copy your Vertex AI credentials into the VM
7. Print the SSH command to connect

## Connecting

Use the SSH command printed at the end of the playbook run:

```
ssh -i keys/id_rsa fedora@<vm-ip>
```

Inside the VM, Claude Code is ready with Vertex AI configured:

```bash
claude
```

## Network isolation

The VM can reach the internet but cannot access your host LAN. A libvirt network hook inserts iptables rules on network start:

- `ACCEPT` traffic to the VM's own subnet (gateway access)
- `DROP` traffic to `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`

Rules are applied idempotently and re-applied automatically when the libvirt network restarts.

## VM defaults

| Setting | Default |
|---------|---------|
| vCPUs | 2 |
| RAM | 4 GB |
| Disk | 20 GB |
| Network subnet | 192.168.150.0/24 |
| OS | Fedora 43 |

Override any default by adding a variable to `vars/config.yml` (see `roles/libvirt_vm/defaults/main.yml` for all available variables).

## Re-running

The playbook is idempotent. Re-running against an existing VM skips image download, disk creation, VM definition, and network setup. Only the Claude Code and gcloud configuration steps run again (gcloud and dnf installs are no-ops if already installed; Claude Code will upgrade to the latest version).

To reprovision from scratch, delete the VM disk:

```bash
sudo virsh destroy claude-basement
sudo virsh undefine claude-basement --remove-all-storage
sudo virsh net-destroy claude-net
sudo virsh net-undefine claude-net
```

Then re-run the playbook.
