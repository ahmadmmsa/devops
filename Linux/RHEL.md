# RHEL Hardening & Compliance

Security/compliance toolkit for Red Hat Enterprise Linux (and rebuilds: Rocky, Alma, CentOS Stream).
Mandatory access control is **SELinux** (not AppArmor); compliance is scanned/remediated with
**OpenSCAP** against a profile (STIG, CIS, PCI-DSS, …); crypto can be put into **FIPS** mode.

## Contents
- [OpenSCAP](#openscap) — scan & remediate against a baseline
- [SELinux](#selinux) — mandatory access control
- [AppArmor](#apparmor) — not on RHEL (see note)
- [FIPS 140-2/3 validation](#fips-140-23-validation)
- [Common Criteria EAL4+](#common-criteria-eal4)
- [DoD STIG profiles](#dod-stig-profiles)

---

# OpenSCAP

Scans the system against an SCAP **profile** (STIG/CIS/etc.) shipped in the
`scap-security-guide` package, reports pass/fail per rule, and can auto-remediate.

```bash
sudo dnf install -y openscap-scanner scap-security-guide   # scanner + the SSG content

# where the content lives, and what's inside
ls /usr/share/xml/scap/ssg/content/                         # ssg-rhel9-ds.xml = the datastream
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml # lists profiles + their IDs
```

```bash
# SCAN against a profile -> HTML report you can open in a browser
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis \
  --results-arf  arf.xml \                # machine-readable results (for re-use below)
  --report       report.html \            # human-readable: red = fail, with fix text
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
#   exit 0 = all pass, 2 = some rules failed (expected on first run), 1 = error

# scan a single rule (fast iteration while fixing)
sudo oscap xccdf eval --profile cis --rule xccdf_org.ssgproject.content_rule_package_aide_installed \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

```bash
# REMEDIATE — apply fixes for whatever failed
sudo oscap xccdf eval --profile cis --remediate \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml   # changes the live system in place

# Safer: generate a remediation script/playbook, review it, THEN run
oscap xccdf generate fix --fix-type bash     --profile cis --result-id "" arf.xml > fix.sh
oscap xccdf generate fix --fix-type ansible  --profile cis --result-id "" arf.xml > fix.yml
sudo ansible-playbook fix.yml                # idempotent, version-controllable, repeatable
```

> **Gotcha:** `--remediate` edits sshd/pam/auditd/sysctl live — it *can* lock you out (e.g.
> disables root SSH, tightens ciphers). Always scan → read `report.html` → review the generated
> `fix.sh`/`fix.yml` before applying, and keep console access. Re-scan after to confirm.

# SELinux

Mandatory access control. Every process/file has a **context** `user:role:type:level`; the
policy decides what a **type** (domain) may touch. Denials are logged, not silent.

```bash
getenforce                      # Enforcing | Permissive | Disabled
sestatus                        # full status: mode, policy, mount

setenforce 1                    # -> Enforcing now (runtime only)
setenforce 0                    # -> Permissive: log denials but allow (debugging)
sudo nano /etc/selinux/config   # SELINUX=enforcing  <- persists across reboot
```

```bash
# see contexts (the -Z flag is everywhere)
ls -Z /var/www/html             # file contexts (httpd_sys_content_t etc.)
ps -auxZ | grep nginx           # process domain
id -Z                           # your context

# booleans = supported on/off knobs (no policy editing needed)
getsebool -a | grep httpd                         # list httpd booleans
sudo setsebool -P httpd_can_network_connect on    # -P = persist; let httpd make outbound conns
```

```bash
# FIXING denials — the real day-to-day work
sudo ausearch -m AVC,USER_AVC -ts recent          # raw denials from the audit log
sudo sealert -a /var/log/audit/audit.log          # human explanation + suggested fix
sudo dnf install -y setroubleshoot-server          # provides sealert

# relabel a file/dir to the right context (e.g. serving content from a non-default path)
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"  # add a persistent rule
sudo restorecon -Rv /srv/web                       # apply it to existing files
chcon -t httpd_sys_content_t /srv/web/index.html   # one-off, NON-persistent (lost on relabel)

# allow a non-standard port for a service
sudo semanage port -a -t http_port_t -p tcp 8888

# last resort: build a policy module from logged denials (only after confirming it's legit)
sudo ausearch -m AVC -ts recent | audit2allow -M mypol && sudo semodule -i mypol.pp
```

> **Gotcha:** don't reach for `setenforce 0` to "fix" an app — that disables protection
> system-wide. 90% of denials are a wrong file context → `restorecon`, or a missing boolean →
> `setsebool -P`. Use `chcon` only for quick tests; it doesn't survive a filesystem relabel.

# AppArmor

**Not used on RHEL.** AppArmor is the MAC system on Debian/Ubuntu/SUSE; RHEL ships **SELinux**
instead (see above) and AppArmor is not packaged or supported. If you're hardening a RHEL host,
do it through SELinux. Cheat-sheet for the Debian/Ubuntu side, for cross-reference:

```bash
# (Ubuntu/Debian only — won't apply on RHEL)
sudo aa-status                         # loaded profiles + enforce/complain counts
sudo aa-enforce  /etc/apparmor.d/usr.sbin.nginx   # enforce a profile
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx   # log-only (like SELinux permissive)
sudo aa-logprof                        # interactively update a profile from denials
```

| | RHEL → SELinux | Ubuntu → AppArmor |
|---|---|---|
| Model        | label/type based (every object)     | path based (per-program profile) |
| Mode toggle  | `setenforce 0/1`                    | `aa-complain` / `aa-enforce`     |
| Debug        | `ausearch` / `sealert`              | `aa-logprof`                     |

# FIPS 140-2/3 validation

Puts the OS crypto libraries into FIPS mode so only validated algorithms are used — required
for US federal / regulated workloads. One command flips it system-wide.

```bash
# enable (RHEL 8/9) — sets the FIPS crypto policy + initramfs flag, then REBOOT
sudo fips-mode-setup --enable
sudo reboot

# verify after reboot
fips-mode-setup --check          # "FIPS mode is enabled."
cat /proc/sys/crypto/fips_enabled # 1 = on
update-crypto-policies --show     # should report FIPS

# disable
sudo fips-mode-setup --disable && sudo reboot
```

> **Gotcha:** enable FIPS **at install time** if you can (kickstart `fips=1`) — turning it on
> after the fact doesn't retroactively validate keys/certs generated earlier (e.g. SSH host keys
> made with a now-disallowed algorithm). FIPS bans MD5, some curves, short RSA — legacy clients,
> old TLS, and MD5 checksums will break. Test in staging first.

# Common Criteria EAL4+

A formal product certification (the *evaluated configuration*), not something you "install".
RHEL is CC-certified at **EAL4+** on specific hardware + a specific minor release. You don't run
a command to "be EAL4"; you reproduce the evaluated configuration and prove it.

```bash
# confirm exact version/build (must match the certified one in the cert's docs)
cat /etc/redhat-release
uname -r

# the evaluated config is, in practice: SELinux enforcing + FIPS on + audit configured
getenforce                                   # Enforcing
fips-mode-setup --check                       # enabled
sudo systemctl is-active auditd               # active — the audit trail is mandatory for CC
sudo auditctl -s                              # audit subsystem status
```

- The certified version, hardware, and required settings live in the **CC Security Target /
  Configuration Guide** Red Hat publishes per certification — match it exactly.
- In practice CC = SELinux enforcing + FIPS + a locked-down auditd policy + documented procedures.
- Check the validated list: <https://www.commoncriteriaportal.org/products/>

# DoD STIG profiles

DISA STIG = the US DoD hardening baseline. Shipped as an OpenSCAP profile, so you scan/remediate
it the same way as any other profile — just point at the STIG profile ID.

```bash
sudo dnf install -y openscap-scanner scap-security-guide

# find the exact STIG profile ID for your release
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml | grep -i stig
#   xccdf_org.ssgproject.content_profile_stig      (full STIG)
#   xccdf_org.ssgproject.content_profile_stig_gui  (STIG for systems with a GUI)
```

```bash
# SCAN against STIG
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --results-arf stig-arf.xml --report stig-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# REMEDIATE via reviewed Ansible (preferred over --remediate for STIG)
oscap xccdf generate fix --fix-type ansible \
  --profile xccdf_org.ssgproject.content_profile_stig \
  --result-id "" stig-arf.xml > stig-fix.yml
sudo ansible-playbook stig-fix.yml

# Red Hat also ships ready-made STIG playbooks:
ls /usr/share/scap-security-guide/ansible/ | grep stig    # rhel9-playbook-stig.yml
sudo ansible-playbook /usr/share/scap-security-guide/ansible/rhel9-playbook-stig.yml
```

Apply STIG at install time (kickstart) so the box is born compliant:
```
# in the kickstart %addon section
%addon org_fedora_oscap
  content-type = scap-security-guide
  profile = xccdf_org.ssgproject.content_profile_stig
%end
```

> **Gotcha:** STIG is aggressive — it disables ctrl-alt-del, enforces password/lockout policy,
> kills weak ciphers, sets `pam_faillock`, mandates partition mount options (`/tmp` noexec, etc.).
> Remediating a *running* prod box can lock out admins or break apps. Scan first, read
> `stig-report.html`, remediate in staging, and bake it into the image/kickstart rather than
> retrofitting. Re-scan to produce the evidence/score auditors want.
