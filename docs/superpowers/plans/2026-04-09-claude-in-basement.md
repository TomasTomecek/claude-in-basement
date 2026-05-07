# Claude in Basement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ansible playbook that provisions a network-isolated libvirt/KVM Fedora 43 VM with Claude Code and Vertex AI credentials, printing an SSH command at the end.

**Architecture:** Two Ansible plays — play 1 (localhost) creates the VM via libvirt; play 2 (VM over SSH) installs gcloud and Claude Code. A libvirt network hook enforces network isolation by inserting iptables rules on network start.

**Tech Stack:** Ansible, libvirt/KVM, Fedora 43 Cloud qcow2, cloud-init (cloud-localds), community.libvirt / community.crypto / community.general collections.

---

## File Map

| File | Purpose |
|------|---------|
| `.gitignore` | Ignores `keys/` directory |
| `ansible.cfg` | Sets roles path, remote user, private key, disables host key checking |
| `requirements.yml` | Ansible collection dependencies |
| `inventory/hosts.yml` | Static localhost inventory for play 1 |
| `vars/config.yml` | User-editable config: shared mounts, ADC path, GCP project/region |
| `roles/libvirt_vm/defaults/main.yml` | VM defaults: name, vCPU, RAM, disk, network, image URL |
| `roles/libvirt_vm/templates/network.xml.j2` | libvirt NAT network XML |
| `roles/libvirt_vm/templates/user-data.j2` | cloud-init user-data (creates fedora user, injects SSH key) |
| `roles/libvirt_vm/templates/meta-data.j2` | cloud-init meta-data (hostname) |
| `roles/libvirt_vm/templates/network_hook.sh.j2` | libvirt network hook that inserts iptables isolation rules |
| `roles/libvirt_vm/tasks/main.yml` | All VM provisioning tasks |
| `roles/claude_setup/defaults/main.yml` | claude_setup defaults |
| `roles/claude_setup/tasks/main.yml` | ADC copy, gcloud install, Claude Code install, env vars |
| `playbook.yml` | Entry point: two plays + SSH command output |

---

## Task 1: Project Scaffolding

**Files:**
- Create: `.gitignore`
- Create: `ansible.cfg`
- Create: `requirements.yml`
- Create: `inventory/hosts.yml`
- Create: `vars/config.yml`

- [ ] **Step 1: Create `.gitignore`**

```
keys/
*.retry
```

- [ ] **Step 2: Create `ansible.cfg`**

```ini
[defaults]
roles_path = roles
inventory = inventory/hosts.yml
host_key_checking = False
remote_user = fedora
private_key_file = keys/id_rsa

[ssh_connection]
ssh_args = -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null
pipelining = True
```

- [ ] **Step 3: Create `requirements.yml`**

```yaml
collections:
  - name: community.libvirt
  - name: community.crypto
  - name: community.general
  - name: ansible.posix
```

- [ ] **Step 4: Create `inventory/hosts.yml`**

```yaml
all:
  hosts:
    localhost:
      ansible_connection: local
```

- [ ] **Step 5: Create `vars/config.yml`**

```yaml
# Shared mounts: list of {host_path, guest_path} pairs
# Leave empty to disable shared mounts
shared_mounts: []
# Example:
# shared_mounts:
#   - host_path: /home/user/projects
#     guest_path: /home/fedora/projects

# Path to gcloud ADC credentials on host
adc_credentials_path: "~/.config/gcloud/application_default_credentials.json"

# Vertex AI / Google Cloud settings
# google_project_id is required — set it before running
google_project_id: ""
google_cloud_location: "global"
```

- [ ] **Step 6: Install Ansible collections**

```bash
ansible-galaxy collection install -r requirements.yml
```

Expected output: collections installed to `~/.ansible/collections/`

- [ ] **Step 7: Commit**

```bash
git add .gitignore ansible.cfg requirements.yml inventory/hosts.yml vars/config.yml
git commit -m "feat: project scaffolding — config, inventory, vars, collections"
```

---

## Task 2: libvirt_vm Role — Defaults and Templates

**Files:**
- Create: `roles/libvirt_vm/defaults/main.yml`
- Create: `roles/libvirt_vm/templates/network.xml.j2`
- Create: `roles/libvirt_vm/templates/user-data.j2`
- Create: `roles/libvirt_vm/templates/meta-data.j2`
- Create: `roles/libvirt_vm/templates/network_hook.sh.j2`

- [ ] **Step 1: Create role directories**

