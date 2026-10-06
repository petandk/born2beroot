*This project has been created as part of the 42 curriculum by rmanzana*

# Born2beRoot

## Description

**Born2beRoot** is a system administration project. The goal is to set up a secure Linux server inside a virtual machine, following strict rules: encrypted partitions with LVM, a hardened SSH service, a firewall, a strong password policy, a strict `sudo` configuration, and a monitoring script that broadcasts the state of the system every 10 minutes.

The project does not produce a program: it is about understanding how a server is installed, partitioned, secured and administered from the command line, without a graphical interface.

This repository contains:

| File | Content |
|---|---|
| `monitoring.sh` | The monitoring script run by `cron` every 10 minutes |
| `born2beroot.txt` | Step-by-step setup notes |

## Project description

### Operating system choice: Debian

The server runs the latest stable version of **Debian**, without a graphical environment.

| | Debian | Rocky Linux |
|---|---|---|
| **Origin** | Community-driven, one of the oldest distributions | Enterprise distribution, binary-compatible with Red Hat Enterprise Linux |
| **Pros** | Very stable, huge documentation and community, simple to install and administer, `apt` package manager | Enterprise-grade, same ecosystem as RHEL, SELinux enabled by default, long support cycles |
| **Cons** | Packages are older in the stable branch | More complex setup (SELinux, KDump), smaller community for beginners |

Debian was chosen because it is the recommended option for people new to system administration: its installation is straightforward, and its documentation makes it easier to understand each configuration step.

### Main design choices

- **Partitioning:** manual partitioning with an unencrypted `/boot` partition and an **encrypted partition (LUKS)** containing an **LVM** volume group, split into logical volumes for `root`, `swap`, `home`, `var`, `srv`, `tmp` and `var/log`. Separate volumes prevent, for example, logs filling up `/var/log` from bringing down the whole system, and LVM allows resizing them later.
- **SSH:** the service listens on port **4242** only, and logging in as `root` over SSH is disabled.
- **Firewall:** **UFW** is enabled and only port 4242 is open.
- **Password policy:**
  - Passwords expire every 30 days (`PASS_MAX_DAYS 30`), cannot be changed more than once every 2 days (`PASS_MIN_DAYS 2`), and a warning is shown 7 days before expiry (`PASS_WARN_AGE 7`).
  - With `libpam-pwquality`: at least 10 characters, one uppercase letter and one digit, no more than 3 identical consecutive characters, no username inside the password, and at least 7 characters different from the previous password.
- **sudo:** 3 attempts maximum, a custom error message, every command logged in `/var/log/sudo/` (input and output), `requiretty` enabled, and a restricted `secure_path`.
- **User management:** a user named after the login belongs to the `user42` and `sudo` groups.
- **Monitoring:** `monitoring.sh` is run by `cron` every 10 minutes and its output is broadcast to every terminal with `wall`.

### Comparisons

**Debian vs Rocky Linux** — see the table above. Debian prioritises simplicity and stability for general use; Rocky prioritises compatibility with the Red Hat enterprise ecosystem.

**AppArmor vs SELinux**

| | AppArmor | SELinux |
|---|---|---|
| **Model** | Path-based: profiles restrict what each program can access by file path | Label-based: every file, process and port has a security context |
| **Default on** | Debian, Ubuntu | Rocky, RHEL, Fedora |
| **Complexity** | Easier to read and write profiles | More powerful and granular, but much harder to configure |

Both are **Mandatory Access Control** systems: even if a program is compromised, it can only do what its policy allows. Debian ships with AppArmor enabled.

**UFW vs firewalld**

| | UFW | firewalld |
|---|---|---|
| **Goal** | "Uncomplicated Firewall": a simple front-end for `iptables`/`nftables` | A dynamic firewall manager based on zones |
| **Usage** | `ufw allow 4242` | `firewall-cmd --add-port=4242/tcp --permanent` |
| **Default on** | Debian, Ubuntu | Rocky, RHEL, Fedora |

UFW is simpler for a single server with a few open ports; firewalld is more flexible when different network interfaces need different trust levels (zones), and it can change rules without restarting the service.

**VirtualBox vs UTM**

| | VirtualBox | UTM |
|---|---|---|
| **Platforms** | Windows, Linux, macOS (Intel) | macOS only (Intel and Apple Silicon) |
| **Engine** | Oracle's own hypervisor | QEMU, and Apple's Hypervisor framework |
| **Use case** | The standard choice on x86 machines, widely documented | The practical choice on Apple Silicon Macs, where VirtualBox support is limited |

VirtualBox was used for this project, running on the campus computers (Ubuntu hosts).

## Instructions

### Setting up the virtual machine

1. Create a new VM in VirtualBox with the latest stable Debian ISO (no graphical install).
2. Use manual partitioning: `/boot` + an encrypted partition with LVM and the logical volumes listed above.
3. Install and configure `sudo`, SSH (port 4242, no root login), UFW, the password policy and the `user42` group, as described in [Main design choices](#main-design-choices).
4. Set the hostname to `<login>42`.

### Installing the monitoring script

```bash
sudo cp monitoring.sh /root/monitoring.sh
sudo chmod +x /root/monitoring.sh
sudo crontab -e
# add the line:
*/10 * * * * /root/monitoring.sh | wall
```

### Connecting to the server

```bash
ssh <user>@<vm-ip> -p 4242
```

### What the monitoring script shows

- Architecture and kernel version
- Physical and virtual CPUs
- RAM and disk usage (with percentages)
- CPU load
- Date and time of the last reboot
- Whether LVM is active
- Number of established TCP connections
- Number of logged-in users
- IPv4 and MAC address of the server
- Number of commands run with `sudo`

## Resources

- [Debian Administrator's Handbook](https://debian-handbook.info/)
- [Debian Wiki — LVM](https://wiki.debian.org/LVM) and [Cryptsetup / LUKS](https://wiki.debian.org/CryptsetupDebug)
- [UFW — Ubuntu Community Help](https://help.ubuntu.com/community/UFW)
- [`sudoers(5)` manual](https://man7.org/linux/man-pages/man5/sudoers.5.html)
- [`pam_pwquality(8)` manual](https://linux.die.net/man/8/pam_pwquality)
- [`crontab(5)` manual](https://man7.org/linux/man-pages/man5/crontab.5.html)
- [AppArmor documentation](https://apparmor.net/) and [SELinux Project](https://selinuxproject.org/)
