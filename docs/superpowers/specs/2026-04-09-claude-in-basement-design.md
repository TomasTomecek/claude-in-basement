# Claude in Basement — Design Spec

**Date:** 2026-04-09
**Status:** Approved

## Overview

An Ansible playbook that provisions a libvirt/KVM VM running Fedora 43, installs Claude Code inside it with Vertex AI authentication, applies network isolation so the VM cannot reach the host LAN, and prints an SSH command so the user can connect and use Claude Code interactively.

---

## Goals

- Contain Claude Code inside a VM so it cannot affect the host system
- Network-isolate the VM (internet access only, no host LAN access)
- Support configurable shared folder mounts between host and VM
- Install Claude Code and gcloud with Vertex AI ADC credentials inside the VM
- Output an SSH command at the end so the user can connect immediately

## Non-Goals

- Auto-starting Claude Code on login
- Snapshotting or lifecycle management beyond initial provisioning
- Multi-VM setups
- macOS or Windows host support

---

## Architecture

Two Ansible plays in sequence:

```
Play 1 (localhost): libvirt_vm role
  → Downloads Fedora 43 cloud image
  → Creates and customizes VM disk (cloud-init or virt-customize)
  → Defines libvirt network with NAT isolation
  → Defines and starts the VM
  → Waits for SSH to become available
  → Registers VM IP, adds host to in-memory inventory

Play 2 (vm host): claude_setup role
  → Installs gcloud CLI from Google's RPM repo
  → Copies ADC credentials from host to VM
  → Installs Node.js via dnf
  → Installs Claude Code via npm
  → Configures GOOGLE_APPLICATION_CREDENTIALS env var

playbook.yml: prints SSH command at end
```

---

## Components

### `playbook.yml`

Entry point. Runs both plays sequentially. After both plays complete, uses `debug` to print the SSH command: `ssh fedora@<vm_ip>`.

### `inventory/hosts.yml`

Static inventory targeting `localhost` for play 1. Play 2 uses `add_host` to add the new VM dynamically — no file changes needed.

### `vars/config.yml`

User-editable configuration. Included by `playbook.yml` via `vars_files`. Contents:

```yaml
# Path to your SSH public key (added to VM's authorized_keys)
ssh_public_key_path: "~/.ssh/id_rsa.pub"

# Shared mounts: list of {host_path, guest_path} pairs
# Leave empty to disable shared mounts
shared_mounts: []
# Example:
# shared_mounts:
#   - host_path: /home/user/projects
#     guest_path: /home/fedora/projects

# Path to gcloud ADC credentials on host
adc_credentials_path: "~/.config/gcloud/application_default_credentials.json"
```

### `roles/libvirt_vm/`

**Purpose:** Create and start the VM.