```bash
mkdir -p roles/libvirt_vm/{defaults,tasks,templates}
```

- [ ] **Step 2: Create `roles/libvirt_vm/defaults/main.yml`**

```yaml
vm_name: claude-basement
vm_vcpus: 2
vm_ram_mb: 4096
vm_disk_gb: 20
vm_network_name: claude-net
vm_network_subnet: "192.168.150.0/24"
vm_network_gateway: "192.168.150.1"
vm_network_netmask: "255.255.255.0"
vm_network_dhcp_start: "192.168.150.100"
vm_network_dhcp_end: "192.168.150.200"
fedora_image_url: "https://download.fedoraproject.org/pub/fedora/linux/releases/43/Cloud/x86_64/images/Fedora-Cloud-Base-Generic.x86_64-43.qcow2"
fedora_image_dest: "/var/lib/libvirt/images/Fedora-Cloud-Base-Generic.x86_64-43.qcow2"
```

- [ ] **Step 3: Create `roles/libvirt_vm/templates/network.xml.j2`**

```xml
<network>
  <name>{{ vm_network_name }}</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr-claude' stp='on' delay='0'/>
  <ip address='{{ vm_network_gateway }}' netmask='{{ vm_network_netmask }}'>
    <dhcp>
      <range start='{{ vm_network_dhcp_start }}' end='{{ vm_network_dhcp_end }}'/>
    </dhcp>
  </ip>
</network>
```

- [ ] **Step 4: Create `roles/libvirt_vm/templates/user-data.j2`**

```yaml
#cloud-config
hostname: {{ vm_name }}
users:
  - name: fedora
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - {{ ssh_public_key }}
```

- [ ] **Step 5: Create `roles/libvirt_vm/templates/meta-data.j2`**

```yaml
instance-id: {{ vm_name }}
local-hostname: {{ vm_name }}
```

- [ ] **Step 6: Create `roles/libvirt_vm/templates/network_hook.sh.j2`**

This hook runs when the libvirt network starts and idempotently inserts iptables rules
that block the VM from reaching the host LAN while allowing internet and own-subnet traffic.
Rules are inserted in reverse priority order so the ACCEPT own-subnet rule ends up first.

```bash
#!/bin/bash
# Managed by Ansible — do not edit manually
# Network isolation hook for {{ vm_network_name }}

NETWORK_NAME="$1"
ACTION="$2"

[ "$NETWORK_NAME" = "{{ vm_network_name }}" ] && [ "$ACTION" = "started" ] || exit 0

# Idempotent insert: check first, insert at position 1 only if missing
ipt() {
    iptables -C "$@" 2>/dev/null || iptables -I "$@"
}

# Insert in reverse priority order — each goes to position 1, pushing others down.
# Final order in FORWARD chain:
#   1. ACCEPT src={{ vm_network_subnet }} dst={{ vm_network_subnet }}  (own subnet, gateway)
#   2. DROP   src={{ vm_network_subnet }} dst=10.0.0.0/8
#   3. DROP   src={{ vm_network_subnet }} dst=172.16.0.0/12
#   4. DROP   src={{ vm_network_subnet }} dst=192.168.0.0/16
#   5. [libvirt ACCEPT rules for established/related and general forward]

ipt FORWARD -s {{ vm_network_subnet }} -d 192.168.0.0/16 -j DROP
ipt FORWARD -s {{ vm_network_subnet }} -d 172.16.0.0/12 -j DROP
ipt FORWARD -s {{ vm_network_subnet }} -d 10.0.0.0/8 -j DROP
ipt FORWARD -s {{ vm_network_subnet }} -d {{ vm_network_subnet }} -j ACCEPT
```

- [ ] **Step 7: Commit**

```bash
git add roles/libvirt_vm/
git commit -m "feat: libvirt_vm role defaults and templates"
```

---

## Task 3: libvirt_vm Role — Tasks

**Files:**
- Create: `roles/libvirt_vm/tasks/main.yml`

- [ ] **Step 1: Create `roles/libvirt_vm/tasks/main.yml`**

