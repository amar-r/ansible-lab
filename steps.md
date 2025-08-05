## Project Phases

| Phase | Description |
|-------|-------------|
| 1 | Set up control node and secure SSH access to macOS. Install Ansible on Ubuntu, set up SSH key-based access, and verify connectivity with `ansible -m ping`. |
| 2 | Initialize Ansible project structure and inventory. Use a YAML-based inventory, modular folder layout, and `.gitignore` for secrets. Prepare the repo for use with `ansible-pull` and AWX. |
| 3 | Write modular playbooks using best practices. Use roles, tags, and group variables. Start with a `base` role (e.g., Homebrew + CLI tools). Maintain `bootstrap.yml` as the main entry point. |
| 4 | Automate realistic tasks for macOS. Install and configure software, set system preferences using `defaults`, apply security hardening, and simulate enterprise-style compliance tasks. |
| 5 | Secure secrets and enable pull-based self-management. Use `ansible-vault` or `.env` for secrets. Set up `ansible-pull` to allow the Mac to self-apply changes via scheduled jobs using `launchd`. |
| 6 | Deploy and integrate AWX for web-based automation control. Use Docker Compose to stand up AWX. Sync your Git repo, import inventory, create credentials, and run `bootstrap.yml` as a job template. |
| 7 | (Optional) Schedule runs and add observability. Use `launchd` on macOS and `cron` or AWX schedules to auto-run playbooks. Add logging or reporting for auditing changes. |
| 8 | (Optional) Expand to other systems. Add your second Mac or other Linux hosts. Use groups, `host_vars`, and separate roles for shared vs per-host automation logic. |

## Phase 1: Set up control node and secure SSH access to macOS

### 1.1 Install Ansible on Ubuntu

```bash
sudo apt update
sudo apt install ansible -y
ansible --version
```

---

### 1.2 Generate SSH Key on Ubuntu (if needed)

```bash
ssh-keygen -t ed25519 -C "ansible@control-node"
ssh-add ~/.ssh/id_ed25519
```

- ✅ Save to default location: `~/.ssh/id_ed25519`  
- ✅ Use a strong passphrase

---

### 1.3 Enable SSH Access on macOS

Run via CLI (requires Terminal to have Full Disk Access):

```bash
sudo systemsetup -setremotelogin on
```

If prompted:

- Go to **System Settings > Privacy & Security > Full Disk Access**
- Add **Terminal.app**

Or enable via GUI:

- **System Settings > General > Sharing** → Toggle **Remote Login**

---

### 1.4 Copy SSH Key to macOS

```bash
ssh-copy-id -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519.pub -F /dev/null <macos_user>@<macos_ip>
```

---

### 1.5 Verify SSH Works

```bash
ssh <macos_user>@<macos_ip>
```

> You should be prompted for your **SSH key passphrase**, not your macOS password.

---

## Phase 2: Initialize Ansible project structure and inventory

### 2.1 Scaffold Directory Structure

```bash
mkdir -p ~/mac-ansible-lab/{inventory,playbooks,roles,group_vars,host_vars}
cd ~/mac-ansible-lab
touch inventory/hosts.yml
touch playbooks/bootstrap.yml
```

---

### 2.2 Add `.gitignore`

Create a `.gitignore` file at the root of your project:

```gitignore
.env
*.vault.yml
*.retry
*.log
__pycache__/
```

---

### 2.3 Define Inventory (`inventory/hosts.yml`)

```yaml
all:
  children:
    macos:
      hosts:
        macbook:
          ansible_host: <macos_ip>
          ansible_user: <macos_user>
          ansible_ssh_private_key_file: ~/.ssh/id_ed25519
```

---

### 2.4 Test Connectivity

```bash
ansible -i inventory/hosts.yml macos -m ping
```

Expected output:

```json
macbook | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Phase 3: Write modular playbooks using best practices. Use roles, tags, and group variables.

### 3.3 Create `playbooks/bootstrap.yml`

Create a new file under playbooks dir called `bootstrap.yml`

```yaml
---
- name: Bootstrap macOS system with base role
  hosts: macos
  gather_facts: true
  become: false

  roles:
    - base
```

Explanation:

* `hosts: macos`: targets teh group defined in hosts.yml
* `gather_facts: true`: collects info like architecture, OS, etc.
* `roles: -base`: calls the `roles/base` logic

### 3.4 Run the playbook

From the repo root:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml
```