**Tasks:**
1. Assert required packages are present on host (`virt-install`, `libvirt`, `qemu-kvm`, `python3-libvirt`, `cloud-utils`)
2. Download Fedora 43 Cloud Base qcow2 image to `/var/lib/libvirt/images/` (idempotent — skip if exists)
3. Create a VM-specific copy of the image (resized to 20GB)
4. Generate a cloud-init `user-data` ISO: sets hostname, creates `fedora` user, injects SSH public key, enables passwordless sudo
5. Define a libvirt NAT network (`claude-net`) with firewall rules blocking access to host LAN (iptables `FORWARD` drop rules for RFC-1918 ranges except the VM's own subnet)
6. Define and start the VM via `community.libvirt.virt`
7. Wait for port 22 to be reachable
8. Fetch the VM's IP from `virsh domifaddr`
9. Configure shared mounts via virtiofs filesystem devices (when `shared_mounts` is non-empty)
10. Register VM IP and add to in-memory inventory as group `vm`

**Defaults (`defaults/main.yml`):**
```yaml
vm_name: claude-basement
vm_vcpus: 2
vm_ram_mb: 4096
vm_disk_gb: 20
vm_network_name: claude-net
vm_network_subnet: "192.168.150.0/24"
fedora_image_url: "https://download.fedoraproject.org/pub/fedora/linux/releases/43/Cloud/x86_64/images/Fedora-Cloud-Base-Generic.x86_64-43.qcow2"
```

**Templates:**
- `network.xml.j2` — libvirt NAT network XML with the configured subnet

### `roles/claude_setup/`

**Purpose:** Provision Claude Code and Vertex AI credentials inside the VM.

**Tasks:**
1. Add Google Cloud SDK RPM repo
2. Install `google-cloud-cli` via dnf
3. Install `nodejs` and `npm` via dnf
4. Install Claude Code globally: `npm install -g @anthropic-ai/claude-code`
5. Create `~/.config/gcloud/` directory on VM
6. Copy ADC credentials file from host path (`adc_credentials_path`) to `~/.config/gcloud/application_default_credentials.json` on VM
7. Add `GOOGLE_APPLICATION_CREDENTIALS` export to `~/.bashrc` and `~/.bash_profile`

**Defaults (`defaults/main.yml`):**
```yaml
claude_user: fedora
node_package: nodejs
gcloud_package: google-cloud-cli
```

---

## Data Flow

```
Host filesystem
  ├── SSH public key ──────────────────→ VM authorized_keys (via cloud-init)
  ├── ADC credentials ─────────────────→ VM ~/.config/gcloud/application_default_credentials.json
  └── shared_mounts host_paths ────────→ VM guest_paths (virtiofs)

Fedora CDN
  └── Fedora 43 Cloud qcow2 ───────────→ /var/lib/libvirt/images/

Google RPM repo (internet)
  └── google-cloud-cli ────────────────→ VM (dnf)

npm registry (internet)
  └── @anthropic-ai/claude-code ───────→ VM (npm -g)
```

---

## Network Isolation

The libvirt network uses NAT mode, giving the VM internet access through the host. Additional iptables rules (managed by Ansible) are inserted in the `FORWARD` chain to drop traffic destined for RFC-1918 host LAN ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) — except for the VM's own subnet (`vm_network_subnet`) which must remain reachable for virtiofs and SSH from the host.

Rules are inserted idempotently using the `ansible.builtin.iptables` module and persisted via `iptables-save` → `/etc/sysconfig/iptables`.

---

## Shared Mounts

When `shared_mounts` contains entries, the `libvirt_vm` role:
1. Adds a `<filesystem type='mount' accessmode='passthrough'>` virtiofs device to the VM XML for each entry
2. Adds corresponding `/etc/fstab` entries inside the VM via play 2 (`claude_setup` role) over SSH to mount them at boot

Virtiofs requires `virtiofsd` on the host (installed as a dependency check).

---

## Idempotency

- Image download is skipped if the file already exists
- VM definition is skipped if `virsh dominfo <vm_name>` succeeds
- Network definition is skipped if `virsh net-info <network_name>` succeeds
- iptables rules use `ansible.builtin.iptables` which is idempotent by default
- All dnf/npm installs use idempotent Ansible modules

Running the playbook a second time against an existing VM is safe and fast.

---

## Error Handling

- Playbook asserts host dependencies (`libvirt`, `virt-install`, `qemu-kvm`, `virtiofsd`) before doing any work — fails fast with a clear message
- SSH wait task has a configurable timeout (default 5 minutes) with a meaningful failure message
- ADC credentials copy task fails explicitly if the source file does not exist on the host

---

## Testing

Manual verification steps after provisioning:
1. SSH into VM using the printed command — confirms connectivity
2. Run `claude --version` inside VM — confirms Claude Code installed
3. Run `gcloud auth application-default print-access-token` inside VM — confirms ADC credentials work
4. From inside VM, `ping 192.168.1.1` (host LAN) should fail — confirms network isolation
5. `curl https://example.com` from inside VM should succeed — confirms internet access
6. If shared mounts configured, verify the mount point is visible inside VM

---

## Ansible Collections Required

- `community.libvirt` — VM and network management
- `ansible.builtin` — standard modules (file, copy, template, iptables, wait_for, debug)

Install via: `ansible-galaxy collection install community.libvirt`
