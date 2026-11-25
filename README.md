*This project has been created as part of the 42 curriculum by ebayle.*

# Born2beRoot

## Description

> Don't want to install Linux? Just virtualize it brother.

Born2beRoot is a system administration project. The goal is to set up a minimal, secure Linux server inside a virtual machine, following strict rules: encrypted LVM partitions, a hardened SSH service, a firewall, a strong password policy, a strict sudo configuration and a monitoring script broadcast to every terminal.

This server runs **Debian 13 "trixie"** in **VirtualBox**, with no graphical interface. The bonus part is also done: a detailed partition layout, a WordPress site (lighttpd + MariaDB + PHP) and an extra service (Fail2ban).

## Project description

### Choice of operating system

I chose **Debian (stable)** over Rocky Linux. It is the distribution recommended for beginners in system administration, it has a huge community and documentation, and its package manager (`apt`) and default security module (AppArmor) are simpler to configure than their Rocky equivalents.

### Design choices

#### Partitioning

The 30 GB virtual disk is split into a small unencrypted `/boot` partition and a logical partition encrypted with LUKS (`sda5_crypt`). Inside it, a single LVM volume group (`LVMGroup`) holds every other partition:

| Partition | Size | Mount point | Why it is separate |
|---|---|---|---|
| sda1 | 500 MB | `/boot` | Must stay unencrypted so GRUB can load the kernel |
| LVMGroup-root | 10 GB | `/` | System and installed packages |
| LVMGroup-swap | 2.3 GB | `[SWAP]` | Safety margin when RAM is full |
| LVMGroup-home | 5 GB | `/home` | User data, isolated from the system |
| LVMGroup-var | 3 GB | `/var` | Variable data (web files, MariaDB database) |
| LVMGroup-srv | 3 GB | `/srv` | Data served by the server's services |
| LVMGroup-tmp | 3 GB | `/tmp` | Temporary files can't fill the root partition |
| LVMGroup-var--log | ~3.2 GB | `/var/log` | Logs that grow out of control can't fill `/var` or `/` |

LVM makes it possible to resize these volumes later without repartitioning the disk, and the encryption protects the data if the disk file is stolen.

#### Security policies

- **SSH**: runs on port **4242** only, and `PermitRootLogin no` forbids logging in as root.
- **Firewall (UFW)**: enabled at boot; only port **4242** (SSH) and port **80** (WordPress, bonus) are open.
- **AppArmor**: enabled at boot (Debian default).
- **Password policy**:
  - in `/etc/login.defs` and with `chage`: passwords expire every 30 days, the minimum time between changes is 2 days, and a warning is shown 7 days before expiry
  - in `libpam-pwquality` (`/etc/pam.d/common-password`): at least 10 characters, with an uppercase letter, a lowercase letter and a digit; no more than 3 identical consecutive characters; the username is not allowed; 7 characters must differ from the old password (not checked for root); the rules also apply to root (`enforce_for_root`)
- **sudo** (`/etc/sudoers.d/sudo_config`):
  - 3 password attempts maximum, with a custom error message
  - every command is logged, inputs and outputs included, in `/var/log/sudo/`
  - `requiretty` is enabled
  - `secure_path` restricts the paths sudo can use

#### User management

- `root` and the user `ebayle` both exist.
- `ebayle` belongs to the groups `sudo` and `user42`.
- After the configuration was done, every password (root included) was changed to follow the new policy.

#### Installed services

- `openssh-server`: remote administration
- `ufw`: firewall
- `sudo`: privilege management
- `libpam-pwquality`: password strength rules
- `cron`: runs `monitoring.sh` at boot and every 10 minutes
- Bonus: `lighttpd`, `mariadb-server`, `php-cgi` (+ `php-mysql`), WordPress, `fail2ban`

### Comparisons

#### Debian vs Rocky Linux

| | Debian | Rocky Linux |
|---|---|---|
| Origin | Community project, one of the oldest distributions | Free rebuild of Red Hat Enterprise Linux (RHEL) |
| Package manager | `apt` / `dpkg` (`.deb`) | `dnf` / `rpm` (`.rpm`) |
| Security module | AppArmor | SELinux |
| Strengths | Easy to learn, very stable, huge documentation | Close to RHEL, widely used in companies |
| Weaknesses | Packages in stable can be quite old | More complex to configure for beginners |

#### AppArmor vs SELinux

Both are Mandatory Access Control (MAC) systems: they limit what each program can do, even when it runs as root.
- **AppArmor** uses profiles based on **file paths**. It is simpler to read and write, and it is the default on Debian and Ubuntu.
- **SELinux** gives a **label** (context) to every file, process and port. It is more precise and more powerful, but also much more complex. It is the default on Red Hat-based systems such as Rocky.

