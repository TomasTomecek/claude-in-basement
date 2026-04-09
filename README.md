# Claude living in my basement

This is a simple project to lock down Claude Code in a VM so it cannot disrupt anything from the host system.

There is a Ansible playbook here that will provision a VM, sets up Claude Code inside, copies Vertex AI authentication inside and print `ssh` command to access the VM.