```yaml
---
# ── Pre-flight checks ──────────────────────────────────────────────────────────

- name: Assert required host packages are installed
  ansible.builtin.command: rpm -q {{ item }}
  loop:
    - virt-install
    - libvirt
    - qemu-kvm
    - python3-libvirt
    - cloud-utils
    - virtiofsd
  changed_when: false

# ── SSH key generation ─────────────────────────────────────────────────────────

- name: Create keys directory
  ansible.builtin.file:
    path: "{{ playbook_dir }}/keys"
    state: directory
    mode: '0700'
  become: false

- name: Generate SSH key pair for VM access
  community.crypto.openssh_keypair:
    path: "{{ playbook_dir }}/keys/id_rsa"
    type: rsa
    size: 4096
  become: false

# ── Fedora cloud image ─────────────────────────────────────────────────────────

- name: Download Fedora 43 Cloud Base image
  ansible.builtin.get_url:
    url: "{{ fedora_image_url }}"
    dest: "{{ fedora_image_dest }}"
    mode: '0644'
  register: image_download

- name: Check if VM disk image already exists
  ansible.builtin.stat:
    path: "/var/lib/libvirt/images/{{ vm_name }}.qcow2"
  register: vm_disk

- name: Create VM disk as thin overlay on base image
  ansible.builtin.command: >
    qemu-img create -f qcow2 -F qcow2
    -b {{ fedora_image_dest }}
    /var/lib/libvirt/images/{{ vm_name }}.qcow2
    {{ vm_disk_gb }}G
  when: not vm_disk.stat.exists

# ── cloud-init ISO ─────────────────────────────────────────────────────────────

- name: Read SSH public key
  ansible.builtin.set_fact:
    ssh_public_key: "{{ lookup('file', playbook_dir + '/keys/id_rsa.pub') }}"

- name: Create cloud-init user-data
  ansible.builtin.template:
    src: user-data.j2
    dest: "/tmp/user-data-{{ vm_name }}"
    mode: '0600'

- name: Create cloud-init meta-data
  ansible.builtin.template:
    src: meta-data.j2
    dest: "/tmp/meta-data-{{ vm_name }}"
    mode: '0644'

- name: Create cloud-init ISO
  ansible.builtin.command: >
    cloud-localds
    /var/lib/libvirt/images/cloud-init-{{ vm_name }}.iso
    /tmp/user-data-{{ vm_name }}
    /tmp/meta-data-{{ vm_name }}
  args:
    creates: "/var/lib/libvirt/images/cloud-init-{{ vm_name }}.iso"

# ── libvirt network ────────────────────────────────────────────────────────────

- name: Check if libvirt network exists
  ansible.builtin.command: virsh net-info {{ vm_network_name }}
  register: net_exists
  failed_when: false
  changed_when: false

- name: Define libvirt NAT network
  community.libvirt.virt_net:
    name: "{{ vm_network_name }}"
    xml: "{{ lookup('template', 'network.xml.j2') }}"
    state: present
  when: net_exists.rc != 0

- name: Start libvirt network
  community.libvirt.virt_net:
    name: "{{ vm_network_name }}"
    state: active

- name: Enable network autostart
  community.libvirt.virt_net:
    name: "{{ vm_network_name }}"
    autostart: true

# ── Network isolation hook ─────────────────────────────────────────────────────

- name: Ensure libvirt hooks directory exists
  ansible.builtin.file:
    path: /etc/libvirt/hooks
    state: directory
    mode: '0755'

- name: Deploy network isolation hook
  ansible.builtin.template:
    src: network_hook.sh.j2
    dest: /etc/libvirt/hooks/network
    mode: '0755'
    owner: root
    group: root

- name: Apply isolation rules now (simulate network started event)
  ansible.builtin.command: /etc/libvirt/hooks/network {{ vm_network_name }} started
  changed_when: false

# ── VM creation ────────────────────────────────────────────────────────────────

- name: Check if VM already exists
  ansible.builtin.command: virsh dominfo {{ vm_name }}
  register: vm_exists
  failed_when: false
  changed_when: false

- name: Build virtiofs args for shared mounts
  ansible.builtin.set_fact:
    virtiofs_args: >-
      {% if shared_mounts | length > 0 %}
      --memorybacking source.type=memfd,access.mode=shared
      {% for mount in shared_mounts %}
      --filesystem source.dir={{ mount.host_path }},target.dir=mount_tag_{{ loop.index0 }},driver.type=virtiofs
      {% endfor %}
      {% else %}
      {% endif %}

- name: Create VM with virt-install
  ansible.builtin.command: >-
    virt-install
    --name {{ vm_name }}
    --ram {{ vm_ram_mb }}
    --vcpus {{ vm_vcpus }}
    --disk path=/var/lib/libvirt/images/{{ vm_name }}.qcow2,format=qcow2
    --disk path=/var/lib/libvirt/images/cloud-init-{{ vm_name }}.iso,device=cdrom
    --network network={{ vm_network_name }}
    --os-variant fedora38
    --import
    --noautoconsole
    {{ virtiofs_args }}
  when: vm_exists.rc != 0

# ── Wait for VM and discover IP ────────────────────────────────────────────────

- name: Wait for VM to acquire DHCP lease
  ansible.builtin.command: virsh domifaddr {{ vm_name }} --source lease
  register: vm_ip_result
  until: vm_ip_result.stdout | regex_search('\d+\.\d+\.\d+\.\d+')
  retries: 30
  delay: 5
  changed_when: false

- name: Extract VM IP address
  ansible.builtin.set_fact:
    vm_ip: "{{ vm_ip_result.stdout | regex_search('(\\d+\\.\\d+\\.\\d+\\.\\d+)') }}"

- name: Wait for SSH port to open
  ansible.builtin.wait_for:
    host: "{{ vm_ip }}"
    port: 22
    timeout: 300

# ── Register VM in inventory ───────────────────────────────────────────────────

- name: Add VM to in-memory inventory
  ansible.builtin.add_host:
    name: "{{ vm_ip }}"
    groups: vm
    ansible_user: fedora
    ansible_ssh_private_key_file: "{{ playbook_dir }}/keys/id_rsa"
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
    vm_ip: "{{ vm_ip }}"
```

