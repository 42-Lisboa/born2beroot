*This project has been created as part of the 42 curriculum by jcosta-a.*

# 🌳 Born2beRoot

<p align="left">
  <img src="https://img.shields.io/badge/OS-Debian-A81D33?style=for-the-badge&logo=debian&logoColor=white" />
  <img src="https://img.shields.io/badge/Hypervisor-VirtualBox-183A61?style=for-the-badge&logo=virtualbox&logoColor=white" />
  <img src="https://img.shields.io/badge/Bonus-Partitions-blue?style=for-the-badge" />
</p>

> First contact with system administration: a Debian server inside VirtualBox, with encrypted LVM partitions, SSH on port 4242, UFW, a strong password policy, a hardened `sudo` and a `monitoring.sh` script that broadcasts system info every 10 minutes.

## 📋 Summary

- [Description](#description)
- [Instructions](#instructions)
- [Evaluation](#evaluation)
- [Resources](#resources)

<a name="description"></a>
<details open><summary><h2>📖 Description</h2></summary>

Born2beRoot is my first real contact with system administration. The goal is to set up a Debian server inside a virtual machine (VirtualBox) with no graphical interface, following a strict list of rules: encrypted LVM partitions, an SSH service on port 4242, a firewall, a strong password policy, a hardened `sudo` configuration and a small `monitoring.sh` script that broadcasts system information every 10 minutes.

I also did the partition part of the bonus (extra logical volumes).

The repository only contains this README and a [signature.txt](./signature.txt) with the SHA-1 of the virtual disk. The virtual machine itself is not included.

### 🔹 Design choices

- **Operating system:** I went with Debian. It is stable, beginner-friendly and uses `apt`, which I already knew from using Linux before. Rocky is more enterprise-oriented and its setup is more complex (SELinux, `dnf`, firewalld).
- **Partitioning:** everything lives inside an encrypted LVM setup. On top of the `/boot` partition, the volume group holds separate logical volumes for `/`, `/home`, `/var`, `/var/log`, `/srv`, `/tmp` and `[SWAP]`. Splitting things like this keeps a full `/var/log` or `/tmp` from taking the whole system down, and LVM makes resizing easy later.
- **Security policies:** password expiration and complexity rules are set through `login.defs` and `libpam-pwquality`. SSH runs on port 4242 with root login disabled. The only open port in UFW is 4242.
- **User management:** besides root, there is the `jcosta-a` user, which belongs to the `sudo` and `user42` groups. The hostname follows the subject's rule (login + 42, so `jcosta-a42`).
- **Sudo:** limited to 3 password attempts, custom error message, full input/output logging in `/var/log/sudo/`, TTY required and a restricted `secure_path`.
- **Services:** UFW as the firewall, AppArmor active at boot for Mandatory Access Control, SSH and cron (for the monitoring script).

### 🔹 Debian vs Rocky Linux

Debian is very stable, has a huge package repository, a simple `apt` workflow and lots of documentation, which makes it a good choice for a first server. The downside is that its packages tend to be older and it has no commercial backing like Red Hat. Rocky is a RHEL-compatible distribution aimed at enterprise environments, with long support cycles and SELinux enabled by default, but it is harder to configure and has a steeper learning curve for a beginner.

### 🔹 Comparisons

| Feature | My choice | Alternative | Notes |
| :---- | :---- | :---- | :---- |
| Operating system | Debian | Rocky Linux | Debian is simple, stable and uses `apt`. Rocky is a Red Hat derivative and uses `dnf`. |
| Security module | AppArmor | SELinux | AppArmor is path-based MAC (simpler). SELinux is label-based MAC (more powerful, more complex). |
| Firewall | UFW | firewalld | UFW is straightforward and beginner-friendly. `firewalld` works with zones and has more advanced rules. |
| Virtualization | VirtualBox | UTM | VirtualBox is a Type 2 hypervisor, widely used on x86. UTM is the option for Apple Silicon Macs. |

<p align="right"><a href="#-summary">⬆ back to summary</a></p>
</details>


<a name="instructions"></a>
<details open><summary><h2>⚙️ Instructions</h2></summary>

This assumes Debian is already installed with the encrypted LVM partitions (check them with `lsblk`) and no graphical environment. Log in as root and follow the steps below.

### 🔸 Step 1: Install `sudo`

```bash
apt install sudo
dpkg -l | grep sudo
reboot
```

### 🔸 Step 2: Users and groups

```bash
sudo addgroup user42
sudo adduser jcosta-a user42
sudo adduser jcosta-a sudo
```

To check the membership:

```bash
getent group sudo user42
```

### 🔸 Step 3: Hostname

The hostname must be the login followed by 42:

```bash
sudo hostnamectl set-hostname jcosta-a42
```

Don't forget to update the old name in `/etc/hosts` as well.

### 🔸 Step 4: SSH

```bash
sudo apt install openssh-server
sudo service ssh status
```

Edit the config file:

```bash
sudo vi /etc/ssh/sshd_config
```

Find `#Port 22` and change it to `Port 4242` (and remove the hash). Then set `PermitRootLogin` to `no`.

Restart the service:

```bash
sudo service ssh restart
```

To connect from the host machine, add a port forwarding rule in the VM network settings on VirtualBox (host port 4241 to guest port 4242) and then run:

```bash
ssh jcosta-a@localhost -p 4241
```

### 🔸 Step 5: UFW

I allow the SSH port before enabling the firewall, so I don't lock myself out:

```bash
sudo apt install ufw
sudo ufw allow 4242
sudo ufw enable
sudo ufw status
```

Only port 4242 should be open.

### 🔸 Step 6: Sudo policies

The `sudo` rules live in `/etc/sudoers.d/sudo_config`. The log directory has to exist first:

```bash
sudo mkdir -p /var/log/sudo
sudo visudo -f /etc/sudoers.d/sudo_config
```

What the project asks for:

- `passwd_tries`: number of password attempts
- `badpass_message`: message shown when the password is wrong
- `logfile`: where the sudo log is stored
- `log_input` / `log_output`: log what is typed and what is printed
- `iolog_dir`: directory for the input/output logs
- `requiretty`: TTY mode required
- `secure_path`: trusted PATH used by sudo

```
Defaults  passwd_tries=3
Defaults  badpass_message="Wrong password."
Defaults  logfile="/var/log/sudo/sudo_config"
Defaults  log_input
Defaults  log_output
Defaults  iolog_dir="/var/log/sudo"
Defaults  requiretty
Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

### 🔸 Step 7: Password policy

This one needs two files. The first is `/etc/login.defs`, where I set:

```
PASS_MAX_DAYS   30
PASS_MIN_DAYS   2
PASS_WARN_AGE   7
```

Check with:

```bash
grep ^PASS_ /etc/login.defs
```

These values only apply to new users, so existing accounts need `chage`:

```bash
sudo chage -m 2 -M 30 -W 7 jcosta-a
sudo chage -m 2 -M 30 -W 7 root
```

The second one is `libpam-pwquality`:

```bash
sudo apt install libpam-pwquality
```

The config is in `/etc/pam.d/common-password`. The options used are:

- `minlen`: minimum length
- `ucredit`, `lcredit`, `dcredit`: uppercase, lowercase and digit requirements (a negative value means "at least")
- `maxrepeat`: maximum number of consecutive identical characters
- `usercheck`: rejects passwords that contain the username
- `difok`: how many characters must differ from the previous password
- `enforce_for_root`: applies the rules to root as well

The subject says `difok` must not apply to root. To handle that I use `pam_succeed_if.so`: if the user is root (`uid=0`), it skips the next line, which is the one with `difok`.

```
password    [success=1 default=ignore]    pam_succeed_if.so uid=0
password    requisite    pam_pwquality.so retry=3 minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 maxrepeat=3 usercheck=1 difok=7
password    requisite    pam_pwquality.so retry=3 minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 maxrepeat=3 usercheck=1 enforce_for_root
```

After this, change the password of every account (root included) so they follow the new rules.

### 🔸 Step 8: `monitoring.sh`

The script is written in bash and uses `wall` to broadcast the information to every terminal:

```bash
#!/bin/bash

arch=$(uname -a)
cpu_phy=$(grep "physical id" /proc/cpuinfo | wc -l)
cpu_vrt=$(grep "processor" /proc/cpuinfo | wc -l)
ram_total=$(free --mega | awk '$1 == "Mem:" {print $2}')
ram_usage=$(free --mega | awk '$1 == "Mem:" {print $3}')
ram_prcnt=$(free --mega | awk '$1 == "Mem:" {printf("%.2f"), $3/$2*100}')
disk_total=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_t += $2} END {printf("%.1fGb"), disk_t/1024}')
disk_usage=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_u += $3} END {print disk_u}')
disk_prcnt=$(df -m | grep "/dev/" | grep -v "/boot" | awk '{disk_t += $2} {disk_u += $3} END {printf("%d"), disk_u/disk_t*100}')
cpu_load=$(top -bn1 | grep "Cpu(s)" | grep -oP "\d+\.\d+ id" | awk '{printf "%.1f", 100 - $1}')
last_boot=$(who -b | awk '$1 == "system" {print $3 " " $4}')
lvm_usage=$(if [ $(lsblk | grep "lvm" | wc -l) -gt 0 ]; then echo yes; else echo no; fi)
tcp_conn=$(ss -ta | grep ESTAB | wc -l)
user_log=$(users | wc -w)
ip_addr=$(hostname -I | awk '{print $1}')
mac_addr=$(ip link | grep "link/ether" | awk '{print $2}')
sudo_cmds=$(journalctl -q _COMM=sudo | grep COMMAND | wc -l)

wall "
#Architecture: $arch
#Physical CPU: $cpu_phy
#vCPU: $cpu_vrt
#Memory Usage: $ram_usage/${ram_total}MB ($ram_prcnt%)
#Disk Usage: $disk_usage/${disk_total} ($disk_prcnt%)
#CPU load: $cpu_load%
#Last boot: $last_boot
#LVM use: $lvm_usage
#TCP Connections: $tcp_conn ESTABLISHED
#User log: $user_log
#Network: IP $ip_addr ($mac_addr)
#Sudo: $sudo_cmds cmd
"
```

Then I scheduled it with cron, every 10 minutes:

```bash
sudo crontab -u root -e
```

```
*/10 * * * * /home/jcosta-a/monitoring.sh
```

To stop the script without touching it (needed in the evaluation), just comment out or remove that line in the crontab.

<p align="right"><a href="#-summary">⬆ back to summary</a></p>
</details>


<a name="evaluation"></a>
<details open><summary><h2>🧑‍💻 Evaluation</h2></summary>

### 🔹 Useful checks

Checks the OS and its version:

```bash
head -n 2 /etc/os-release
```

Checks that AppArmor is loaded and which profiles are active:

```bash
sudo aa-status
```

Lists the ports listening for connections (should show only SSH on 4242):

```bash
ss -tunlp
```

Checks that UFW is active and which ports are allowed:

```bash
sudo ufw status
```

### 🔹 Signature

Run this on the host machine, inside the VirtualBox VMs folder, and paste the result in `signature.txt`:

```bash
sha1sum b2br.vdi
```

The signature changes as soon as the VM is started again, so I always work on a clone (or a snapshot) and keep the original untouched for the evaluation.

<p align="right"><a href="#-summary">⬆ back to summary</a></p>
</details>


<a name="resources"></a>
<details open><summary><h2>🔗 Resources</h2></summary>

| Title | Author | Description |
| :---- | :---- | :---- |
| [O que é virtualização e máquinas virtuais?](https://www.youtube.com/watch?v=juh6DipSclg) | [Vlog de TI](https://www.youtube.com/@vlogdeti) | Intro to virtualization and virtual machines. |
| [Qual a diferença entre Máquina Virtual e Hypervisor?](https://www.youtube.com/watch?v=K756DRm2Vho) | [Bóson Treinamentos](https://www.youtube.com/@bosontreinamentos) | Helped me understand hypervisors and the Type 1 vs Type 2 difference. |
| [O que é o Kernel???](https://www.youtube.com/watch?v=HX12Jw5lppU) | [TITAN](https://www.youtube.com/@titandapoli) | Explains what the kernel is and what it does. |
| [O que é um Kernel - Conceitos Básicos de Computação](https://www.youtube.com/watch?v=GuBpmasRVus) | [Bóson Treinamentos](https://www.youtube.com/@bosontreinamentos) | Basic computing concepts about the kernel. |
| [O que é GNU/Linux?](https://www.debian.org/releases/trixie/s390x/ch01s02.pt.html) | Debian | Debian documentation page explaining what GNU/Linux is. |
| [GNU Operating System](https://www.gnu.org/) | GNU Project / Free Software Foundation | Helped me understand the difference between the Linux kernel and the GNU/Linux system. |
| [Entenda a estrutura do LVM em 5 minutos](https://www.youtube.com/watch?v=k5ZrQzwHW88) | [Mateus Muller](https://www.youtube.com/@MateusMuller) | Quick explanation of how LVM is structured (PV, VG and LV). |
| [Porta (redes de computadores)](https://pt.wikipedia.org/wiki/Porta_%28redes_de_computadores%29) | Wikipedia | What a network port is and how it is used by applications. |
| [What is a computer port?](https://www.cloudflare.com/pt-br/learning/network-layer/what-is-a-computer-port/) | Cloudflare | Another explanation of computer ports and how they work. |
| [Born2BeRoot Guide](https://noreply.gitbook.io/born2beroot) | Unknown | A step-by-step detailed guide through creating and configuring your first server using VirtualBox and Debian. |

### 🤖 How I used AI

I used an LLM (Gemini) only as a study aid, not to do the project for me. I used it to:

- understand the LVM concepts (Physical Volumes, Volume Groups and Logical Volumes);
- clarify the difference between AppArmor (path-based MAC) and SELinux (label-based MAC);
- understand "security by obscurity", why the SSH port was changed to 4242 and why root login is disabled;
- understand the difference between the Linux kernel and a GNU/Linux operating system;
- understand Type 1 (bare-metal) vs Type 2 (hosted) hypervisors;
- proofread and structure this README.

<p align="right"><a href="#-summary">⬆ back to summary</a></p>
</details>

---

<p align="left">
  Developed by <a href="https://github.com/jalvescodes">Jonathan Alves</a>
</p>