#### UFW vs firewalld

Both are front-ends to the Linux kernel firewall (netfilter: iptables/nftables).
- **UFW** (Uncomplicated Firewall) uses simple commands (`ufw allow 4242`) and is ideal for a small server.
- **firewalld** is built around **zones** (public, home, trusted…) and can change rules at runtime without a restart. It is more flexible, but more complex. It is the default on Rocky.

#### VirtualBox vs UTM

- **VirtualBox** (Oracle) is a free hypervisor for Windows, Linux and Intel Macs. It is mature and has lots of features (snapshots, port forwarding, shared folders).
- **UTM** is built on QEMU and made for macOS, especially Apple Silicon (M1/M2…) Macs, where VirtualBox support is limited. It can also emulate other architectures.

I used VirtualBox, because it is the tool required by the subject on a compatible machine.

## Instructions

### Starting the server

1. Import or open the VM in VirtualBox and start it.
2. Enter the disk encryption passphrase.
3. Log in as `ebayle`.

### Connecting with SSH from the host

In VirtualBox, go to Settings → Network → Adapter 1 (NAT) → Port Forwarding, and add a rule from host port `4243` to guest port `4242`. Then run, from the host:

```bash
ssh ebayle@localhost -p 4243
```

### Useful checks

```bash
head -n 2 /etc/os-release        # OS version
sudo aa-status                   # AppArmor status
sudo ss -tunlp                   # SSH listening on 4242
sudo ufw status numbered         # firewall rules
lsblk                            # partitions
sudo chage -l ebayle             # password expiry policy
groups ebayle                    # sudo + user42
hostnamectl                      # hostname (ebayle42)
sudo crontab -u root -l          # monitoring schedule
```

### Monitoring script

The script is `/usr/local/bin/monitoring.sh`. root's crontab runs it at boot (`@reboot`) and every 10 minutes (`*/10 * * * *`). It uses `wall` to broadcast the following information:
- the architecture and kernel version
- the number of physical and virtual CPUs
- RAM, disk and CPU usage
- the date and time of the last boot
- whether LVM is used
- the number of active TCP connections
- the number of logged-in users
- the IP and MAC addresses
- the number of commands run with sudo

To stop it without changing the script:

```bash
sudo systemctl stop cron      # or comment out the lines in: sudo crontab -u root -e
```

### Bonus

- **WordPress**: lighttpd serves the files in `/var/www/html`, PHP runs through FastCGI (`lighty-enable-mod fastcgi fastcgi-php`), and MariaDB stores the `wordpress` database, which is accessed by a dedicated `wpuser` account. With a port forward from host `8080` to guest `80`, the site is available at `http://localhost:8080`.
- **Fail2ban** (extra service): it reads SSH login failures from the systemd journal and bans an IP for 10 minutes after 3 failures within 10 minutes (`/etc/fail2ban/jail.local`). I chose it because the server exposes SSH, so blocking brute-force attacks adds a real layer of protection on top of UFW and the password policy.

  ```bash
  sudo fail2ban-client status sshd
  sudo fail2ban-client set sshd unbanip <IP>
  ```

### Submission

The repository only contains this `README.md` and `signature.txt`, which holds the SHA1 of the VM disk:

```bash
sha1sum ~/VirtualBox\ VMs/Born2beRoot/Born2beRoot.vdi
```

## Resources

- [The Debian Administrator's Handbook](https://www.debian.org/doc/manuals/debian-handbook/)
- [Debian Wiki: LVM](https://wiki.debian.org/LVM) and [AppArmor](https://wiki.debian.org/AppArmor)
- [Debian man pages](https://manpages.debian.org/): `sudoers(5)`, `sshd_config(5)`, `pam_pwquality(8)`, `login.defs(5)`, `chage(1)`, `wall(1)`, `crontab(5)`
- [ArchWiki: Sudo](https://wiki.archlinux.org/title/Sudo)
- [Ubuntu community help: UFW](https://help.ubuntu.com/community/UFW)
- [firewalld documentation](https://firewalld.org/documentation/)
- [Rocky Linux documentation](https://docs.rockylinux.org/)
- [VirtualBox manual](https://www.virtualbox.org/manual/)
- [lighttpd wiki](https://redmine.lighttpd.net/projects/lighttpd/wiki)
- [WordPress documentation](https://wordpress.org/documentation/)
- [Fail2ban on GitHub](https://github.com/fail2ban/fail2ban)

### AI usage

AI usage: Claude was used as a peer reviewer as a tester, it help us to writing this README and to understand wordpress.

Every configuration was tested on the VM and understood before the defense.