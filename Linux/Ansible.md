
# Ansible

Agentless config management over SSH. Control node pushes; managed nodes need only Python + SSH (no agent).

- [Install](#install)
- [Inventory](#inventory)
- [Ad-hoc commands](#ad-hoc-commands)
- [Run playbooks](#run-playbooks)
- [Playbook anatomy](#playbook-anatomy)
- [Variables, vars files & Vault](#variables-vars-files--vault)
- [Loops, conditionals, handlers](#loops-conditionals-handlers)
- [Templates & files](#templates--files)
- [Roles layout](#roles-layout)
- [Playbook: update + reboot + backup](#playbook-update--reboot--backup)
- [Playbook: MySQL backup](#playbook-mysql-backup)
- [Playbook: PostgreSQL backup](#playbook-postgresql-backup)
- [Gotchas](#gotchas)

## Install
```bash
sudo apt install -y ansible          # full package (incl. ansible-playbook, ansible-vault, ansible-galaxy)
ansible --version                    # confirm + shows config file in use
pipx install ansible                 # alt: newer version, isolated from system python
```
> Only the **control node** needs Ansible. Managed hosts just need SSH access + a Python 3 interpreter.

## Inventory
```ini
# inventory.ini
[serverlist]
8.8.8.8

[web]
192.168.8.10
192.168.8.11

[db]
db1 ansible_host=192.168.8.20 ansible_user=ahmad ansible_port=2222

[db:vars]
ansible_python_interpreter=/usr/bin/python3   # avoid "python not found" on minimal hosts

[prod:children]      # group of groups
web
db
```
```bash
ansible-inventory -i inventory.ini --list      # dump parsed inventory as JSON
ansible-inventory -i inventory.ini --graph     # tree view of groups/hosts
ansible all -i inventory.ini --list-hosts      # who would a play target
```

## Ad-hoc commands
One-off `module` calls without writing a playbook: `ansible <pattern> -m <module> -a "<args>"`.
```bash
ansible all -i inventory.ini -m ping                      # SSH + python reachability check
ansible all -i inventory.ini -m setup                     # dump all facts (ansible_* vars)
ansible web -i inventory.ini -a "uptime"                  # -m command is the default
ansible all -i inventory.ini -b -m apt -a "name=htop state=present"   # -b = become root (sudo)
ansible all -i inventory.ini -m service -a "name=nginx state=restarted" -b
ansible all -i inventory.ini -m copy -a "src=./f dest=/tmp/f" -b
ansible all -i inventory.ini -m shell -a "df -h | grep /dev/sda"      # shell = pipes/redirects; command = no shell
```
> `command` (default) is safer but **no pipes/redirects/globs** — use `shell` when you need them.

## Run playbooks
```bash
ansible-playbook -i inventory.ini server-update.yml
ansible-playbook update-backup.yml -i inventory.ini
ansible-playbook -i inventory.ini get_ip.yml

# run against the control node itself, no SSH:
ansible-playbook -i "localhost," -c local get_ip.yml      # trailing comma = inline host list; -c local = local connection

# essential flags
ansible-playbook site.yml -i inventory.ini --check        # dry-run, report changes, change nothing
ansible-playbook site.yml -i inventory.ini --diff         # show file content diffs
ansible-playbook site.yml -i inventory.ini --limit db1    # only this host
ansible-playbook site.yml -i inventory.ini --tags backup  # only tasks tagged 'backup'
ansible-playbook site.yml -i inventory.ini --start-at-task "Upgrade packages"
ansible-playbook site.yml -i inventory.ini -K             # prompt for sudo (become) password
ansible-playbook site.yml -i inventory.ini -vvv           # verbose (up to -vvvv for connection debug)
```

## Playbook anatomy
```yml
- name: Describe the play              # a playbook = list of plays
  hosts: web                           # which inventory group/pattern
  become: true                         # run tasks as root via sudo
  vars:
    pkg: nginx
  tasks:
    - name: Install package            # every task should be named (shows in output)
      apt:
        name: "{{ pkg }}"
        state: present
      notify: restart nginx            # fires handler only if THIS task changed something
  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```
> Tasks are **idempotent**: re-running converges to the same state. `state: present` ≠ `latest` — `present` won't upgrade an already-installed package.

## Variables, vars files & Vault
```yml
# precedence (low → high): role defaults < inventory < vars: < extra-vars(-e)
vars_files:
  - vars/main.yml          # external var file
```
```bash
ansible-playbook site.yml -e "mysql_database=app env=prod"     # extra-vars win over everything
ansible-playbook site.yml -e "@vars/prod.yml"                  # load vars from a file
```
```bash
# Vault: encrypt secrets at rest (passwords, keys)
ansible-vault create   secrets.yml          # new encrypted file
ansible-vault edit     secrets.yml          # edit in place
ansible-vault view     secrets.yml
ansible-vault encrypt  vars/db.yml          # encrypt an existing plaintext file
ansible-vault rekey    secrets.yml          # change the vault password
ansible-playbook site.yml --ask-vault-pass            # prompt at runtime
ansible-playbook site.yml --vault-password-file ~/.vault_pass   # CI-friendly
```
> Never commit plaintext DB passwords. Put `{{ mysql_password }}` in a **vault-encrypted** vars file, not in the playbook.

## Loops, conditionals, handlers
```yml
- name: Install several packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - htop
    - git
    - curl

- name: Only on Ubuntu 24.04
  apt:
    upgrade: dist
  when: ansible_distribution == "Ubuntu" and ansible_distribution_version == "24.04"

- name: Register output and branch on it
  command: id -u
  register: uid_out
  changed_when: false                  # read-only command, never report 'changed'
  failed_when: uid_out.rc not in [0]
```

## Templates & files
```yml
- name: Render config from Jinja2 template
  template:
    src: nginx.conf.j2                 # uses {{ vars }} inside
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: "0644"
  notify: restart nginx

- name: Copy a static file
  copy:
    src: files/motd
    dest: /etc/motd
```

## Roles layout
```bash
ansible-galaxy init roles/webserver    # scaffold a role
```
```
roles/webserver/
├── tasks/main.yml        # entry point, auto-included
├── handlers/main.yml
├── templates/            # .j2 files
├── files/                # static files to copy
├── vars/main.yml         # high-precedence vars
└── defaults/main.yml     # lowest-precedence (overridable) vars
```
```yml
# use it in a play
- hosts: web
  become: true
  roles:
    - webserver
```

## Playbook: update + reboot + backup
`server-update.yml` — patch, reboot if the kernel asks for it, then run the backup script.
```yml
- name: Update and Upgrade Ubuntu 24.04 Servers and Run Backup Script
  hosts: all
  become: true
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Upgrade packages
      apt:
        upgrade: dist

    - name: Check if a reboot is required
      register: reboot_required_file
      stat:
        path: /var/run/reboot-required      # kernel/libc updates create this marker
        get_checksum: no

    - name: Reboot if required
      reboot:
        reboot_timeout: 300
      when: reboot_required_file.stat.exists

    - name: Wait for server to come back up
      wait_for_connection:
        timeout: 600
      when: reboot_required_file.stat.exists

    - name: Run backup script
      shell: bash /home/backup.sh
      register: backup_output
      changed_when: backup_output.rc != 0   # consider 'failed_when' instead, depending on need
      ignore_errors: true                    # useful for non-critical backup failures

    - name: Display backup script output (optional)
      debug:
        var: backup_output.stdout_lines
      when: backup_output.rc != 0            # only show if backup exited non-zero
```
> The `reboot` module handles the disconnect/reconnect itself; `wait_for_connection` is a belt-and-suspenders guard for slow hosts.

## Playbook: MySQL backup
```yml
    - name: Backup MySQL database
      shell: >                                 # shell (not command) so the '>' redirect works
        mysqldump
        -u {{ mysql_user }}
        -p'{{ mysql_password }}'
        --databases {{ mysql_database }}
        > /path/to/backups/{{ mysql_database }}_{{ ansible_hostname }}_{{ ansible_date_time.iso8601 }}.sql
      register: backup_result
      changed_when: backup_result.rc != 0
      failed_when: backup_result.rc != 0

    - name: Compress the backup
      command: gzip /path/to/backups/{{ mysql_database }}_{{ ansible_hostname }}_{{ ansible_date_time.iso8601 }}.sql
```
> `ansible_date_time` is a dict — use `{{ ansible_date_time.iso8601 }}` (or `.date`) for a real timestamp, not the bare dict. The `>` redirect only works because no `shell` issues arise here; if it fails, switch `command:` to `shell:`.

## Playbook: PostgreSQL backup
```yml
    - name: Backup PostgreSQL database
      command: >
        pg_dump
        -U {{ postgres_user }}
        -h {{ postgres_host }}
        -p {{ postgres_port }}
        -Fc
        -f /path/to/backups/{{ postgres_database }}_{{ ansible_hostname }}_{{ ansible_date_time.iso8601 }}.dump {{ postgres_database }}
      environment:
        PGPASSWORD: "{{ postgres_password }}"   # pg_dump reads the password from env
      register: backup_result
      changed_when: backup_result.rc != 0
      failed_when: backup_result.rc != 0
```
> `-Fc` = custom compressed format; restore with `pg_restore`, not `psql`. Plain SQL dumps use `-Fp` and restore with `psql -f`.

## Gotchas
- **First connection fails on host key prompt** — pre-accept keys (`ssh-keyscan -H host >> ~/.ssh/known_hosts`) or set `host_key_checking = False` in `ansible.cfg` (lab only, not prod).
- **`python not found`** on minimal images — set `ansible_python_interpreter=/usr/bin/python3` per host/group.
- **`become` needs a sudo password** — pass `-K` (or `--ask-become-pass`); passwordless sudo avoids the prompt.
- **`command` silently drops `|`, `>`, `*`** — these are shell features; use the `shell` module when you need them.
- **`changed_when`/`failed_when` invert easily** — `rc != 0` means "changed/failed on non-zero exit"; double-check the logic per task.
- **`--check` (dry-run) lies for `shell`/`command`** — those modules can't predict changes, so check mode may report nothing. Add `check_mode: false` to always run read-only probes.
- **`ansible_date_time` requires facts** — it's only populated if fact-gathering ran (`gather_facts: true`, the default).