- [ ] **Step 2: Commit**

```bash
git add roles/libvirt_vm/tasks/main.yml
git commit -m "feat: libvirt_vm role tasks — VM provisioning, network isolation, inventory"
```

---

## Task 4: claude_setup Role

**Files:**
- Create: `roles/claude_setup/defaults/main.yml`
- Create: `roles/claude_setup/tasks/main.yml`

- [ ] **Step 1: Create role directories**

```bash
mkdir -p roles/claude_setup/{defaults,tasks}
```

- [ ] **Step 2: Create `roles/claude_setup/defaults/main.yml`**

```yaml
claude_user: fedora
adc_credentials_path: "~/.config/gcloud/application_default_credentials.json"
google_project_id: ""
google_cloud_location: "global"
```

- [ ] **Step 3: Create `roles/claude_setup/tasks/main.yml`**

```yaml
---
# ── Pre-flight ─────────────────────────────────────────────────────────────────

- name: Assert google_project_id is configured
  ansible.builtin.assert:
    that:
      - google_project_id | length > 0
    fail_msg: >
      google_project_id must be set in vars/config.yml.
      Set it to your Vertex AI GCP project ID (e.g. itpc-gcp-global-eng-claude).

# ── ADC credentials ────────────────────────────────────────────────────────────
# Must happen before gcloud install tasks (gcloud checks for this file)

- name: Create gcloud config directory
  ansible.builtin.file:
    path: "/home/{{ claude_user }}/.config/gcloud"
    state: directory
    owner: "{{ claude_user }}"
    group: "{{ claude_user }}"
    mode: '0700'

- name: Copy ADC credentials from host to VM
  ansible.builtin.copy:
    src: "{{ adc_credentials_path }}"
    dest: "/home/{{ claude_user }}/.config/gcloud/application_default_credentials.json"
    owner: "{{ claude_user }}"
    group: "{{ claude_user }}"
    mode: '0600'

# ── Google Cloud CLI ───────────────────────────────────────────────────────────

- name: Add Google Cloud SDK yum repository
  ansible.builtin.yum_repository:
    name: google-cloud-sdk
    description: Google Cloud CLI
    baseurl: "https://packages.cloud.google.com/yum/repos/cloud-sdk-el9-x86_64"
    enabled: true
    gpgcheck: true
    repo_gpgcheck: false
    gpgkey:
      - "https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg"

- name: Install gcloud CLI and dependencies
  ansible.builtin.dnf:
    name:
      - libxcrypt-compat
      - google-cloud-cli
    state: present

# ── Node.js and Claude Code ────────────────────────────────────────────────────

- name: Install Node.js
  ansible.builtin.dnf:
    name: nodejs
    state: present

- name: Install Claude Code globally via npm
  community.general.npm:
    name: "@anthropic-ai/claude-code"
    global: true
    state: latest

# ── Vertex AI environment variables ───────────────────────────────────────────

- name: Write Vertex AI environment variables to profile.d
  ansible.builtin.copy:
    dest: /etc/profile.d/claude_code.sh
    owner: root
    group: root
    mode: '0644'
    content: |
      export CLAUDE_CODE_USE_VERTEX=1
      export CLOUD_ML_REGION={{ google_cloud_location }}
      export ANTHROPIC_VERTEX_PROJECT_ID={{ google_project_id }}

# ── Shared mount fstab entries ─────────────────────────────────────────────────

- name: Create guest mount point directories
  ansible.builtin.file:
    path: "{{ item.guest_path }}"
    state: directory
    owner: "{{ claude_user }}"
    group: "{{ claude_user }}"
    mode: '0755'
  loop: "{{ shared_mounts }}"
  when: shared_mounts | length > 0

- name: Add virtiofs entries to /etc/fstab
  ansible.posix.mount:
    path: "{{ item.guest_path }}"
    src: "mount_tag_{{ loop.index0 }}"
    fstype: virtiofs
    opts: defaults
    state: mounted
  loop: "{{ shared_mounts }}"
  when: shared_mounts | length > 0
```

