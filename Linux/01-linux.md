# Linux Cheat Sheet

## Table of Contents

-   [Package Management](#package-management)
-   [System Information](#system-information)
-   [Networking](#networking)
-   [Processes](#processes)
-   [Find & Grep](#find--grep)
-   [Disk & Memory](#disk--memory)
-   [Compression](#compression)
-   [File Listing](#file-listing)
-   [Systemctl](#systemctl)
-   [Users](#users)
-   [Groups](#groups)
-   [Cron Jobs](#cron-jobs)
-   [SSH](#ssh)
-   [Firewall](#firewall)


------------------------------------------------------------------------

## Package Management

``` bash
sudo apt update
sudo apt upgrade
apt search <string>
apt show <package>
sudo apt install <pkg>
sudo apt remove <pkg>
dpkg --list
snap list
```

------------------------------------------------------------------------

## System Information

``` bash
cat /etc/os-release
uname -a
uptime
hostnamectl set-hostname ubuntu-server-1
cat /proc/cpuinfo
cat /proc/meminfo
```

------------------------------------------------------------------------

## Networking

Disable cloud-init networking
```bash
nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```
```TOML
network: {config: disabled}
```
Netplan config

```bash
nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.8.120/24
      routes:
        - to: default
          via: 192.168.8.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```
```bash
netplan apply
reboot
```

Get private IP
```bash
hostname -I | awk '{print $1}'
```
```bash
ip -4 addr show scope global | grep inet | awk '{print $2}' | cut -d/ -f1
```
```bash
ip -4 -o addr show | awk '{print $4}' | cut -d/ -f1
```
```bash
ip route get 1.1.1.1 | awk '{print $7}'
```

Get Public IP
```bash
curl ifconfig.me
# OR
wget -qO- eth0.me
# using Python 3
python3 -c "import socket; from urllib.request import urlopen; old_getaddrinfo = socket.getaddrinfo; socket.getaddrinfo = lambda *args, **kwargs: old_getaddrinfo(args[0], args[1], socket.AF_INET, *args[3:]); print(urlopen('https://ifconfig.me').read().decode())"
```

iproute2
``` bash
# show all interfaces
ip a
# Show only active interfaces
ip -brief address
# Show routing table
ip route
# show neighbour
ip neigh
ip neigh flush all

# Port & Socket Inspection (ss)
# t → TCP
# u → UDP
# l → listening
# n → numeric (no DNS)
# p → process info
ss -tulnp
# Filter specific port
ss -tulnp | grep 5432
# Check DB server exposure
ss -tulnp | grep -E '5432|3306'

# Traffic Control (tc)
# Show queueing disciplines
tc qdisc show
# Limit bandwidth (example)
tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 400ms

ping host
whois domain

# Get DNS information for domain
dig domain
# Reverse lookup host
dig -x host

# Download file & Continue download
wget file
wget -c file
```

Disable IPv6
``` bash
nano /etc/sysctl.conf
```
```
sysctl -p
```

------------------------------------------------------------------------

## Processes

```bash
# minimalist view
ps -He
# view a detailed "tree" of all running processes on your system.
ps -axjf
# Show all running processes (detailed, user-friendly)
ps -aux
# Show all processes (full structured format)
ps -ef
ps -e
# Show processes for a specific user
ps -u root
# Show a specific process by PID
ps -p 1234
# Show full-format details
ps -f
# Show long/low-level format
ps -l
# Custom output columns
ps -eo pid,ppid,cmd,%mem,%cpu
# Find process by name
ps aux | grep nginx
# Show process tree hierarchy
ps -ef --forest
# Show top memory consumers
ps aux --sort=-%mem | head
# Show only PID + command
ps -eo pid,cmd
# Show top CPU consumers
ps -eo pid,ppid,cmd,%cpu --sort=-%cpu | head
```

Lists all packages installed via Debian package manager (APT/dpkg)
```bash
# ii → installed OK
# rc → removed but config remains
dpkg --list
# Auditing the entire installation
dpkg -L nginx
```

Lists all installed Snap packages
```bash
snap list
```

------------------------------------------------------------------------

## Find & Grep

```bash
# "/"  Every file on the hard drive.
# "."  Everything inside current folder.
# "~"  Files belonging to your specific user.
# ".." One level up from where you are.
```
``` bash
# Find by file size
find . -type f -size +1G -exec ls -lh {} +
# sort by largest
find . -type f -size +1G -exec du -h {} + | sort -h
# delete large file
find . -type f -size +1G -delete

# Modification time
find . -type f -size +1G -mtime -7
#  7 → exactly 7 days ago
# -7 → the last 7 days
# +7 → more than 7 days ago

find . -type f -size +1G -mtime -7 -mtime +1
# Between 1 and 7 days old

find . -type f -size +500M -mmin -60
# Modified within the last 60 minutes

find . -type f -size +1G -atime +30
# not accessed in 30+ days

find . -type f -size +1G -ctime -1
# last metadata change (permissions, ownership, etc.)
# -1 last 24 hours

# -size  1G exaclty
# -size -1G less then
# -size +1G greater than

# units 
# c	bytes
# k	kilobytes (1024 bytes)
# M	megabytes
# G	gigabytes
```

```bash
find /path -type f -iname "*.txt" -exec grep -liF "Keyword" {} +
# recursively searches under /path
# -type f avoids directories
# -name  case-sensitive.
# -iname case-insensitive.
# -exec execute a command on the matched files.
# grep -l print filename.
# grep -li Case-insensitive search.
# grep -liF literal match, fixed-string search

# Multiple search terms
find /path -type f -name "*.txt" -exec grep -lE "error|warning|failed" {} +
# -E enables extended regex
# "error|warning|failed" files containing at least one of those terms

# Exclude a single directory
find /path -type d -name ".git" -prune -o -type f -name "*.txt" -exec grep -l "text" {} +
# -type d  find only directories
# -prune   excludes directories
# -o       (OR) Right side runs on everything else

# Exclude multiple directories
find /path \
  \( -type d \( -name ".git" -o -name "node_modules" -o -name "proc" \) -prune \) -o \
  \( -type f -name "*.txt" -exec grep -l "text" {} + \)
# First group → directories to skip
# Second group → files to process  

# Search for multiple terms, exclude dirs, match .txt
find /var \
  \( -type d \( -name "log" -o -name "cache" \) -prune \) -o \
  \( -type f -name "*.txt" -exec grep -lE "error|failed" {} + \)

# Instead of -prune, you can filter paths:
find /path -type f -name "*.txt" \
  ! -path "*/.git/*" \
  ! -path "*/node_modules/*" \
  -exec grep -l "text" {} +

# match multiple file extensions
find /path -type f \( -iname "*.txt" -o -iname "*.log" -o -iname "*.cfg" \) -exec grep -liF "error" {} +
#or
find /path -type f -iregex '.*\.\(txt\|log\|cfg\)$' -exec grep -liF "error" {} +
# -regex  case-sensitive
# -iregex case-insensitive

#  .*\.\(txt\|log\|cfg\)$
# ---------------------------
#  .*         → any prefix
#  \.         → literal dot
#  \( ... \)  → group
#  \|         → OR
#  $          → end of string

# {} is a placeholder that gets replaced with file paths.
# + (batch mode)
find . -name "*.txt" -exec grep -l "text" {} +
# becomes
grep -l "text" ./a.txt ./b.txt ./sub/c.txt

# \; (per-file mode)
find . -name "*.txt" -exec grep -l "text" {} \;
# becomes
grep -l "text" ./a.txt
grep -l "text" ./b.txt
grep -l "text" ./sub/c.txt

# example find txt files and copy them
find . -name "*.txt" -exec cp {} {}.bak \;
# this becomes
cp ./a.txt ./a.txt.bak

# redirection of standard error (stderr)
# Suppresses error messages such as:
#   Permission denied
#   No such file or directory
find / -iname php.ini 2>/dev/null
# 2           → file descriptor for stderr
# >           → redirect operator
# /dev/null   → special device that discards all input
```

```bash
# Search everything except keyword
grep -v "keyword" file
# Count the keyword
grep -c "keyword" file
# Line number
grep -n "keyword" file
# restrict Binary files.
grep -I "keyword" file
```
```bash
# text extraction → normalization → formatting
grep keyword file | cut -d: -f1,5 | sort | tr “:” “ “ | column -t
# grep searches for "keyword" inside file. 
# cut -d: -f1,5
# cut grabs specific "columns" of data,  -d: columns are separated by a colon, 
# -f1,5  extract only the 1st and 5th fields
# sort puts the lines in alphabetical order based on the first character.
# tr ":" " " translate. It looks for every colon : and replaces it with a space.
# column -t This formats the input into a table 

#example
# Avoid tr by letting column handle delimiter
grep error log.txt | cut -d: -f1,5 | sort | column -t -s:

# awk instead (more flexible)
awk -F: '/keyword/ {print $1, $5}' file | sort | column -t
```


------------------------------------------------------------------------

## Disk & Memory

system's disk space usage
``` bash
df -ahT --total

# -a All
# -h Human readable
# -T List mounted and types
# --total sums up the "Used" and "Available"
```

```bash
du -sh /var/log/*

# -a count all file and directories
# -s summarize
# -h human readable
# -c grand total
```

```bash
# Show memory and swap usage
free
# finds Executable, config, and docs.
whereis app
# Show which app will be run by default
which app
```

Tip: "shortcut" by adding an alias

```bash
source ~/.bashrc
```

```bash
alias dft='df -ahT --total'
```

------------------------------------------------------------------------

## Compression

``` bash
# Compress
tar -zcvf archive.tar.gz /dir
# Extract
tar -xzf archive.tar.gz

# -c Create archive
# -x Extract

# -z gzip compression .tar.gz
# -j bzip2 compression .tar.bz2 *Better compression, slower*
# -J xz compression .tar.xz *Best compression, slowest*

# -v verbosely
# -f File/filename

# -t List files in archive
# -r Add to Archive without recreating
# -u Create & Add to existing archive
# -A multiple archives into single archive
# -W Verifies archive integrity

# -p Preserve permissions Important for system backups
# -k Don’t overwrite on extract
```

--exclude

```bash
tar --exclude='/dir/folder_to_skip' -zcvf archive.tar.gz /dir
```

```bash
tar --exclude='node_modules' --exclude='.git' -zcvf project.tar.gz ./project
```

Exclude Using a List File
```bash
nano exclude_list.txt
```

```bash
# Exclude specific folders
node_modules/
temp_data/
.git
cache
# Exclude specific file types
*.mp4
*.zip
*.log
# Exclude a specific file
secret_key.pem
```

```bash
tar -zcvf archive.tar.gz -X exclude_list.txt /dir
```

--strip-components
```bash
tar -xzvf archive.tar.gz --strip-components=1
# remove a specific number of leading directories when extracting a tar archive
```

**Examples:**
```bash
# z: Decompresses a gzip archive while protecting local files.
tar -xzkf archive.tar.gz
# j: Decompresses a bzip2 archive while protecting local files.
tar -xjkf archive.tar.bz2
# Extract to specific directory
tar -xzf backup.tar.gz -C /tmp/
# List contents without extracting
tar -tzf backup.tar.gz
# Compress and Preserve permissions
tar -czpf backup.tar.gz /etc/
```

------------------------------------------------------------------------

## File Listing


``` bash
ls -a
# -a  All inc hidden
# -l  one File per line
# -h  Human readable
# -m  list all separated by a comma
# -S  Sort by file size,Largest first
# -X  Sort Asc by extension
# -Q  enclose with double quotes
# -R  list subdirectories recursively
# -i  print index number
# -lt sort by ctime
```



------------------------------------------------------------------------

## Systemctl

Shows only active (running) units
``` bash
systemctl list-units --state running
```

``` bash
systemctl list-units --all --type=service --no-pager
# --type=service only services
# --all include inactive + failed
# --no-pager print everything directly
```

Lists all installed unit files on disk

``` bash
systemctl list-unit-files --no-pager
# includes services that: are not currently loaded, have never been started
```

``` bash
# Check what’s running
systemctl list-units --state=running
# Check if something failed
systemctl list-units --failed
# Check if a service is enabled at boot
systemctl is-enabled nginx
```

------------------------------------------------------------------------

## Users

basic commands

``` bash
sudo su
su username
passwd username
adduser username
deluser username
```
Who is Logged In & who is online
``` bash
whoami
who
w
```

List Users
``` bash
compgen -u
```

``` bash
# All stored user info
cat /etc/passwd
# specific user info
cat /etc/passwd | grep username
```

grant sudo rights
``` bash
sudo usermod -aG sudo username
# user can run docker without sudo
sudo usermod -aG docker username

# -a (Append)
# -G (Groups)
```

```bash
visudo
```

```bash
username ALL=(ALL:ALL) NOPASSWD: ALL
username ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
username ALL=(ALL:ALL) NOPASSWD: /usr/bin/apt, /usr/bin/systemctl
```
Change Age (chage)
```bash
# see the current password expiry status
chage -l username
# set expiration date
chage -E 2026-12-31 username
# "freeze" an account without deleting their files
chage -E 0 username
# Password Never Expires
chage -m 0 -M 99999 -I -1 -E -1 username
# -m minimum days between password change, 0 is anytime
# -M days pass is valid, 99999 is 273 years
# -I inactive days, -1 negative one, disables this lock
# -E expiration date, -1 negative one, never expires
```

passwd
```bash
passwd -S username

# Example Output:
username L 04/30/2026 0 90 7 28
# L (Locked), P (Password set/Usable), or NP (No Password)
# 04/30/2026 Date the password was last updated.
# 0 Days before the user is allowed to change it again.
# 90 (expiry) Days before the password must be changed
# 7 Days before expiry that the user gets a warning
# 28 Days after expiry before the account is disabled.

# Lock user password
passwd -l username
# Unlock user password
passwd -u username
# forces user to change password every 90 days
passwd -x 90 username
# must wait at least 5 days before they can change password
passwd -n 5 username
# warn 7 days before the password expires
passwd -w 7 username
# forces password change on next login
passwd -e

# will refuse to unlock an account that has no password
# if double locked will remove one exclamation point
```

```bash
grep username /etc/shadow
# example outputs:
# ! Locked with passwd -l or usermod -L
username:!$6$somesalt$somehash...:19850:0:90:7:::
#!! Double Locked or created without a password
username:!!:19850:0:90:7:::

# hashing algorithm:
# $1$: MD5 (Ancient, do not use)
# $2a$: Blowfish (bcrypt)
# $5$: SHA-256
# $6$: SHA-512  (user on a fresh ubuntu 24.04 install)
# $y$: Yescrypt (The new Ubuntu default)
```
Failed login 
```bash
faillock --user username
# Clear all failed login records
faillock --user username --reset
```
usermod
```bash
# Lock user
usermod -L username
# Unlock user
usermod -U username

# Fully lock an account (including SSH key access)
usermod -s /sbin/nologin username
# for system service accounts (like www-data or mysql) 
# that need to own files and run processes 
# but should never be used by a human to log in.
```



------------------------------------------------------------------------

## Groups

list groups from file
``` bash
cat /etc/group

# list only group names
cut -d: -f1 /etc/group

# list groups created by users
awk -F: '$3 >= 1000 {print $1}' /etc/group
```


get entries, talks to Name Service Switch (NSS)
``` bash
getent group
# list "sudo" users
getent group sudo | cut -d: -f4
```

list groups for user
``` bash
groups username
```


``` bash
groupadd webmasters
# audo assign GID

groupadd -r sysadmins
# -r (--system) Flag
# It chooses a Group ID (GID) from the system range. 
# regular groups usually start at GID 1000. 
# System groups use a lower range (typically 100–999).

groupadd -g 2000 dbadmin
# -g GID --gid
# Forces the group to have a specific ID number.
# for Matching GIDs across multiple servers.
```

------------------------------------------------------------------------

## Cron Jobs

``` bash
# Edit
crontab -e
# List
crontab -l
# Remove
crontab -r
# List user crontab file
crontab -u username -l
# Edit user crontab file
crontab -u username
```

Format:

```
* * * * * command(s)
| | | | |
| | | | ----- Day of week (0 - 7) (Sunday=0 or 7)
| | | ------- Month (1 - 12)
| | --------- Day of month (1 - 31)
| ----------- Hour (0 - 23)
------------- Minute (0 - 59)
```

Examples:

``` bash
# Reboot Server Monthly
0 4 1 * * /sbin/reboot
# database backup every single night runs every day at 2:30 AM.
30 2 * * * /usr/bin/mysqldump -u root -pYourPass db_name > /backups/db_$(date +\%F).sql
30 2 * * * PGPASSWORD='YourPass' /usr/bin/pg_dump -U root -d db_name > /backups/db_$(date +\%F).sql
# Clear Temporary Files Every Hour
0 * * * * /usr/bin/find /tmp -type f -mtime +1 -delete
# Restart nginx Every Night
0 3 * * * /usr/bin/systemctl restart nginx
# Run a Python Script Every 10 Minutes 
# Polling APIs, scraping data, IoT ingestion.
*/10 * * * * /usr/bin/python3 /home/user/scripts/data_collector.py
# Weekly Log Rotation (Sunday at Midnight)
0 0 * * 0 /usr/sbin/logrotate /etc/logrotate.conf
# Send a Health Check Ping Every 5 Minutes
*/5 * * * * curl -fsS https://example.com/health || echo "FAIL" >> /var/log/health.log
# Delete Old Backups (Older Than 7 Days)
0 5 * * * /usr/bin/find /backups -type f -mtime +7 -delete
# Sync Files to Remote Server (Every Night)
0 1 * * * /usr/bin/rsync -avz /local/dir user@remote:/backup/
# Run Script Only on Weekdays (9 AM) 
# script should have chmod +x
0 9 * * 1-5 /home/user/work/report.sh
# Run Every 2 Hours Between 8 AM–8 PM
0 8-20/2 * * * /home/user/scripts/task.sh
```
Cron Keywords
```bash
# Run Script on Server Boot
@reboot /home/user/startup.sh
# Hourly Log Cleanup
@hourly /usr/bin/find /var/log/myapp -type f -size +100M -delete
# Daily Backup at Midnight
@daily /usr/local/bin/backup.sh
# Restart Service Daily
@daily /usr/bin/systemctl restart nginx
# Ensure containers recover after system restart
@reboot /usr/bin/docker start my_container
# Automated backups to remote server
@daily /usr/bin/rsync -av /data user@backup:/data

# @reboot                           Run once at startup
# @yearly               0 0 1 1 *   Once a year (Jan 1)
# @annually             0 0 1 1 *   Same as yearly
# @monthly              0 0 1 * *   First day of month
# @weekly               0 0 * * 0   Every Sunday
# @daily                0 0 * * *   Every day at midnight
# @midnight             0 0 * * *   Same as daily
# @hourly               0 * * * *   Every hour

# Every minute          * * * * *
# Every 5 minutes       */5 * * * *
# Every 15 minutes      */15 * * * *
# Every day at 9 AM     0 9 * * *
# Weekdays at 9 AM      0 9 * * 1-5
```

Create password for postgresql 
```bash
nano ~/.pgpass
```

```bash
localhost:5432:db_name:postgres:YourActualPassword
# or
*:*:*:postgres:YourActualPassword
```

```bash
chmod 0600 ~/.pgpass
```

```bash
30 2 * * * /usr/bin/pg_dump -U postgres -d db_name > /backups/db_$(date +\%F).sql
```


------------------------------------------------------------------------

## SSH

``` bash
ssh-keygen -t ed25519 -C "user"
nano ~/.ssh/authorized_keys
```

------------------------------------------------------------------------

## Firewall

``` bash
ufw allow 22/tcp
```
``` bash
ufw allow 5432/tcp
```
``` bash
ufw status numbered
```
``` bash
ufw delete 3
```
``` bash
ufw status verbose
```
``` bash
journalctl -xe | grep ufw
```