Had to create a file `ansible.cfg` on the root

```ini
[defaults]
inventory = ./inventory/hosts.yml
roles_path = ./roles
host_key_checking = False
```

### 3.5 Add Tags to Role Tasks

Update main.yml like this:

```yaml
---
- name: Ensure Homebrew is installed
  shell: |
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  args:
    creates: /usr/local/bin/brew
  become: false
  tags: [brew]

- name: Add Homebrew to PATH in ~/.zprofile
  lineinfile:
    path: ~/.zprofile
    line: 'eval "$(/usr/local/bin/brew shellenv)"'
    create: yes
  when: ansible_facts['os_family'] == 'Darwin'
  tags: [brew]

- name: Load Homebrew environment in current shell
  shell: eval "$(/usr/local/bin/brew shellenv)"
  args:
    executable: /bin/bash
  when: ansible_facts['os_family'] == 'Darwin'
  tags: [brew]

- name: Install CLI tools using Homebrew
  homebrew:
    name: "{{ brew_cli_tools }}"
    state: present
  tags: [cli]
```

### 3.6 Move Tool List to `group_vars/macos.yml`

Create a new file `macos.yml` in the `group_vars` directory

And add:

```yaml
brew_cli_tools:
  - htop
  - jq
  - bat
  - wget
```

### 3.7 Run with Tags

Only install brew:

```bash
ansible-playbook playbooks/bootstrap.yml --tags brew
```

Only install CLI tools:

```bash
ansible-playbook playbooks/bootstrap.yml --tags cli
```

Run everything:

```bash
ansible-playbook playbooks/bootstrap.yml
```

Fixing an issue with variables:

Error:

```bash
fatal: [macbook]: FAILED! => {"msg": "The task includes an option with an undefined variable. The error was: 'brew_cli_tools' is undefined\n\nThe error appears to be in '/../roles/base/tasks/main.yml': line 26, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n\n- name: Install CLI tools using Homebrew\n  ^ here\n"}
```

Fix:

Add the following to `roles/base/defaults/main.yml`:

```yaml
---
brew_cli_tools: []
```

## Phase 4: Automate realistic tasks for macOS

### 4.1 Scaffold New Roles

```bash
ansible-galaxy init roles/preferences
ansible-galaxy init roles/security
ansible-galaxy init roles/updates
ansible-galaxy init roles/compliance
```

This should create the following:

```text
roles/
├── preferences/
├── security/
├── updates/
└── compliance/
```

### 4.2 Add System Preference Tasks

See [roles/preferences/tasks/main.yml](roles/preferences/tasks/main.yml). 

### 4.3 Add to Playbook

Update `playbooks/bootstrap.yml` by appending `preferences` to the roles section:

```yaml
role:
    - base
    - preferences
```

Run to validate

```bash
ansible-playbook playbooks/bootstrap.yml --tags preferences
```

### 4.4 Create the security role

Update the boostrap.yml:

```yaml
roles:
  - base
  - preferences
  - security
```

### 4.5 Add Hardening Tasks

See [roles/security/tasks/main.yml](roles/security/tasks/main.yml)

## Phase 5: Secure secrets and enable pull-based self-management

### 5.1

Create or edit your vault file:

```bash
ansible-vault edit group_vars/macos/vault.yml
```

Add:

```bash
ansible_become_password: "yourpassword"
```

Ensure group_vars/macos/macos.yml includes:

```yaml
ansible_become: true
ansible_become_method: sudo
```

Run with vault

> Currently not getting this step done

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml --extra-vars "@group_vars/macos/vault.yml" --ask-vault-pass
```

### 5.2 Enable pull based automation with `ansible-pull`

```bash
  ansible-pull \
    -U git@github.com:amar-r/ansible-lab.git \
    -i inventory/hosts.yml \
    -d ~/ansible-pull \
    -v \
    playbooks/bootstrap.yml
```
> Pausing ansible-pull for now

## Phase 6: Deploy and Integrate AWX with Docker Compose

### 6.1 Install Prerequisites on Control Node (Ubuntu)

Ensure the following are installed:

* docker.io
* docker compose
* git
* python3-pip

### Clone AWX Docker Repo