- [ ] **Step 4: Commit**

```bash
git add roles/claude_setup/ requirements.yml
git commit -m "feat: claude_setup role — gcloud, Claude Code, Vertex AI env vars, shared mounts"
```

---

## Task 5: playbook.yml

**Files:**
- Create: `playbook.yml`

- [ ] **Step 1: Create `playbook.yml`**

```yaml
---
- name: Provision libvirt VM
  hosts: localhost
  connection: local
  become: true
  gather_facts: true
  vars_files:
    - vars/config.yml
  pre_tasks:
    - name: Generate SSH key pair (runs as invoking user, not root)
      community.crypto.openssh_keypair:
        path: "{{ playbook_dir }}/keys/id_rsa"
        type: rsa
        size: 4096
      become: false
  roles:
    - libvirt_vm

- name: Configure VM with Claude Code and Vertex AI
  hosts: vm
  gather_facts: false
  become: true
  vars_files:
    - vars/config.yml
  roles:
    - claude_setup
  post_tasks:
    - name: Print SSH access command
      ansible.builtin.debug:
        msg: "Connect to your Claude VM: ssh -i {{ playbook_dir }}/keys/id_rsa fedora@{{ inventory_hostname }}"
```

Note: SSH key generation is in `pre_tasks` with `become: false` so the key files are owned by the invoking user, not root. The `libvirt_vm` role also generates the key, but having it in `pre_tasks` ensures it runs before `become: true` tasks in play 1 need to read it. Remove the duplicate key generation from the role:

- [ ] **Step 2: Remove duplicate key generation from libvirt_vm tasks**

In `roles/libvirt_vm/tasks/main.yml`, remove the two tasks `Create keys directory` and `Generate SSH key pair for VM access` — they are now handled by `playbook.yml` pre_tasks. Keep only the `Read SSH public key` task which uses the already-generated key.

Updated `roles/libvirt_vm/tasks/main.yml` — remove these two tasks:

```yaml
# DELETE these two tasks (they move to playbook.yml pre_tasks):

- name: Create keys directory          # DELETE
  ...

- name: Generate SSH key pair          # DELETE
  ...
```

The pre_tasks in `playbook.yml` now own key generation. The role reads the key with:

```yaml
- name: Read SSH public key
  ansible.builtin.set_fact:
    ssh_public_key: "{{ lookup('file', playbook_dir + '/keys/id_rsa.pub') }}"
```

- [ ] **Step 3: Commit**

```bash
git add playbook.yml roles/libvirt_vm/tasks/main.yml
git commit -m "feat: playbook.yml — two-play entry point with SSH command output"
```

---

## Manual Verification

After setting `google_project_id` in `vars/config.yml`, run:

```bash
ansible-playbook playbook.yml --ask-become-pass
```

Verification checklist (run inside the VM via the printed SSH command):

```bash
# 1. Claude Code is installed
claude --version

# 2. Vertex AI env vars are set
grep VERTEX /etc/profile.d/claude_code.sh

# 3. ADC credentials are present
ls -la ~/.config/gcloud/application_default_credentials.json

# 4. gcloud can use the credentials
gcloud auth application-default print-access-token

# 5. Internet access works
curl -s https://example.com | head -5

# 6. Host LAN is blocked (replace 192.168.1.1 with your actual gateway)
ping -c 2 -W 2 192.168.1.1 && echo "FAIL: LAN accessible" || echo "PASS: LAN blocked"

# 7. Shared mounts (if configured)
ls -la /path/to/guest_path
```
