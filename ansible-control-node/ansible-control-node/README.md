# Containerized Ansible Control Node

This pack provides a pinned Ansible control node for learning environments. It keeps Ansible and its validation tools inside a container, so the host computer does not need a separate Ansible installation.

Included tools:

- ansible-core 2.17.7
- ansible-lint 24.9.2
- yamllint 1.35.1
- OpenSSH client
- `sshpass` for interactive password authentication through Ansible's `--ask-pass` option

The `ansible-lab/` directory is mounted at `/work` and contains the initial Ansible and YAML lint configuration. It can also be used as the working directory for inventories, playbooks, variables, and templates.

## Build the image

Build with the current host UID and GID so files created inside `/work` remain writable on the host:

```bash
docker build \
  --build-arg UID="$(id -u)" \
  --build-arg GID="$(id -g)" \
  -t ansible-control:2.17.7 \
  -f Dockerfile.control .
```

## Start the control node

```bash
docker run -it --rm \
  --mount "type=bind,src=$(pwd)/ansible-lab,dst=/work" \
  ansible-control:2.17.7
```

## Verify the toolchain

Inside the container:

```bash
id
ansible --version
ansible-lint --project-dir . --version
yamllint --version
ansible-config dump --only-changed
```

The container runs as a non-root user. Privilege escalation for a managed host is configured in a playbook with `become`, independently of the control-node user.

## Authentication

For password-based SSH access, first confirm the connection with the SSH client. Ansible can then prompt for the same password with `--ask-pass`:

```bash
ssh -p <PORT> <USER>@<HOST>
ansible all -m ansible.builtin.ping --ask-pass
```

Do not store passwords in inventory files, playbooks, shell history, or version control.

## Host-key policy

The included SSH and Ansible configuration disables host-key checking for disposable learning hosts that are frequently recreated. Do not reuse this setting in production. Production control nodes should verify host keys from a trusted `known_hosts` source.
