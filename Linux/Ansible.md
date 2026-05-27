


ansible-playbook update-backup.yml -i inventory.ini

ansible-playbook -i inventory.ini get_ip.yml


ansible-playbook -i "localhost," -c local get_ip.yml




nano inventory.ini
[serverlist]
8.8.8.8

nano server-update.yml

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
        path: /var/run/reboot-required
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
      changed_when: backup_output.rc != 0 # consider using 'failed_when' instead, depending on your need>
      ignore_errors: true # useful for handling non-critical backup failures

    - name: Display backup script output (optional)
      debug:
        var: backup_output.stdout_lines
      when: backup_output.rc != 0 # only display if the backup script had a non-zero exit code.
```






```yml
- name: Backup MySQL database
      command: >
        mysqldump
        -u {{ mysql_user }}
        -p'{{ mysql_password }}'
        --databases {{ mysql_database }}
        > /path/to/backups/{{ mysql_database }}_{{ ansible_hostname }}_{{ ansible_date_time }}.sql
      register: backup_result
      changed_when: backup_result.rc != 0
      failed_when: backup_result.rc != 0

    - name: Compress the backup
      command: gzip /path/to/backups/{{ mysql_database }}_{{ ansible_hostname }}_{{ ansible_date_time }}.sql
```







```yml
- name: Backup PostgreSQL database
      command: >
        pg_dump
        -U {{ postgres_user }}
        -h {{ postgres_host }}
        -p {{ postgres_port }}
        -Fc
        -f /path/to/backups/{{ postgres_database }}_{{ ansible_hostname }}_{{ ansible_date_time }}.dump {{ postgres_database }}
      environment:
        PGPASSWORD: "{{ postgres_password }}"
      register: backup_result
      changed_when: backup_result.rc != 0
      failed_when: backup_result.rc != 0

```
