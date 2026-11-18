# Help Desk Home Lab: osTicket on Ubuntu Server (VMware Fusion)

A virtualized help desk environment built from scratch: an **Ubuntu Server** hosting the **osTicket** ticketing system (Apache, MySQL, PHP), with a **Windows 11** client reaching it over a shared NAT network. I built it to practice deploying and supporting a ticketing platform, and to document troubleshooting the way a help desk would: symptom, diagnosis, root cause, fix, verification.

> **Scope note:** both VMs run on one Mac inside VMware Fusion's NAT network, so this is a single-site lab, not a production deployment. No credentials are published in this repo, and the database password is redacted in the screenshots.

## Environment

| Component | Details |
|---|---|
| Host | Apple Silicon Mac, VMware Fusion |
| Server VM | Ubuntu 26.10 (interim release), 2 vCPU, 6144 MB RAM, hostname `homelab` |
| Client VM | Windows 11 (ARM), hostname `DESKTOP-UG7VF7K` |
| Network | Fusion NAT ("Share with my Mac"), subnet `192.168.157.0/24`, gateway `192.168.157.2` |
| Server address | `192.168.157.128/24` (manually configured) |
| Client address | `192.168.157.131` (DHCP from the NAT service) |
| Web stack | Apache 2.4.68, MySQL 9.x, PHP 8.5.9 |
| Application | osTicket v1.18.4 |

```
Windows 11 VM (end user)                      Ubuntu VM (osTicket server)
  192.168.157.131        --- HTTP :80 --->      192.168.157.128
  browser: portal + staff panel                 Apache + PHP + MySQL (localhost only)
              \_____ Fusion NAT 192.168.157.0/24, gateway .2 _____/
```

## What I built

### 1. Virtual network

Both VMs use the same Fusion NAT setting, so they share a private subnet and can reach each other and the internet.

![Ubuntu VM network adapter set to Share with my Mac](Screenshots/01-fusion-ubuntu-network-nat.png)
*Ubuntu VM network adapter: "Share with my Mac" (NAT).*

![Windows VM network adapter set to Share with my Mac](Screenshots/02-fusion-windows-network-nat.png)
*Windows VM network adapter: the same NAT setting.*

![Fusion overview of the Ubuntu VM](Screenshots/03-fusion-ubuntu-vm-overview.png)
*Fusion overview of the server VM: 2 cores, 6144 MB RAM, Ubuntu 26.10, IP 192.168.157.128.*

**Connectivity checks**

![Ubuntu networking output and ping tests](Screenshots/04-ubuntu-network-config-and-ping-tests.png)
*Ubuntu: one IPv4 address (`192.168.157.128/24`), default route via `192.168.157.2` (static), and 4/4 replies from google.com.*

![Windows ipconfig and ping to the server](Screenshots/05-windows-ipconfig-and-ping-to-server.png)
*Windows: `192.168.157.131/24` on a `vmxnet3` adapter, same subnet and gateway, and 4/4 replies when pinging the server.*

The ping from Ubuntu to the Windows VM shows 100% loss while the reverse direction works. That is most likely Windows Defender Firewall dropping inbound ping by default, not a network fault. The Windows-to-Ubuntu direction is the one that matters for this lab.

### 2. Web stack

Apache, MySQL, and PHP were installed on the server and verified from the Windows client.

![Listening ports and PHP version](Screenshots/06-ubuntu-listening-ports-and-php-version.png)
*`sudo ss -tlnp` shows Apache on port 80, SSH on 22, and MySQL bound to `127.0.0.1` only, so the database is not reachable from the network. `php -v` reports PHP 8.5.9.*

![Apache default page loaded from the Windows VM](Screenshots/07-windows-apache-default-page.png)
*The Apache default page loading in the Windows VM's browser at `http://192.168.157.128`.*

### 3. Database with least privilege

I created a dedicated `osticket` database (utf8mb4) and an `osticket` account limited to that database, instead of letting the application use MySQL's root account.

![MySQL setup and login test](Screenshots/08-mysql-database-user-and-login-test.png)
*Database creation, user creation (password redacted), grants limited to `osticket.*`, and a login test. The first login attempt was denied; the second succeeded and lists only the built-in schemas plus `osticket` (details in the problems log).*

### 4. osTicket installation

![Installer reporting the config file missing](Screenshots/09-osticket-installer-config-file-missing.png)
*The installer (v1.18.4) reporting that `include/ost-config.php` did not exist yet.*

![osTicket install complete](Screenshots/10-osticket-install-success.png)
*After creating the config file from the sample, the installer completed and gave the portal and staff panel URLs.*

### 5. Hardening after install

| Check | Before | After |
|---|---|---|
| `include/ost-config.php` permissions | `rw-rw-rw-` (world-writable, required by the installer) | `rw-r--r--` |
| Installer directory `setup/` | Present | Removed |
| Web root `/var/www/html/osticket` | `drwxrwxrwx` (777) | `drwxr-xr-x` (755) |
| Database account | n/a | Dedicated user limited to one database, localhost only |

![Config file permissions before](Screenshots/11-config-permissions-before.png)
*Config file permissions right after install: `-rw-rw-rw-`.*

![Config permissions after and setup removed](Screenshots/12-config-permissions-after-setup-removed.png)
*`chmod 0644` applied, `setup` directory deleted, and the directory listing no longer shows it.*

![Web root listing showing 777 permissions](Screenshots/13-web-root-listing-777-permissions.png)
*Audit finding: the web root directory itself was `drwxrwxrwx`. Fixing it led to the 403 incident described below.*

### 6. Verification

![Staff control panel](Screenshots/14-osticket-staff-panel.png)
*Staff control panel after install, showing the default "osTicket Installed!" ticket.*

![Customer support portal](Screenshots/15-osticket-customer-portal.png)
*Customer-facing Support Center loading from the Windows VM.*

## Troubleshooting summary

Every issue I hit is documented in [`docs/problems-and-fixes.md`](docs/problems-and-fixes.md) with screenshots, root cause, fix, and verification.

| # | Problem | Root cause | Result |
|---|---|---|---|
| 1 | Windows setup had no usable network adapter | Installer lacked a driver for the VM's virtual NIC | Network working, client on the server's subnet |
| 2 | Package install failed with a `dpkg` error | Root filesystem was 100% full | Logical volume extended from about 10 GB to 37 GB |
| 3 | `systemctl` reported a degraded system state | Two failed services (`dnsmasq`, `apparmor`) | State back to `running`, 0 failed units |
| 4 | New MySQL account was denied on first login | See log | Login succeeded, verified least-privilege access |
| 5 | osTicket installer said the config file was missing | `ost-config.php` had not been created from the sample | Install completed |
| 6 | Hardening the web root caused a 403 Forbidden | A `chmod 644` was applied to directories instead of files, removing their execute bit | Permissions corrected, site restored |

## Skills demonstrated

- **Linux administration:** package management, `systemctl` service checks, `ss` port inspection, LVM and partition resizing (`growpart`, `pvresize`, `lvextend`), file ownership and permissions
- **Troubleshooting method:** reading logs and command output, narrowing the cause (for example `namei -l` to trace permissions along a path), verifying each fix
- **Networking:** NAT, subnets and gateways, manual vs DHCP addressing, connectivity testing (`ping`, `ipconfig`, `ip route`)
- **Databases and security:** MySQL user creation, least-privilege grants, localhost-only binding, post-install hardening
- **Application deployment:** Apache, PHP, and osTicket installation and verification from a separate client
- **Documentation:** a problems log with evidence for each incident
