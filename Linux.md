# Linux — Complete Notes (Basics → Advanced + Interview Prep)

---

## Table of Contents
1. [Introduction](#1-introduction)
2. [Linux Architecture](#2-linux-architecture)
3. [File System Hierarchy](#3-file-system-hierarchy)
4. [File System Types & Concepts](#4-file-system-types--concepts)
5. [Basic Navigation & File Commands](#5-basic-navigation--file-commands)
6. [File Permissions & Ownership](#6-file-permissions--ownership)
7. [Users & Groups](#7-users--groups)
8. [Process Management](#8-process-management)
9. [Job Control & Signals](#9-job-control--signals)
10. [Systemd & Services](#10-systemd--services)
11. [Package Management](#11-package-management)
12. [Networking Commands](#12-networking-commands)
13. [Text Processing & Filters](#13-text-processing--filters)
14. [grep, sed, awk Deep Dive](#14-grep-sed-awk-deep-dive)
15. [Shell Scripting Basics](#15-shell-scripting-basics)
16. [Redirection & Pipes](#16-redirection--pipes)
17. [Environment Variables](#17-environment-variables)
18. [Disk & Storage Management](#18-disk--storage-management)
19. [Memory Management](#19-memory-management)
20. [Logging](#20-logging)
21. [Cron & Scheduled Tasks](#21-cron--scheduled-tasks)
22. [SSH & Remote Access](#22-ssh--remote-access)
23. [Archiving & Compression](#23-archiving--compression)
24. [Linux Security](#24-linux-security)
25. [Performance Monitoring & Troubleshooting](#25-performance-monitoring--troubleshooting)
26. [Kernel & Boot Process](#26-kernel--boot-process)
27. [Best Practices Summary](#27-best-practices-summary)
28. [Cheat Sheet](#28-cheat-sheet)
29. [Interview Questions & Answers](#29-interview-questions--answers)

---

## 1. Introduction

**Linux** is a free, open-source, Unix-like operating system kernel, first released by Linus Torvalds in 1991. "Linux" technically refers to the kernel; a complete OS built around it (kernel + GNU tools + package manager + etc.) is called a **distribution** (distro) — e.g., Ubuntu, Debian, CentOS/RHEL, Fedora, Amazon Linux, Alpine.

**Why Linux matters for DevOps/SRE/backend roles:**
- Runs the vast majority of servers, cloud infrastructure, and containers (a Docker container is fundamentally a Linux process using namespaces/cgroups)
- Powerful command-line tools for automation, scripting, and troubleshooting
- Open-source — fully inspectable, customizable, free to use
- Foundation of nearly every CI/CD pipeline, cloud VM, and Kubernetes node

**Key philosophy:** "Everything is a file" — devices, processes, sockets, and configuration are all represented and manipulated through the filesystem, enabling a small set of consistent tools to work across very different kinds of resources.

---

## 2. Linux Architecture

```
┌─────────────────────────────────────────┐
│              User Applications              │   (bash, vim, nginx, docker, your app)
├─────────────────────────────────────────┤
│              Shell / Libraries                │   (bash, glibc)
├─────────────────────────────────────────┤
│              System Call Interface             │   (open, read, write, fork, exec...)
├─────────────────────────────────────────┤
│              Kernel                             │
│  - Process Scheduler                              │
│  - Memory Management                                │
│  - File Systems (VFS)                                 │
│  - Network Stack                                        │
│  - Device Drivers                                          │
├─────────────────────────────────────────┤
│              Hardware                            │   (CPU, RAM, disk, NIC)
└─────────────────────────────────────────┘
```

**Kernel space vs User space:** The kernel runs in a privileged mode with direct hardware access ("kernel space"); regular applications run in a restricted "user space" and must go through **system calls** to request kernel services (reading a file, allocating memory, sending network packets) — this separation is fundamental to system stability and security.

**Monolithic kernel:** Linux is a monolithic kernel (as opposed to a microkernel) — most core OS services (filesystem, networking, drivers) run within the kernel itself for performance, though loadable kernel modules allow dynamic extension without a reboot.

---

## 3. File System Hierarchy

Linux follows the **Filesystem Hierarchy Standard (FHS)** — a single unified tree rooted at `/`, unlike Windows' drive-letter model.

```
/
├── bin/        → Essential user binaries (ls, cp, bash) — often symlinked to /usr/bin in modern distros
├── boot/       → Boot loader files, kernel images
├── dev/        → Device files (e.g., /dev/sda, /dev/null)
├── etc/        → System-wide configuration files
├── home/       → User home directories (/home/username)
├── lib/        → Shared libraries needed by binaries in /bin and /sbin
├── media/      → Mount points for removable media (USB drives, CDs)
├── mnt/        → Temporary mount points for manually mounted filesystems
├── opt/        → Optional/third-party software packages
├── proc/       → Virtual filesystem exposing kernel/process info (not real files on disk)
├── root/       → Home directory for the root user
├── run/        → Runtime data (PIDs, sockets) since last boot
├── sbin/       → System binaries, typically requiring root (e.g., reboot, fdisk)
├── srv/        → Data for services provided by the system (e.g., web server content)
├── sys/        → Virtual filesystem exposing kernel/device/driver info
├── tmp/        → Temporary files, often cleared on reboot
├── usr/        → User programs, libraries, documentation (the largest directory tree typically)
└── var/        → Variable data: logs, caches, spool files, databases
```

**Key directories to know cold for interviews:**
- `/etc` — configuration files (`/etc/passwd`, `/etc/hosts`, `/etc/fstab`, `/etc/ssh/sshd_config`)
- `/var/log` — system and application logs
- `/proc` — a virtual ("pseudo") filesystem; reading `/proc/cpuinfo`, `/proc/meminfo`, or `/proc/<pid>/status` gives live kernel/process info without any real file existing on disk
- `/dev` — device files; e.g., `/dev/null` (discards all input), `/dev/sda` (a block device representing a disk)

---

## 4. File System Types & Concepts

| Concept | Description |
|---|---|
| **inode** | A data structure storing metadata about a file (permissions, owner, size, timestamps, pointers to data blocks) — but **not** the filename itself |
| **Hard link** | A second directory entry pointing to the **same inode** — indistinguishable from the "original," same data, deleting one doesn't affect the other while any link remains |
| **Symbolic (soft) link** | A separate file containing a *path* to another file — breaks if the target is moved/deleted, can cross filesystems, can link to directories |
| **Mount point** | A directory where another filesystem/device is attached into the unified directory tree |
| **Common filesystem types** | `ext4` (most common Linux default), `xfs` (common in RHEL/CentOS, good for large files), `btrfs` (snapshots, checksums), `tmpfs` (RAM-backed, e.g., `/tmp` on some systems) |

```bash
ls -i file.txt                  # show inode number
ln file.txt hardlink.txt        # create a hard link
ln -s file.txt symlink.txt      # create a symbolic link
stat file.txt                   # detailed metadata including inode
df -h                            # show mounted filesystems and disk usage
mount /dev/sdb1 /mnt/data        # mount a device to a directory
umount /mnt/data                 # unmount
```

**Hard link vs Symbolic link (very common interview question):** A hard link is a second name for the **exact same inode/data** — the file isn't actually "deleted" until all hard links to it are removed, and hard links can't span filesystems or point to directories. A symbolic link is a small separate file that just stores a *path string* pointing elsewhere — it can cross filesystems and link to directories, but becomes a "broken link" if the target is removed or moved.

---

## 5. Basic Navigation & File Commands

```bash
pwd                          # print working directory
ls -la                       # list all files (incl. hidden), long format
cd /path/to/dir              # change directory
cd ~                         # go to home directory
cd -                         # go to previous directory

mkdir mydir                  # create directory
mkdir -p a/b/c                # create nested directories
rmdir mydir                   # remove empty directory
rm -rf mydir                  # remove directory and contents recursively (DANGEROUS — no undo)

touch file.txt                # create empty file / update timestamp
cp file.txt dest/             # copy file
cp -r dir/ dest/              # copy directory recursively
mv file.txt newname.txt        # move/rename
rm file.txt                    # delete file

cat file.txt                   # print full file content
less file.txt                  # paginated viewer (q to quit, / to search)
head -n 20 file.txt             # first 20 lines
tail -n 20 file.txt              # last 20 lines
tail -f /var/log/syslog           # follow file in real-time (very common for logs)

find / -name "*.log"               # find files by name
find / -type f -mtime -1             # files modified in the last day
find / -size +100M                    # files larger than 100MB
which python3                          # show path of an executable
locate file.txt                         # fast file search (uses a prebuilt index, updatedb)
```

---

## 6. File Permissions & Ownership

```bash
ls -l file.txt
# -rwxr-xr--  1 user group  4096 Jun 19 10:00 file.txt
```

**Breaking down `-rwxr-xr--`:**
```
-       rwx        r-x        r--
type   owner      group      others
```
- First char: `-` (regular file), `d` (directory), `l` (symlink)
- Each subsequent triplet: `r` (read), `w` (write), `x` (execute) for **owner**, **group**, **others**

**Numeric (octal) permission notation:**
```
r = 4, w = 2, x = 1   → sum per triplet
rwx = 7, r-x = 5, r-- = 4, rw- = 6, --- = 0

chmod 755 file.sh    # rwxr-xr-x  (owner: rwx, group: r-x, others: r-x)
chmod 644 file.txt   # rw-r--r--  (owner: rw-, group: r--, others: r--)
chmod u+x script.sh  # add execute permission for owner only
chmod -R 644 dir/    # recursively set permissions
```

```bash
chown user:group file.txt        # change owner and group
chown -R user:group dir/          # recursively
chgrp group file.txt               # change group only
```

**Special permissions:**
| Bit | Symbol | Effect |
|---|---|---|
| **SUID** | `s` in owner's execute slot (`chmod u+s`) | Executable runs with the **file owner's** privileges, not the invoking user's (e.g., `/usr/bin/passwd` runs as root so any user can change their own password) |
| **SGID** | `s` in group's execute slot (`chmod g+s`) | On a file: runs with the **group's** privileges; on a directory: new files inherit the directory's group |
| **Sticky bit** | `t` in others' execute slot (`chmod +t`) | On a directory: only the file's owner (or root) can delete/rename files within it, even if others have write permission to the directory (classic example: `/tmp`) |

```bash
chmod u+s /usr/bin/someapp    # set SUID
chmod 1777 /tmp                 # sticky bit (1) + rwxrwxrwx — exactly how /tmp is typically configured
```

**For directories, the execute (`x`) permission has a special meaning:** it controls whether you can `cd` into the directory or access files *within* it by name (not list its contents — that's `r`). A directory with `r--` but no `x` lets you list filenames but not access/stat the files inside.

---

## 7. Users & Groups

```bash
whoami                        # current user
id                              # current user's UID, GID, group memberships
cat /etc/passwd                  # all users (username:x:UID:GID:comment:home:shell)
cat /etc/shadow                   # encrypted passwords (root-only readable)
cat /etc/group                     # all groups

sudo useradd -m -s /bin/bash newuser    # create user with home dir + shell
sudo passwd newuser                       # set password
sudo userdel -r newuser                     # delete user + home directory
sudo usermod -aG sudo newuser                 # add user to the 'sudo' group (append, don't remove existing groups!)

sudo groupadd developers
sudo groupdel developers

su - username                  # switch user (full login shell)
sudo command                    # run a single command as root (or another user with -u)
sudo -i                          # interactive root shell
```

**`su` vs `sudo` (interview point):** `su` switches your entire session to another user (typically requiring that user's password). `sudo` runs a single command with elevated privileges (typically requiring *your own* password, governed by `/etc/sudoers` rules) without switching your full session — generally preferred for auditability (logs exactly which command was run, by whom) and following least-privilege practice.

**Root user (UID 0):** Has unrestricted access to the entire system, bypassing normal permission checks — best practice is to avoid logging in directly as root; use `sudo` from a regular account instead, both for audit trails and to reduce the blast radius of mistakes/compromised credentials.

---

## 8. Process Management

```bash
ps aux                          # snapshot of all running processes
ps -ef                           # alternative format (full-format listing)
top                                # real-time interactive process viewer
htop                                # nicer, more interactive alternative to top (often needs separate install)

ps aux | grep nginx                 # find a specific process
pgrep nginx                          # get PID(s) of a process by name
pidof nginx                           # similar

kill <PID>                       # send SIGTERM (graceful termination request)
kill -9 <PID>                     # send SIGKILL (immediate, forceful termination)
killall nginx                      # kill all processes matching a name
pkill -f "pattern"                  # kill processes matching a command-line pattern

nice -n 10 command                # start a process with lower scheduling priority
renice -n 5 -p <PID>                # change priority of a running process

nohup command &                  # run a command immune to hangups, in background
command &                         # run in background
jobs                                # list background jobs in current shell
fg %1                                # bring job 1 to foreground
bg %1                                 # resume job 1 in background
```

**Process states:** `R` (running/runnable), `S` (sleeping, waiting for an event), `D` (uninterruptible sleep, usually I/O), `Z` (zombie — finished but not yet reaped by its parent), `T` (stopped/traced).

**Zombie process (common interview question):** A process that has finished execution but still has an entry in the process table because its parent hasn't yet called `wait()` to read its exit status. A handful of transient zombies is normal; a large/growing number indicates the parent process isn't properly reaping its children — a resource leak (though zombies themselves consume minimal resources, just a process table slot).

---

## 9. Job Control & Signals

| Signal | Number | Meaning |
|---|---|---|
| `SIGHUP` | 1 | Hang up — often used to tell a daemon to reload its config |
| `SIGINT` | 2 | Interrupt — sent by Ctrl+C |
| `SIGKILL` | 9 | Force kill — **cannot be caught, blocked, or ignored** by the process |
| `SIGTERM` | 15 | Terminate gracefully — **default** signal sent by `kill`; can be caught for cleanup |
| `SIGSTOP` | 19 | Pause process — cannot be caught/ignored |
| `SIGCONT` | 18 | Resume a stopped process |

```bash
kill -SIGTERM <PID>     # same as: kill <PID>
kill -SIGKILL <PID>     # same as: kill -9 <PID>
trap 'echo cleanup' SIGTERM    # in a script: catch a signal and run cleanup code
```

**`SIGTERM` vs `SIGKILL` (very common interview question):** `SIGTERM` is a *request* to terminate gracefully — the process can intercept it to flush buffers, close connections, save state, and exit cleanly. `SIGKILL` is enforced directly by the kernel and **cannot be caught or ignored** by the process at all — it's an immediate, forceful termination with no chance for cleanup, used as a last resort when a process is unresponsive to `SIGTERM`. (This directly parallels `docker stop` vs `docker kill`.)

---

## 10. Systemd & Services

**systemd** is the modern init system and service manager used by most major distros (replacing older SysV init) — manages the boot process, services (daemons), mount points, and more, using declarative "unit" files.

```bash
systemctl status nginx          # check service status
systemctl start nginx            # start a service
systemctl stop nginx              # stop a service
systemctl restart nginx            # restart
systemctl reload nginx              # reload config without full restart (if the service supports it)
systemctl enable nginx               # start automatically on boot
systemctl disable nginx               # don't start on boot
systemctl is-enabled nginx              # check if enabled
systemctl list-units --type=service      # list all service units

journalctl -u nginx                # view logs for a specific systemd-managed service
journalctl -f                       # follow logs in real-time (like tail -f)
journalctl --since "1 hour ago"      # filter by time
journalctl -p err                     # filter by priority (errors only)
```

**Basic unit file example** (`/etc/systemd/system/myapp.service`):
```ini
[Unit]
Description=My Application
After=network.target

[Service]
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=on-failure
User=appuser
WorkingDirectory=/opt/myapp

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload     # reload unit files after editing
sudo systemctl enable --now myapp.service   # enable + start in one command
```

**`systemctl reload` vs `restart` (interview point):** `restart` fully stops and starts the process (brief downtime, loses in-memory state). `reload` signals the running process to re-read its configuration without stopping it (e.g., nginx's reload gracefully finishes in-flight requests on old workers while starting new workers with the new config) — only works if the service explicitly supports it.

---

## 11. Package Management

| Distro Family | Package Manager | Package Format |
|---|---|---|
| Debian/Ubuntu | `apt` / `apt-get` (front-end for `dpkg`) | `.deb` |
| RHEL/CentOS/Fedora | `yum` (older) / `dnf` (modern) (front-end for `rpm`) | `.rpm` |
| Alpine | `apk` | `.apk` |
| Universal/cross-distro | `snap`, `flatpak` | sandboxed |

```bash
# Debian/Ubuntu
sudo apt update                  # refresh package index
sudo apt upgrade                  # upgrade installed packages
sudo apt install nginx              # install a package
sudo apt remove nginx                # remove (keep config files)
sudo apt purge nginx                  # remove including config files
apt list --installed                   # list installed packages
apt search nginx                        # search available packages
dpkg -l | grep nginx                     # list via dpkg directly

# RHEL/CentOS/Fedora
sudo dnf install nginx
sudo dnf update
sudo dnf remove nginx
rpm -qa | grep nginx
```

**`apt update` vs `apt upgrade` (commonly confused, worth knowing):** `update` refreshes the local index of *available* package versions from configured repositories (doesn't install/change anything on the system). `upgrade` actually installs newer versions of already-installed packages based on that refreshed index.

---

## 12. Networking Commands

```bash
ip addr show                  # show network interfaces and IP addresses (modern, replaces ifconfig)
ifconfig                       # legacy equivalent (often not installed by default anymore)
ip route show                   # show routing table
ping example.com                 # test reachability
traceroute example.com            # trace the network path/hops
dig example.com                    # DNS lookup, detailed
nslookup example.com                # DNS lookup, simpler
host example.com                     # quick DNS lookup

netstat -tulnp                  # show listening ports and associated processes (legacy)
ss -tulnp                        # modern replacement for netstat — faster, more detail

curl -v https://example.com         # make an HTTP request, verbose output
curl -I https://example.com          # headers only (HEAD-like)
wget https://example.com/file.zip     # download a file

scp file.txt user@host:/path/         # copy file over SSH
rsync -avz src/ user@host:/dest/        # efficient sync (only transfers changes), preserves attributes

systemctl status NetworkManager           # check network service (varies by distro)
iptables -L                                  # list firewall rules (legacy)
nft list ruleset                              # nftables, modern replacement for iptables on many distros
```

**`netstat` vs `ss` (interview point):** `ss` is the modern replacement for `netstat` — significantly faster because it reads directly from kernel data structures rather than parsing `/proc`, and is the recommended tool on current systems (netstat is often deprecated/not installed by default anymore).

---

## 13. Text Processing & Filters

```bash
wc -l file.txt              # count lines
wc -w file.txt               # count words
sort file.txt                 # sort lines alphabetically
sort -n file.txt                # numeric sort
sort -r file.txt                 # reverse sort
uniq file.txt                     # remove adjacent duplicate lines (often used after sort)
uniq -c file.txt                   # count occurrences of each line
cut -d',' -f2 file.csv               # extract the 2nd field, comma-delimited
tr 'a-z' 'A-Z' < file.txt              # translate/transform characters
diff file1.txt file2.txt                # show differences between files
paste file1.txt file2.txt                 # merge lines from files side by side
xargs                                      # build/execute commands from stdin input

# Common combos
cat access.log | grep "404" | wc -l                  # count 404 errors
ps aux | sort -k3 -nr | head -10                       # top 10 processes by CPU usage
find . -name "*.tmp" | xargs rm                          # delete all found files
```

---

## 14. grep, sed, awk Deep Dive

These three are the backbone of Linux text processing — extremely common in both real work and interviews.

### grep — search text
```bash
grep "error" file.log                 # lines containing "error"
grep -i "error" file.log                # case-insensitive
grep -v "debug" file.log                 # invert match (lines NOT containing "debug")
grep -r "TODO" /path/to/dir/               # recursive search through directory
grep -n "error" file.log                    # show line numbers
grep -c "error" file.log                     # count matching lines
grep -E "error|warning" file.log               # extended regex (OR pattern)
grep -B 2 -A 3 "error" file.log                  # show 2 lines before, 3 after each match (context)
```

### sed — stream editor (find/replace, text transformation)
```bash
sed 's/foo/bar/' file.txt              # replace first occurrence of foo with bar, per line
sed 's/foo/bar/g' file.txt               # replace ALL occurrences (global)
sed -i 's/foo/bar/g' file.txt              # edit the file in-place
sed -n '5,10p' file.txt                      # print only lines 5-10
sed '/pattern/d' file.txt                      # delete lines matching pattern
sed -i.bak 's/old/new/g' file.txt                # in-place edit, keep a .bak backup
```

### awk — pattern scanning and field-based processing
```bash
awk '{print $1}' file.txt                   # print first whitespace-delimited field/column
awk -F',' '{print $2}' file.csv                # print 2nd field, comma-delimited
awk '{print $1, $3}' file.txt                    # print specific columns
awk '$3 > 100 {print $0}' file.txt                 # print lines where field 3 > 100
awk '{sum += $2} END {print sum}' file.txt           # sum a column
awk 'NR==5' file.txt                                   # print only the 5th line
awk '{print NR, $0}' file.txt                            # prefix each line with its line number
```

**grep vs sed vs awk (interview summary):** `grep` finds/filters lines matching a pattern. `sed` transforms text (substitution, deletion) on a stream, line by line. `awk` is a full pattern-scanning/field-processing language — best when you need to work with structured columnar data (sums, conditional logic per field, reformatting output).

---

## 15. Shell Scripting Basics

```bash
#!/bin/bash
# A basic shell script

NAME="World"
echo "Hello, $NAME!"

# Variables (no spaces around =)
COUNT=5

# Conditionals
if [ "$COUNT" -gt 3 ]; then
    echo "Count is greater than 3"
elif [ "$COUNT" -eq 3 ]; then
    echo "Count is exactly 3"
else
    echo "Count is 3 or less"
fi

# Loops
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

for file in /var/log/*.log; do
    echo "Found log: $file"
done

i=0
while [ $i -lt 5 ]; do
    echo "i is $i"
    i=$((i + 1))
done

# Functions
greet() {
    local name=$1
    echo "Hello, $name!"
}
greet "Alice"

# Reading user input
read -p "Enter your name: " username
echo "Hi, $username"

# Command substitution
current_date=$(date +%Y-%m-%d)
echo "Today is $current_date"

# Exit codes
if [ $? -eq 0 ]; then
    echo "Previous command succeeded"
fi

# Script arguments
echo "Script name: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Number of args: $#"
```

```bash
chmod +x script.sh    # make executable
./script.sh             # run it
bash script.sh            # or run explicitly with bash
```

**Common test conditions:**
```bash
[ -f file.txt ]    # file exists and is a regular file
[ -d dir ]          # directory exists
[ -z "$VAR" ]        # string is empty
[ -n "$VAR" ]         # string is non-empty
[ "$A" = "$B" ]        # string equality
[ "$A" -eq "$B" ]       # numeric equality
```

---

## 16. Redirection & Pipes

```bash
command > file.txt         # redirect stdout, overwrite file
command >> file.txt          # redirect stdout, append to file
command 2> error.log           # redirect stderr only
command > out.log 2>&1           # redirect both stdout and stderr to same file
command &> all.log                # shorthand for the same thing (bash-specific)
command < input.txt                # use file as stdin

command1 | command2          # pipe — stdout of command1 becomes stdin of command2
command1 | command2 | command3   # chain multiple pipes

command > /dev/null 2>&1     # discard all output entirely
```

**File descriptors:** `0` = stdin, `1` = stdout, `2` = stderr — this is why `2>&1` means "redirect file descriptor 2 (stderr) to wherever file descriptor 1 (stdout) currently points."

---

## 17. Environment Variables

```bash
echo $HOME                    # print a variable
export MY_VAR="value"           # set + export (visible to child processes)
unset MY_VAR                     # remove a variable
env                                # list all environment variables
printenv PATH                       # print a specific one

# Common ones
$PATH       # directories searched for executables
$HOME       # current user's home directory
$USER       # current username
$PWD        # current working directory
$SHELL      # current default shell

# Persisting across sessions
~/.bashrc       # per-user, loaded for interactive non-login shells
~/.bash_profile  # per-user, loaded for login shells
/etc/environment  # system-wide
/etc/profile       # system-wide, loaded for login shells
```

**`export` vs a plain variable assignment (interview point):** A plain `VAR=value` is only visible within the current shell. `export VAR=value` additionally makes it part of the environment passed to any **child processes** spawned from that shell — this is why scripts/programs you run need variables exported to actually see them.

---

## 18. Disk & Storage Management

```bash
df -h                          # disk space usage per mounted filesystem, human-readable
du -sh /var/log                  # total size of a directory
du -sh /var/log/*                  # size of each item within a directory
du -h --max-depth=1 /                # size of each top-level directory

lsblk                            # list block devices (disks, partitions)
fdisk -l                          # list disk partitions (legacy, MBR-focused)
parted -l                          # list partitions (modern, supports GPT)

mkfs.ext4 /dev/sdb1                # format a partition with ext4
mount /dev/sdb1 /mnt/data             # mount manually
umount /mnt/data                       # unmount

cat /etc/fstab                          # filesystem mount config, applied automatically at boot

# LVM (Logical Volume Manager) — flexible disk management
pvcreate /dev/sdb               # create physical volume
vgcreate myvg /dev/sdb            # create volume group
lvcreate -L 10G -n mylv myvg        # create logical volume
lvextend -L +5G /dev/myvg/mylv        # grow a logical volume (then resize the filesystem too)
resize2fs /dev/myvg/mylv                # resize ext4 filesystem after extending the LV
```

**`df` vs `du` (commonly confused, worth knowing):** `df` reports disk space usage at the **filesystem/mount point** level (how full is this disk). `du` reports the disk usage of specific **files/directories** (how much space does this folder actually consume) — they can sometimes disagree slightly (e.g., due to deleted-but-still-open files still consuming space that `df` accounts for, but `du` won't show since the file is no longer in the directory tree).

---

## 19. Memory Management

```bash
free -h                  # show memory usage (total, used, free, cache/buffer), human-readable
cat /proc/meminfo           # detailed kernel memory stats
vmstat 1                     # virtual memory stats, refreshed every 1 second
```

**Reading `free -h` output correctly (common interview gotcha):** The "free" column alone is misleading — Linux aggressively uses otherwise-idle RAM for disk **cache/buffers** to speed up I/O, and will reclaim that cache instantly if an application needs the memory. The more meaningful number is the **"available"** column, which accounts for reclaimable cache — a system showing low "free" but high "available" memory is generally healthy, not low on memory.

**Swap:** Disk-backed virtual memory used as overflow when physical RAM is exhausted — much slower than RAM, heavy swap usage ("thrashing") is usually a sign of memory pressure/undersized instances.
```bash
swapon --show           # show active swap
free -h                   # also shows swap usage
```

---

## 20. Logging

```bash
/var/log/syslog          # general system log (Debian/Ubuntu)
/var/log/messages          # general system log (RHEL/CentOS)
/var/log/auth.log           # authentication/sudo logs (Debian/Ubuntu)
/var/log/secure              # authentication logs (RHEL/CentOS)
/var/log/kern.log              # kernel messages
/var/log/dmesg                  # boot-time kernel ring buffer messages

dmesg                       # print kernel ring buffer (boot/hardware messages)
dmesg -T                      # with human-readable timestamps
dmesg | grep -i error           # search for errors in kernel log

journalctl                   # systemd's centralized log viewer (covers most modern logging)
journalctl -k                  # kernel messages only
journalctl --disk-usage          # how much disk space journal logs are consuming
```

**Traditional flat-file logs (`/var/log/*`) vs `journalctl` (systemd journal) (interview point):** Modern systemd-based distros centralize most logging (including kernel, service, and boot logs) into a structured **binary journal**, queryable via `journalctl` with powerful filtering (by service, time range, priority). Some applications/distros still also write traditional flat-text logs to `/var/log/` for compatibility/specific tools.

---

## 21. Cron & Scheduled Tasks

```bash
crontab -e          # edit current user's crontab
crontab -l            # list current user's cron jobs
crontab -r             # remove all cron jobs for current user
```

**Cron syntax:**
```
* * * * * command_to_run
│ │ │ │ │
│ │ │ │ └── day of week (0-6, Sunday=0)
│ │ │ └──── month (1-12)
│ │ └────── day of month (1-31)
│ └──────── hour (0-23)
└────────── minute (0-59)
```

```bash
0 2 * * *      command     # every day at 2:00 AM
*/15 * * * *    command     # every 15 minutes
0 0 * * 0       command     # every Sunday at midnight
0 9-17 * * 1-5   command     # every hour from 9 AM-5 PM, Monday-Friday
```

**System-wide cron locations:** `/etc/crontab`, `/etc/cron.d/`, `/etc/cron.daily/` (and `.hourly`/`.weekly`/`.monthly`) for scripts run on a schedule without needing a crontab entry.

**`at`** — schedule a one-time future task (vs cron's recurring schedule):
```bash
echo "command" | at 10:00 PM
atq        # list pending at jobs
```

---

## 22. SSH & Remote Access

```bash
ssh user@hostname                     # connect to a remote server
ssh -p 2222 user@hostname                # connect on a non-default port
ssh -i ~/.ssh/mykey.pem user@hostname       # connect using a specific private key

ssh-keygen -t ed25519 -C "comment"            # generate a new key pair (modern, recommended algorithm)
ssh-copy-id user@hostname                       # copy your public key to a remote server's authorized_keys

cat ~/.ssh/id_ed25519.pub                          # view your public key
~/.ssh/authorized_keys                                # public keys allowed to log in as this user
~/.ssh/config                                          # client-side host aliases/shortcuts

scp file.txt user@host:/remote/path/                # copy a file to a remote host
scp user@host:/remote/file.txt ./                      # copy a file from a remote host
rsync -avz -e ssh src/ user@host:/dest/                  # efficient sync over SSH
```

**SSH key authentication flow (interview point):** The client holds a **private key** (never shared); the corresponding **public key** is placed in `~/.ssh/authorized_keys` on the server. During connection, the server challenges the client to prove possession of the private key (via a cryptographic challenge) without the private key ever being transmitted — far more secure than password authentication, and the standard practice for production server access (often with password auth disabled entirely in `sshd_config`).

```ini
# /etc/ssh/sshd_config (server-side hardening, common settings)
PasswordAuthentication no
PermitRootLogin no
Port 2222
```

---

## 23. Archiving & Compression

```bash
tar -cvf archive.tar dir/            # create a tar archive (c=create, v=verbose, f=file)
tar -xvf archive.tar                   # extract a tar archive
tar -czvf archive.tar.gz dir/            # create + gzip-compress (z=gzip)
tar -xzvf archive.tar.gz                   # extract a gzip-compressed tar archive
tar -tzvf archive.tar.gz                     # list contents without extracting

gzip file.txt                # compress (produces file.txt.gz, removes original)
gunzip file.txt.gz             # decompress
zip archive.zip file1 file2      # create a zip archive
unzip archive.zip                  # extract a zip archive
```

**`tar` mnemonic:** Think of common combinations as words — `tar -czvf` ("create, zip, verbose, file") to compress, `tar -xzvf` ("extract, zip, verbose, file") to decompress.

---

## 24. Linux Security

1. **Disable direct root SSH login**, use `sudo` from individual accounts for auditability.
2. **Use SSH key-based authentication**, disable password authentication where possible.
3. **Keep the system patched** — `apt upgrade`/`dnf update` regularly, especially for security advisories.
4. **Use a firewall** — `ufw` (Ubuntu's simplified frontend), `firewalld` (RHEL-family), or raw `iptables`/`nftables` — to restrict open ports to only what's needed.
5. **Principle of least privilege** — don't run services as root unless absolutely necessary; use dedicated service accounts.
6. **File permission hygiene** — avoid `chmod 777`; understand exactly what permissions a file/script actually needs.
7. **SELinux / AppArmor** — Mandatory Access Control (MAC) systems providing an additional security layer beyond standard discretionary Unix permissions, confining what even a compromised/misbehaving process (even running as root) can actually do.
8. **Audit logs regularly** — `/var/log/auth.log` (or `/var/log/secure`) for login attempts, `journalctl` for service-level anomalies.
9. **Use `fail2ban`** or similar tools to automatically block IPs after repeated failed login attempts.
10. **Encrypt sensitive data at rest** (e.g., LUKS for full-disk encryption) where required.

```bash
ufw allow 22/tcp           # allow SSH
ufw allow 443/tcp            # allow HTTPS
ufw enable                     # enable the firewall
ufw status verbose               # check rules

sestatus                  # check SELinux status (RHEL-family)
getenforce                  # quick SELinux mode check (Enforcing/Permissive/Disabled)
aa-status                     # check AppArmor status (Debian/Ubuntu)
```

**SELinux/AppArmor vs standard file permissions (interview point):** Standard Unix permissions are **discretionary** — the file owner decides who can access it, and a process running as root bypasses these checks entirely. SELinux/AppArmor add **mandatory access control** — security policies defined by an administrator that constrain what a process can do *regardless* of standard file permissions or even root privilege, significantly limiting the damage a compromised process or service can cause.

---

## 25. Performance Monitoring & Troubleshooting

```bash
top / htop              # real-time CPU/memory/process overview
uptime                    # load average + how long the system has been up
vmstat 1                   # CPU, memory, I/O, system stats over time
iostat -x 1                  # detailed disk I/O statistics
sar                            # historical system activity (requires sysstat package)
lsof                              # list open files (and which process holds them)
lsof -i :8080                       # find what process is using port 8080
lsof -p <PID>                         # list files opened by a specific process

ps aux --sort=-%cpu | head           # top processes by CPU usage
ps aux --sort=-%mem | head             # top processes by memory usage

strace -p <PID>           # trace system calls made by a running process (deep debugging)
strace command               # trace system calls of a new command
ltrace command                  # trace library calls (less commonly needed)
```

**Reading "load average" (`uptime` output, common interview question):** The three numbers represent the average number of processes wanting CPU time (running + waiting) over the last 1, 5, and 15 minutes. A load average roughly equal to or below your CPU core count generally indicates the system is keeping up; significantly higher indicates contention/queuing. (e.g., a load average of 8 on a 4-core machine suggests the CPU is a bottleneck.)

**Troubleshooting a "server is slow" scenario (common scenario-based interview question):** Check `top`/`htop` first for an obvious CPU or memory hog; check `free -h` for actual memory pressure (vs misleadingly low "free" but healthy "available"); check `iostat`/`df -h` for disk I/O bottlenecks or a full disk; check `uptime`'s load average against core count; check application/system logs (`journalctl`, `/var/log/`) for errors around the same timeframe; use `lsof`/`netstat`/`ss` if it seems network/connection-related.

---

## 26. Kernel & Boot Process

**Boot sequence (high level):**
```
BIOS/UEFI → Bootloader (GRUB) → Kernel loads → initramfs (temporary root FS) →
systemd (PID 1) starts → targets/services start in dependency order → login prompt
```

- **BIOS/UEFI** — firmware that initializes hardware and hands off to the bootloader
- **GRUB** — the most common Linux bootloader; lets you select which kernel/OS to boot
- **Kernel** — loads into memory, initializes core subsystems, mounts the real root filesystem
- **systemd (PID 1)** — the first user-space process; starts all other services/processes according to declared dependencies and **targets** (systemd's modern equivalent of old SysV "runlevels," e.g., `multi-user.target` ≈ old runlevel 3, `graphical.target` ≈ old runlevel 5)

```bash
uname -a                 # kernel version + system info
uname -r                   # kernel version only
lsmod                        # list loaded kernel modules
modprobe module_name           # load a kernel module
systemctl get-default            # show the default boot target
systemctl set-default multi-user.target   # change default boot target
```

---

## 27. Best Practices Summary

- Prefer `sudo` over direct root login for auditability and least privilege
- Use SSH keys, not passwords, for remote access
- Automate repetitive tasks with shell scripts and cron (or better, proper job schedulers for complex pipelines)
- Use `journalctl`/centralized logging rather than hunting across scattered flat files where possible
- Monitor `df -h` and set up alerting for disk space — a full disk causes cascading failures
- Use version-controlled, declarative configuration (Ansible, Terraform, Configuration as Code) for production servers rather than untracked manual changes
- Understand the difference between `SIGTERM` and `SIGKILL`, and design your own services to handle `SIGTERM` gracefully
- Keep systems patched; subscribe to security advisories for your distro
- Use a firewall and disable unnecessary open ports/services
- Regularly audit file permissions, especially for anything world-writable or SUID

---

## 28. Cheat Sheet

```bash
# Navigation
pwd / cd / ls -la / find / locate

# Files
cp / mv / rm / mkdir / touch / cat / less / head / tail -f

# Permissions
chmod 755 file / chown user:group file / chmod u+x file

# Processes
ps aux / top / htop / kill -9 PID / systemctl status/start/stop/restart

# Networking
ip addr / ss -tulnp / curl -v / ping / dig

# Text processing
grep -rn "pattern" / sed -i 's/a/b/g' file / awk '{print $1}' file

# Disk
df -h / du -sh dir/ / lsblk

# Logs
journalctl -u service -f / tail -f /var/log/syslog

# Archives
tar -czvf out.tar.gz dir/ / tar -xzvf out.tar.gz

# SSH
ssh user@host / ssh-keygen -t ed25519 / scp file user@host:/path

# Cron
crontab -e   →   */15 * * * * /path/to/script.sh
```

---

## 29. Interview Questions & Answers

**Q1: What's the difference between a hard link and a symbolic link?**
A: A hard link is an additional directory entry pointing to the exact same inode/data as the original file — the underlying data isn't deleted until every hard link to it is removed, and hard links can't cross filesystems or point to directories. A symbolic link is a small separate file that stores a path string pointing to another file — it can cross filesystems and link to directories, but becomes a broken/"dangling" link if the target is moved or deleted.

**Q2: Explain Linux file permissions (rwx) and how `chmod 755` works.**
A: Each file has three permission triplets — owner, group, and others — each controlling read (r=4), write (w=2), and execute (x=1) access. `chmod 755` sets owner to `rwx` (7 = 4+2+1), and group/others to `r-x` (5 = 4+1) — a very common setting for executable scripts that the owner can fully manage but others can only read and execute.

**Q3: What's the difference between `SIGTERM` and `SIGKILL`?**
A: `SIGTERM` (signal 15, the default for `kill`) is a polite request to terminate — the receiving process can catch it and perform cleanup (closing connections, flushing data) before exiting. `SIGKILL` (signal 9) is enforced directly by the kernel and cannot be caught, blocked, or ignored by the process at all — it terminates immediately with no opportunity for graceful cleanup, used as a last resort for unresponsive processes.

**Q4: What is a zombie process?**
A: A process that has finished executing but still has an entry in the process table because its parent process hasn't yet called `wait()` to read its exit status and release that entry. A small number is generally harmless and transient; a large or growing number indicates the parent isn't properly reaping its children, which is worth investigating even though zombies themselves consume minimal resources.

**Q5: What's the difference between `su` and `sudo`?**
A: `su` switches your entire shell session to another user (typically the target user's own password is required), giving you a full session as that user. `sudo` executes a single command with elevated privileges (typically using your own password, governed by `/etc/sudoers`) without switching your overall session — generally preferred in production for better auditability (clear logs of exactly who ran what) and adherence to least-privilege practice.

**Q6: How would you find out what process is using a specific port?**
A: `lsof -i :8080` or `ss -tulnp | grep 8080` — both show which process (with its PID) currently has that port open/listening, useful for diagnosing "port already in use" errors or identifying unexpected services.

**Q7: What's the difference between `/etc/passwd` and `/etc/shadow`?**
A: `/etc/passwd` stores basic user account information (username, UID, GID, home directory, default shell) and is world-readable. `/etc/shadow` stores the actual encrypted/hashed password data (plus password aging policy) and is restricted to root-only access — separated specifically so that ordinary users/processes that need to read basic account info (e.g., to resolve a UID to a username) don't also gain access to password hashes.

**Q8: Explain the difference between `grep`, `sed`, and `awk`.**
A: `grep` searches/filters text for lines matching a pattern. `sed` is a stream editor for transforming text — typically find-and-replace or deleting matching lines. `awk` is a more complete pattern-scanning and field-processing language, best suited for working with structured/columnar data — extracting specific fields, performing calculations across columns, or conditional per-field logic that `grep`/`sed` alone can't easily express.

**Q9: How does `cron` syntax work? Write an expression for "every weekday at 6 PM."**
A: Cron uses five fields: minute, hour, day-of-month, month, day-of-week. For every weekday (Monday–Friday) at 6:00 PM: `0 18 * * 1-5 command` — minute 0, hour 18, any day-of-month, any month, weekdays 1 through 5 (Monday=1 in standard cron numbering... note some systems use 0 or 7 for Sunday, with 1-5 consistently meaning Mon-Fri across implementations).

**Q10: What does the "load average" shown by `uptime` actually mean?**
A: It's the average number of processes that are either running on the CPU or waiting for CPU time, averaged over the last 1, 5, and 15 minutes respectively. To interpret it meaningfully, compare it against the number of CPU cores — a load average near or below the core count generally means the system is keeping up, while a value significantly higher indicates CPU contention/queuing.

**Q11: Why might `free -h` show low "free" memory but the system still performs fine?**
A: Linux aggressively uses otherwise-idle RAM for disk cache and buffers to speed up I/O, since unused RAM provides no benefit sitting empty. This cached memory is instantly reclaimable if an application actually needs it. The "available" column (not "free") is the more meaningful indicator of how much memory is genuinely available for new processes — a system can show very little "free" memory while having plenty of "available" memory and be perfectly healthy.

**Q12: What's the difference between a process and a thread in Linux (general OS concept, frequently asked alongside Linux questions)?**
A: A process has its own independent memory space and resources; threads within the same process share that memory space and most resources but have their own execution stack/registers. Linux implements threads as lightweight processes internally (via `clone()` with shared memory flags) — so from the kernel's scheduling perspective, threads and processes are scheduled similarly, but they differ in how much they share with their "siblings."

**Q13: How would you troubleshoot a server that's running slow?**
A: Start broad and narrow down: check `top`/`htop` for an obvious CPU or memory-hogging process; check `free -h` (looking at "available," not just "free") for real memory pressure; check `df -h` for a full disk and `iostat` for disk I/O bottlenecks; check `uptime`'s load average against the CPU core count; check relevant logs (`journalctl`, `/var/log/`) for errors around the time the slowness started; and use `lsof`/`ss` if a networking/connection-related cause is suspected.

**Q14: What is the purpose of the `/proc` filesystem?**
A: `/proc` is a virtual (pseudo) filesystem that doesn't store actual files on disk — it exposes live kernel and process information as if they were files, generated on-the-fly when read. For example, `/proc/cpuinfo` shows CPU details, `/proc/meminfo` shows memory stats, and `/proc/<pid>/status` shows details about a specific running process — many system monitoring tools (like `top` and `ps`) actually read their data from `/proc` under the hood.

**Q15: What's the difference between `apt update` and `apt upgrade`?**
A: `apt update` refreshes the local cache of *available* package versions from the configured repositories — it doesn't install or change anything on the system itself. `apt upgrade` then actually installs newer versions of currently-installed packages, based on that refreshed index — you typically run `update` before `upgrade` to ensure you're upgrading based on current information.

**Q16: How does SSH key-based authentication work, and why is it more secure than passwords?**
A: The client holds a private key that never leaves their machine; the matching public key is placed in the server's `~/.ssh/authorized_keys`. During connection, the server issues a cryptographic challenge that can only be correctly answered by someone possessing the corresponding private key — the private key itself is never transmitted over the network. This is more secure than passwords because there's no secret transmitted that could be intercepted or brute-forced as easily, and it enables disabling password authentication entirely as a hardening measure.

**Q17: What is the difference between a hard disk's `df` output and `du` output, and why might they disagree?**
A: `df` reports usage at the filesystem/mount-point level — how full a given disk/partition actually is according to the kernel. `du` reports usage by walking a specific directory tree and summing file sizes. They can disagree when a file has been deleted but is still held open by a running process — the space remains allocated and counted by `df` (since the underlying data still exists on disk) even though `du` won't see it (since it's no longer reachable via any directory entry).

**Q18: What's the role of systemd, and how is it different from older init systems?**
A: systemd is the modern init system (PID 1) responsible for booting the system, starting/managing services with proper dependency ordering, handling mount points, sockets, timers, and logging (via the journal). Compared to older SysV init (which used sequential shell scripts in numbered runlevel directories), systemd uses declarative unit files, parallelizes service startup where dependencies allow, and provides much richer service management/monitoring tooling (`systemctl`, `journalctl`) out of the box.

**Q19: How would you find and kill a process by name without knowing its PID?**
A: `pkill -f processname` (or `killall processname`) finds matching processes by name/command-line pattern and sends them a signal directly, without needing to manually look up the PID first via `ps`/`pgrep`. `pgrep processname` can be used first if you just want to see the matching PID(s) before deciding to kill them.

**Q20: What's the difference between SELinux/AppArmor and standard Linux file permissions?**
A: Standard Unix permissions (`rwx`) are discretionary — controlled by the file owner, and bypassed entirely by a process running as root. SELinux and AppArmor add mandatory access control (MAC) — administrator-defined security policies that constrain what a process is allowed to do (which files it can touch, which network operations it can perform) regardless of standard permissions or even root privilege — significantly limiting the potential damage from a compromised or misbehaving process, even one running with elevated privileges.

---

### Final interview tip
Be ready to **explain file permissions and `chmod` numerically from scratch**, walk through **SIGTERM vs SIGKILL**, and talk through a **"server is running slow, how do you troubleshoot it"** scenario step by step using `top`, `free`, `df`, `iostat`, and logs — this last one is one of the single most common practical Linux interview questions across SRE/DevOps/backend roles. Also be comfortable writing a basic shell script with a loop and conditional live on a whiteboard or shared editor.
