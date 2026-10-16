# Ubuntu Upgrade Guide (Manual + Ansible)

## 1. Ubuntu Package Upgrade (manual)

```bash
sudo apt update                # refresh package list
apt list --upgradable          # preview what will be updated
sudo apt upgrade -y            # install updates (no new/removed packages)
sudo apt full-upgrade -y       # also handles changed dependencies (e.g. new kernels)
sudo apt autoremove -y         # remove orphaned dependencies and old kernels
sudo apt autoclean             # clear old cached .deb files
```

One-liner:

```bash
sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y
```

Preview without changing anything:

```bash
sudo apt full-upgrade --dry-run
sudo apt autoremove --dry-run
```

Check if a reboot is needed:

```bash
[ -f /var/run/reboot-required ] && echo "Reboot required"
sudo reboot
```

Check versions:

```bash
lsb_release -a
cat /etc/os-release
uname -r
```

## 2. Ubuntu OS (Release) Upgrade

Example: 22.04 LTS to 24.04 LTS.

Before you start:

- Take a backup or snapshot.
- Run inside `tmux` or `screen` so a dropped SSH session does not interrupt it.
- Fully update the current release first.

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove -y
sudo apt install update-manager-core -y
sudo reboot
```

After reboot:

```bash
sudo do-release-upgrade
```

Notes:

- LTS systems upgrade to the next LTS only once it gets its first point release (for example 24.04.1). To upgrade earlier, use `sudo do-release-upgrade -d`.
- Check that `Prompt=lts` is set in `/etc/update-manager/release-upgrades`.
- Verify afterwards with `lsb_release -a`.

## 3. `upgrade` values in the Ansible apt module

| Value | Equivalent command | Behavior |
|---|---|---|
| `no` (default) | none | No upgrade |
| `yes` / `safe` | `apt-get upgrade` | Upgrades only, never installs or removes other packages |
| `full` | `aptitude full-upgrade` | Like dist, but uses aptitude |
| `dist` | `apt-get dist-upgrade` | Upgrades, and installs or removes packages as needed |

## 4. Ansible Ad-hoc Commands

Remote hosts (using an inventory):

```bash
ansible all -i inventory.ini -b -m ansible.builtin.apt -a "update_cache=yes upgrade=dist autoremove=yes"
```

Specific group or single host:

```bash
ansible tomcat_servers -i inventory.ini -b -m apt -a "update_cache=yes upgrade=dist autoremove=yes"
ansible all -i inventory.ini -b -m apt -a "update_cache=yes upgrade=dist" --limit 192.168.1.10
```

Dry run (check mode):

```bash
ansible all -i inventory.ini -b -m apt -a "update_cache=yes upgrade=dist autoremove=yes" --check
```

Local machine (no inventory, no SSH):

```bash
ansible localhost -c local -b -K -m ansible.builtin.apt -a "update_cache=yes upgrade=dist autoremove=yes"
```

Flags:

- `-b` runs with sudo (become)
- `-K` asks for the sudo password
- `-c local` runs locally instead of over SSH
- `-f 10` runs on 10 hosts in parallel

Reboot checks:

```bash
ansible all -i inventory.ini -m stat -a "path=/var/run/reboot-required"
ansible all -i inventory.ini -b -m reboot
```

The `reboot` module reboots every targeted host regardless of the stat result, so limit it to hosts that need it.

## 5. Ansible Playbooks

### 5.1 Remote servers (`linuxupgrade.yml`)

```yaml
---
- name: Upgrade Ubuntu servers
  hosts: all
  become: true
  serial: 2                     # upgrade 2 hosts at a time

  tasks:
    - name: Update apt cache and upgrade all packages
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
        autoremove: true
        autoclean: true

    - name: Check if a reboot is required
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required

    - name: Reboot if required
      ansible.builtin.reboot:
        reboot_timeout: 600
      when: reboot_required.stat.exists
```

Run:

```bash
ansible-playbook -i inventory.ini linuxupgrade.yml --syntax-check
ansible-playbook -i inventory.ini linuxupgrade.yml --check
ansible-playbook -i inventory.ini linuxupgrade.yml
```

### 5.2 Local machine (`linuxupgrade-local.yml`)

```yaml
---
- name: Upgrade local Ubuntu
  hosts: localhost
  connection: local
  become: true

  tasks:
    - name: Update packages
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
        autoremove: true
```

Run:

```bash
ansible-playbook linuxupgrade-local.yml -K
```

### 5.3 Release upgrade (OS version) with Ansible

```yaml
---
- name: Upgrade Ubuntu to the next release
  hosts: all
  become: true
  serial: 1

  tasks:
    - name: Fully update current release
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
        autoremove: true

    - name: Install update-manager-core
      ansible.builtin.apt:
        name: update-manager-core
        state: present

    - name: Reboot before release upgrade
      ansible.builtin.reboot:
        reboot_timeout: 600

    - name: Run release upgrade (non-interactive)
      ansible.builtin.command: do-release-upgrade -f DistUpgradeViewNonInteractive
      register: release_upgrade
      changed_when: true

    - name: Reboot after release upgrade
      ansible.builtin.reboot:
        reboot_timeout: 1200

    - name: Show new version
      ansible.builtin.command: lsb_release -ds
      register: new_version
      changed_when: false

    - name: Print version
      ansible.builtin.debug:
        msg: "{{ new_version.stdout }}"
```

Test this on one non-production host first (`--limit host1`). A release upgrade changes many packages and config files, so snapshot or back up beforehand.

## 6. Automatic Security Updates (optional)

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## 7. Quick Reference

| Task | Command |
|---|---|
| Refresh package list | `sudo apt update` |
| Upgrade packages | `sudo apt full-upgrade -y` |
| Clean up | `sudo apt autoremove -y` |
| Release upgrade | `sudo do-release-upgrade` |
| Ansible ad-hoc | `ansible all -b -m apt -a "update_cache=yes upgrade=dist"` |
| Ansible local | `ansible localhost -c local -b -K -m apt -a "update_cache=yes upgrade=dist"` |
| Playbook run | `ansible-playbook -i inventory.ini linuxupgrade.yml` |
