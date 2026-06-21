# Phase 1 — Linux Foundation

## Mục tiêu
- Tự tin thao tác trên Linux server không cần giao diện đồ họa.
- Quản trị được một server: user, quyền, tiến trình, dịch vụ, gói phần mềm.
- Viết được Bash script để tự động hóa công việc lặp lại.

---

## Kiến thức sẽ học

- **Cấu trúc hệ thống file Linux** (FHS), điều hướng, thao tác file/thư mục.
- **Quyền & sở hữu** (permissions, ownership, chmod, chown, sudo).
- **Quản lý user & group** — hệ thống thật luôn nhiều người dùng/dịch vụ.
- **Quản lý tiến trình & dịch vụ** (ps, top, htop, systemd, journalctl).
- **Quản lý gói** (apt/yum/dnf).
- **Text processing** (grep, sed, awk, pipe, redirection) — kỹ năng đọc log.
- **Bash scripting** (biến, vòng lặp, điều kiện, hàm, cron job).

---

## Kiến thức chi tiết

### 1. Filesystem Hierarchy Standard (FHS)

Linux tổ chức toàn bộ hệ thống file theo một cây thư mục thống nhất, bắt đầu từ root `/`. Khác với Windows dùng ổ đĩa (C:, D:), Linux gắn mọi thiết bị vào một cây duy nhất.

#### 1.1 Các thư mục quan trọng

| Thư mục | Mục đích | Ghi chú |
|---------|----------|---------|
| `/bin` | Binary căn bản cho tất cả user | ls, cp, mv, bash |
| `/sbin` | Binary dành cho system admin | fdisk, iptables, reboot |
| `/usr/bin` | Binary ứng dụng người dùng | git, python3, node |
| `/usr/sbin` | Binary admin không thiết yếu khi boot | apache2, nginx |
| `/usr/local/bin` | Binary cài thủ công (ngoài package manager) | pip install --user |
| `/etc` | Toàn bộ config files hệ thống | /etc/nginx/, /etc/ssh/ |
| `/var/log` | Log files của hệ thống và ứng dụng | /var/log/syslog, /var/log/nginx/ |
| `/var/lib` | Dữ liệu runtime của ứng dụng | /var/lib/postgresql/, /var/lib/redis/ |
| `/var/cache` | Cache dữ liệu có thể xóa | /var/cache/apt/ |
| `/home` | Thư mục home của user thường | /home/devops/, /home/shoplite/ |
| `/root` | Thư mục home của root | Không nằm trong /home |
| `/tmp` | File tạm thời | Bị xóa khi reboot, world-writable |
| `/run` | Runtime data (PID files, sockets) | Tạo lại khi boot, nằm trong RAM |
| `/proc` | Virtual filesystem — thông tin kernel & process | /proc/cpuinfo, /proc/meminfo |
| `/sys` | Virtual filesystem — thông tin hardware/kernel | /sys/class/net/ |
| `/dev` | Device files | /dev/sda (disk), /dev/null, /dev/random |
| `/opt` | Ứng dụng third-party cài nguyên bộ | /opt/google/, /opt/shoplite/ |
| `/boot` | Kernel, initramfs, bootloader | Không sửa nếu không biết rõ |
| `/mnt` | Mount point tạm thời | Dùng để mount USB, NFS |
| `/media` | Mount tự động của removable media | CD-ROM, USB tự mount |
| `/lib` | Shared libraries cho /bin và /sbin | .so files |
| `/usr/lib` | Shared libraries cho /usr/bin | |

#### 1.2 Lệnh điều hướng và thao tác file

```bash
# Xem vị trí hiện tại
pwd

# Liệt kê file — các biến thể quan trọng
ls -la              # long format + hidden files
ls -lh              # human-readable size (KB, MB)
ls -lt              # sort by time (newest first)
ls -lS              # sort by size (largest first)
ls -lR              # recursive
ls --color=auto     # màu sắc (thường là default)

# Xem cây thư mục (cài: apt install tree)
tree -L 2           # depth 2
tree -L 2 /etc      # cây của /etc, depth 2
tree -a             # bao gồm hidden files
tree -d             # chỉ thư mục

# Di chuyển
cd /var/log         # đường dẫn tuyệt đối
cd ..               # lên 1 cấp
cd ~                # về home directory
cd -                # quay lại thư mục trước
cd /                # về root

# Tạo thư mục
mkdir logs                      # tạo 1 thư mục
mkdir -p /app/shoplite/logs     # tạo cả cây (nếu chưa có)
mkdir -p dir/{sub1,sub2,sub3}   # tạo nhiều subdirectory cùng lúc

# Xóa
rm file.txt                     # xóa file
rm -f file.txt                  # force (không hỏi)
rm -r directory/                # xóa thư mục đệ quy
rm -rf directory/               # force + recursive (NGUY HIỂM — không có undo)
rmdir empty_dir/                # chỉ xóa thư mục rỗng

# Copy
cp source.txt dest.txt          # copy file
cp -r src_dir/ dest_dir/        # copy thư mục đệ quy
cp -p file dest                 # giữ nguyên permissions và timestamps
cp -a src/ dest/                # archive mode (giống -rp + links)
cp -v file dest                 # verbose (hiện file đang copy)

# Di chuyển / đổi tên
mv oldname.txt newname.txt      # đổi tên
mv file.txt /var/log/           # di chuyển
mv -n src dst                   # không ghi đè nếu dst đã tồn tại
mv -i src dst                   # hỏi trước khi ghi đè

# Tạo file rỗng / cập nhật timestamp
touch newfile.txt
touch -m file.txt               # cập nhật modification time

# Symbolic link (như shortcut)
ln -s /app/shoplite/current /app/shoplite/active
ln -s /usr/local/node/bin/node /usr/local/bin/node
ls -la /app/shoplite/          # thấy -> trỏ đến đâu
readlink -f symlink             # giải ra đường dẫn thực

# Hard link (cùng inode)
ln original.txt hardlink.txt
```

#### 1.3 Khám phá /proc và /sys

```bash
# Thông tin CPU
cat /proc/cpuinfo | grep "model name" | head -1
nproc                           # số core
cat /proc/cpuinfo | grep processor | wc -l

# Thông tin RAM
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable"

# Load average (1 phút, 5 phút, 15 phút)
cat /proc/loadavg
uptime                          # hiện thị load average dễ đọc hơn

# Thông tin kernel
cat /proc/version
uname -r                        # chỉ kernel version

# Network interfaces từ /sys
ls /sys/class/net/
cat /sys/class/net/eth0/operstate    # up hay down
cat /sys/class/net/eth0/speed        # tốc độ (Mbps)
```

---

### 2. Permissions & Ownership

#### 2.1 Đọc hiểu permission string

Khi chạy `ls -la`, mỗi dòng bắt đầu bằng một chuỗi 10 ký tự:

```
-rwxr-xr-x  1  devops  www-data  4096  Jun 21 10:00  script.sh
^^^^^^^^^^ ^^ ^^^^^^^ ^^^^^^^^^ ^^^^
|          |  |       |         |
|          |  |       |         +-- File size
|          |  |       +------------ Group owner
|          |  +-------------------- User owner
|          +----------------------- Number of hard links
+---------------------------------- Permission bits
```

Phân tích permission string `-rwxr-xr-x`:
- Ký tự 1: `-` = regular file, `d` = directory, `l` = symlink, `b` = block device, `c` = char device
- Ký tự 2-4: `rwx` = quyền của **owner** (read, write, execute)
- Ký tự 5-7: `r-x` = quyền của **group** (read, no-write, execute)
- Ký tự 8-10: `r-x` = quyền của **others** (read, no-write, execute)

Ý nghĩa từng bit:
| Ký tự | File | Directory |
|-------|------|-----------|
| `r` (4) | Đọc nội dung | List files trong thư mục |
| `w` (2) | Sửa nội dung | Tạo/xóa file trong thư mục |
| `x` (1) | Thực thi | Vào thư mục (cd) |
| `-` (0) | Không có quyền | Không có quyền |

#### 2.2 chmod — thay đổi quyền

**Dạng symbolic (dễ đọc, an toàn hơn):**

```bash
# u = user/owner, g = group, o = others, a = all
# + thêm quyền, - bỏ quyền, = đặt chính xác

chmod u+x script.sh             # thêm execute cho owner
chmod g-w config.conf           # bỏ write của group
chmod o=r public.txt            # set others chỉ có read
chmod a+x deploy.sh             # thêm execute cho tất cả
chmod u+x,g+r,o-rwx secret.sh  # nhiều thay đổi cùng lúc
chmod -R g+rX /app/shoplite     # recursive: thêm read + execute (chỉ thư mục) cho group
```

**Dạng numeric (octal — nhanh hơn khi thành thạo):**

```bash
# Công thức: r=4, w=2, x=1, cộng lại
# 7 = 4+2+1 = rwx
# 6 = 4+2+0 = rw-
# 5 = 4+0+1 = r-x
# 4 = 4+0+0 = r--
# 0 = 0+0+0 = ---

chmod 755 script.sh     # rwxr-xr-x — script chạy được cho tất cả
chmod 644 config.conf   # rw-r--r-- — config đọc được, chỉ owner sửa
chmod 600 private.key   # rw------- — SSH key, chỉ owner đọc/sửa
chmod 700 ~/.ssh        # rwx------ — thư mục SSH, chỉ owner vào được
chmod 640 app.log       # rw-r----- — log đọc được bởi group
chmod 750 /app/shoplite # rwxr-x--- — app dir, group chạy được, others không vào
chmod 777 /tmp/shared   # rwxrwxrwx — mọi người đều có full quyền (TRÁNH trong prod)

# Recursive
chmod -R 750 /app/shoplite
chmod -R 644 /app/shoplite/public  # static files
```

**Ví dụ thực tế cho ShopLite:**

```bash
# Cấu trúc quyền cho ứng dụng ShopLite
/app/shoplite/
├── current/          # 750 shoplite:www-data — app code
├── releases/         # 750 shoplite:www-data
├── shared/
│   ├── .env          # 600 shoplite:shoplite — secrets, chỉ app đọc
│   ├── logs/         # 770 shoplite:www-data — app + web server ghi log
│   └── uploads/      # 775 shoplite:www-data — web server upload được
└── scripts/          # 750 shoplite:shoplite — scripts

# Áp dụng
chmod 600 /app/shoplite/shared/.env
chmod -R 770 /app/shoplite/shared/logs
chmod -R 775 /app/shoplite/shared/uploads
```

#### 2.3 chown — thay đổi sở hữu

```bash
# Cú pháp: chown [user]:[group] file
chown shoplite /app/shoplite/current
chown shoplite:www-data /app/shoplite/current
chown :www-data /app/shoplite/shared/uploads  # chỉ đổi group
chown -R shoplite:www-data /app/shoplite      # recursive

# Xem user/group hiện tại
ls -la /app/shoplite
stat /app/shoplite/current

# Ví dụ setup thực tế
# Tạo user cho app
useradd -r -s /sbin/nologin shoplite          # system user, không login shell
# Tạo thư mục và phân quyền
mkdir -p /app/shoplite/{current,releases,shared/logs,shared/uploads}
chown -R shoplite:www-data /app/shoplite
chmod -R 750 /app/shoplite
chmod -R 775 /app/shoplite/shared/uploads
chmod 600 /app/shoplite/shared/.env
```

#### 2.4 sudo — leo thang đặc quyền

```bash
# Chạy lệnh với quyền root
sudo apt update
sudo systemctl restart nginx
sudo -i                         # mở shell root (interactive)
sudo -s                         # mở shell root (current env)
sudo -u shoplite command        # chạy với quyền user khác
sudo -l                         # liệt kê quyền sudo của user hiện tại

# Cấu hình sudoers — LUÔN dùng visudo (kiểm tra syntax trước khi save)
sudo visudo
# Hoặc tạo file riêng (khuyến nghị — dễ quản lý hơn)
sudo visudo -f /etc/sudoers.d/devops

# Cú pháp sudoers:
# user    ALL=(ALL:ALL) ALL          -- full quyền, cần password
# user    ALL=(ALL:ALL) NOPASSWD:ALL -- full quyền, không cần password (NGUY HIỂM)
# %group  ALL=(ALL) ALL              -- group (bắt đầu bằng %)

# Ví dụ thực tế — giới hạn quyền:
# devops user chỉ được restart các service cụ thể
devops ALL=(root) NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl restart shoplite
# deploy user chỉ được chạy deploy script
deploy ALL=(shoplite) NOPASSWD: /app/shoplite/scripts/deploy.sh

# File /etc/sudoers.d/ — tạo file riêng cho từng user/role
# /etc/sudoers.d/devops-team
%devops ALL=(ALL) NOPASSWD: /bin/systemctl, /usr/bin/apt
```

#### 2.5 Special permissions: SUID, SGID, Sticky Bit

```bash
# SUID (Set User ID) — bit 4 — chạy file với quyền owner
# Ví dụ: /usr/bin/passwd chạy với quyền root dù user thường gọi
ls -la /usr/bin/passwd
# -rwsr-xr-x  (s ở vị trí execute của owner = SUID + executable)
chmod u+s /usr/bin/special      # symbolic
chmod 4755 /usr/bin/special     # numeric (4 = SUID)

# SGID (Set Group ID) — bit 2
# Trên file: chạy với quyền group owner
# Trên directory: file mới tạo trong thư mục kế thừa group
chmod g+s /app/shoplite/shared  # file mới tạo trong shared có group = www-data
chmod 2750 /app/shoplite/shared

# Sticky Bit — bit 1 — trên directory: chỉ owner mới xóa được file của mình
ls -la /tmp
# drwxrwxrwt  (t ở cuối = sticky bit)
# Ứng dụng: /tmp ai cũng write, nhưng không xóa file của người khác
chmod +t /shared/uploads
chmod 1777 /tmp                 # standard /tmp permissions

# Tìm file có SUID/SGID (kiểm tra bảo mật)
find / -perm -4000 -type f 2>/dev/null    # SUID files
find / -perm -2000 -type f 2>/dev/null    # SGID files
```

---

### 3. User & Group Management

#### 3.1 Tạo và quản lý user

```bash
# useradd — tạo user
useradd username                          # tạo user cơ bản (không home, không shell tốt)
useradd -m username                       # -m: tạo home directory
useradd -m -s /bin/bash username          # -s: chỉ định shell
useradd -m -s /bin/bash -c "DevOps Engineer" devops  # -c: comment/full name
useradd -m -s /bin/bash -G sudo,docker devops        # -G: thêm vào groups
useradd -r -s /sbin/nologin shoplite      # -r: system user (uid < 1000), không login

# Đặt password
passwd username                           # interactive
echo "username:password" | chpasswd      # non-interactive (script)
passwd -e username                        # force change on next login
passwd -l username                        # lock account
passwd -u username                        # unlock account

# usermod — sửa user
usermod -aG docker devops                 # -a: append, -G: supplementary group
usermod -aG sudo devops                   # thêm vào sudo group
usermod -s /bin/zsh devops               # đổi shell
usermod -d /new/home devops              # đổi home dir (không move file)
usermod -d /new/home -m devops           # đổi home dir + move files
usermod -l newname oldname               # đổi username

# userdel — xóa user
userdel username                          # xóa user, GIỮ home directory
userdel -r username                       # xóa user + home + mail spool

# Xem thông tin user
id devops                                 # uid, gid, groups
groups devops                             # chỉ groups
who                                       # ai đang login
w                                         # chi tiết hơn who
last                                      # lịch sử login
lastlog                                   # lần login cuối của mỗi user
```

#### 3.2 Cấu trúc file /etc/passwd và /etc/shadow

```bash
# /etc/passwd — thông tin user (đọc được bởi tất cả)
cat /etc/passwd | head -5
# Format: username:x:uid:gid:comment:home_dir:shell
# root:x:0:0:root:/root:/bin/bash
# shoplite:x:1001:1001:ShopLite App:/home/shoplite:/bin/bash
# nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin

# Giải thích:
# username   — tên đăng nhập
# x          — password (đã chuyển sang /etc/shadow)
# uid        — User ID (0=root, 1-999=system, 1000+=regular)
# gid        — Primary Group ID
# comment    — Tên đầy đủ hoặc mô tả
# home_dir   — Thư mục home
# shell      — Login shell (/bin/bash, /sbin/nologin, /bin/false)

# /etc/shadow — mật khẩu đã hash (chỉ root đọc được)
sudo cat /etc/shadow | grep shoplite
# Format: username:$hash:lastchanged:min:max:warn:inactive:expire:reserved
# shoplite:$6$salt$hashedpassword:19000:0:99999:7:::

# /etc/group — thông tin group
cat /etc/group | grep docker
# Format: groupname:x:gid:members
# docker:x:999:devops,jenkins
```

#### 3.3 Quản lý group

```bash
# Tạo group
groupadd developers
groupadd -g 2000 devops-team    # chỉ định GID cụ thể

# Thêm/xóa member
gpasswd -a devops developers    # thêm devops vào group developers
gpasswd -d devops developers    # xóa devops khỏi group

# Xóa group
groupdel developers

# Thay đổi group hiện tại (trong session)
newgrp docker                   # chuyển sang group docker cho session này
```

#### 3.4 Chuyển đổi user

```bash
# su — switch user
su - username                   # login shell (load profile của user đó)
su username                     # chỉ đổi uid, giữ environment
su -                            # chuyển sang root với login shell
su - -c "command" username      # chạy lệnh với quyền user khác

# sudo với user cụ thể
sudo -u shoplite /app/shoplite/scripts/start.sh
sudo -u postgres psql           # chạy psql với quyền postgres

# Kiểm tra environment
env                             # xem tất cả biến môi trường
sudo env                        # env khi sudo (thường sạch hơn)
sudo -E env                     # sudo giữ nguyên environment của user hiện tại
```

---

### 4. Process Management

#### 4.1 ps — xem processes

```bash
# ps aux — xem tất cả process
ps aux
# Các cột:
# USER   — owner của process
# PID    — Process ID
# %CPU   — CPU usage
# %MEM   — Memory usage (% of RAM)
# VSZ    — Virtual memory size (KB)
# RSS    — Resident Set Size — RAM thực sự dùng (KB)
# TTY    — terminal (?=none, pts/0=ssh session)
# STAT   — trạng thái process
# START  — thời điểm khởi động
# TIME   — tổng CPU time đã dùng
# COMMAND — lệnh khởi động process

# STAT codes — quan trọng để debug
# R  — Running (đang chạy)
# S  — Sleeping, interruptible (chờ event)
# D  — Disk sleep, uninterruptible (chờ I/O — không kill được bằng SIGTERM)
# Z  — Zombie (đã chết nhưng chưa được parent reap)
# T  — Stopped (Ctrl+Z hoặc SIGSTOP)
# I  — Idle kernel thread
# <  — high priority
# N  — low priority (nice > 0)
# +  — foreground process group
# l  — multi-threaded
# s  — session leader

ps aux | grep nginx             # tìm process nginx
ps aux | grep -v grep | grep nginx  # loại bỏ grep khỏi kết quả
ps -ef                          # format khác, hiện PPID (parent PID)
ps -ef --forest                 # hiện process tree
ps -p 1234                      # thông tin process cụ thể theo PID
ps -u shoplite                  # processes của user shoplite
```

#### 4.2 top và htop

```bash
# top — real-time process monitor
top
# Dòng summary trên cùng:
# top - 10:30:01 up 5 days, 2:30, 2 users, load average: 0.52, 0.48, 0.41
#                                                           ^^^^^^^^^^^^^^^^^^^^
#                                                           load avg 1/5/15 phút

# Giải thích load average:
# Load average = số lượng process đang chạy HOẶC chờ CPU/disk I/O
# Rule of thumb: load avg nên < số CPU cores
# Nếu bạn có 4 core và load avg = 4.0 → CPU đang 100% utilization
# Nếu load avg = 8.0 trên 4 core → có 4 process đang đợi CPU

# Trong top, gõ:
# q         — thoát
# k         — kill process (nhập PID)
# r         — renice (đổi priority)
# M         — sort by memory
# P         — sort by CPU (default)
# T         — sort by time
# 1         — hiện từng CPU core
# u username — lọc theo user
# f         — chọn columns hiển thị

# htop — interactive, đẹp hơn top (cài: apt install htop)
htop
# Phím tắt trong htop:
# F2        — cấu hình
# F3        — search
# F4        — filter
# F5        — tree view (hiện process hierarchy)
# F6        — sort by column
# F9        — kill (gửi signal)
# F10       — quit
# Space     — tag process
# /         — search (giống F3)
```

#### 4.3 Signals và kill

```bash
# Signals quan trọng
# SIGHUP  (1)  — Hangup: reload configuration, không dừng process
# SIGINT  (2)  — Interrupt: Ctrl+C, graceful stop
# SIGTERM (15) — Terminate: yêu cầu dừng gracefully (mặc định của kill)
# SIGKILL (9)  — Kill: force kill, KHÔNG thể bị ignore hoặc catch
# SIGUSR1 (10) — User-defined: thường dùng để trigger log rotation
# SIGUSR2 (12) — User-defined: tùy ứng dụng

# Gửi signal
kill 1234                       # SIGTERM (15) — mặc định
kill -15 1234                   # SIGTERM rõ ràng
kill -TERM 1234                 # cùng SIGTERM, dùng tên
kill -9 1234                    # SIGKILL — force, dùng khi SIGTERM không hiệu quả
kill -HUP 1234                  # SIGHUP — reload config (nginx, rsyslog)
kill -0 1234                    # kiểm tra process có tồn tại không (exit code)
killall nginx                   # kill tất cả process tên nginx
killall -9 nginx                # force kill tất cả nginx

# pkill/pgrep — theo tên hoặc pattern
pgrep nginx                     # liệt kê PID của nginx
pgrep -f "node.*shoplite"       # tìm theo full command line
pkill nginx                     # kill process tên nginx (SIGTERM)
pkill -9 -f "node.*shoplite"    # kill theo pattern, force

# Quy trình xử lý process hung:
# 1. Thử SIGTERM trước
kill -15 $(pgrep -f shoplite)
# 2. Đợi vài giây
sleep 5
# 3. Nếu vẫn còn, dùng SIGKILL
kill -9 $(pgrep -f shoplite)
```

#### 4.4 Background processes và job control

```bash
# Chạy process ở background
command &                       # chạy background, sẽ chết khi terminal đóng
nohup command &                 # chạy background, tồn tại khi logout
nohup ./start-server.sh > /var/log/shoplite/start.log 2>&1 &

# Job control
jobs                            # liệt kê background jobs của terminal hiện tại
jobs -l                         # bao gồm PID
fg                              # đưa job gần nhất về foreground
fg %2                           # đưa job số 2 về foreground
bg                              # tiếp tục job bị stopped ở background
bg %2                           # tiếp tục job 2 ở background
Ctrl+Z                          # stop foreground process (đưa về background, trạng thái T)
disown                          # detach job khỏi terminal (tồn tại sau khi đóng terminal)
disown -h %1                    # keep running sau khi terminal đóng

# screen và tmux (session management — nên dùng thay nohup)
# tmux new -s deploy             # tạo session tên deploy
# tmux attach -t deploy          # attach vào session
# tmux ls                        # liệt kê sessions
# Ctrl+B D                       # detach (session vẫn chạy)
```

---

### 5. systemd Service Management

#### 5.1 systemctl — quản lý services

```bash
# Các lệnh cơ bản
systemctl start nginx           # khởi động service
systemctl stop nginx            # dừng service
systemctl restart nginx         # dừng và khởi động lại
systemctl reload nginx          # reload config không dừng service (nếu service hỗ trợ)
systemctl status nginx          # xem trạng thái chi tiết

# Enable/disable — tự động khởi động khi boot
systemctl enable nginx          # enable (tạo symlink)
systemctl disable nginx         # disable
systemctl enable --now nginx    # enable + start ngay lập tức
systemctl disable --now nginx   # disable + stop ngay lập tức

# Kiểm tra trạng thái
systemctl is-active nginx       # active/inactive/failed
systemctl is-enabled nginx      # enabled/disabled/static
systemctl is-failed nginx       # trả về 0 nếu failed

# Liệt kê services
systemctl list-units --type=service             # tất cả service đang load
systemctl list-units --type=service --state=running   # chỉ đang chạy
systemctl list-units --type=service --state=failed    # chỉ failed
systemctl list-unit-files --type=service        # tất cả service files

# Reload systemd sau khi sửa unit file
systemctl daemon-reload

# Xem dependencies
systemctl list-dependencies nginx
systemctl list-dependencies --reverse nginx     # ai depend vào nginx
```

#### 5.2 journalctl — đọc logs

```bash
# Xem logs của service
journalctl -u nginx                     # tất cả log của nginx
journalctl -u nginx -f                  # follow (real-time, như tail -f)
journalctl -u nginx -n 50               # 50 dòng cuối
journalctl -u nginx -n 100 --no-pager   # không dùng pager (dùng trong script)
journalctl -u nginx --since today       # từ hôm nay
journalctl -u nginx --since "2024-01-15 10:00:00"
journalctl -u nginx --since "2024-01-15" --until "2024-01-16"
journalctl -u nginx -p err              # chỉ error level trở lên
journalctl -u nginx -p warning..err    # từ warning đến error

# Priority levels: emerg(0), alert(1), crit(2), err(3), warning(4), notice(5), info(6), debug(7)

# Logs hệ thống
journalctl                              # tất cả logs
journalctl -b                           # logs từ lần boot hiện tại
journalctl -b -1                        # logs từ lần boot trước
journalctl --list-boots                 # liệt kê các lần boot

# Filter
journalctl _PID=1234                    # logs của PID cụ thể
journalctl _UID=1001                    # logs của UID cụ thể
journalctl -k                           # kernel messages (dmesg)

# Disk usage của journal
journalctl --disk-usage
journalctl --vacuum-size=500M           # giới hạn 500MB
journalctl --vacuum-time=7d             # xóa log cũ hơn 7 ngày
```

#### 5.3 Viết systemd unit file

Tạo file `/etc/systemd/system/shoplite.service`:

```ini
[Unit]
Description=ShopLite Application Server
Documentation=https://github.com/yourorg/shoplite
After=network.target postgresql.service redis.service
Wants=postgresql.service redis.service
# After: đảm bảo start sau các service này
# Wants: nếu postgresql/redis không start được, shoplite vẫn thử start
# Requires: nếu postgresql/redis fail, shoplite cũng fail

[Service]
# User và group chạy service
User=shoplite
Group=www-data

# Thư mục làm việc
WorkingDirectory=/app/shoplite/current

# Load environment variables từ file
EnvironmentFile=/app/shoplite/shared/.env

# Lệnh khởi động
ExecStart=/usr/bin/node /app/shoplite/current/server.js
# Với Python:
# ExecStart=/usr/bin/python3 /app/shoplite/current/main.py

# Lệnh reload (optional — nếu app hỗ trợ graceful reload)
ExecReload=/bin/kill -HUP $MAINPID

# Restart policy
Restart=always                  # luôn restart nếu exit
# Restart=on-failure            # chỉ restart khi exit code != 0
# Restart=on-abnormal           # restart khi signal, timeout, watchdog
RestartSec=5                    # đợi 5 giây trước khi restart
StartLimitInterval=60           # trong 60 giây
StartLimitBurst=3               # không restart quá 3 lần
# Nếu restart quá 3 lần trong 60 giây → systemd đánh dấu failed

# Resource limits
LimitNOFILE=65536               # max open file descriptors
LimitNPROC=4096                 # max processes
MemoryLimit=512M                # giới hạn RAM (systemd v208+)
# MemoryMax=512M                # systemd v230+ syntax

# Security hardening
NoNewPrivileges=true            # không cho escalate privilege
PrivateTmp=true                 # /tmp riêng, không chia sẻ

# Standard I/O
StandardOutput=journal          # stdout → journald
StandardError=journal           # stderr → journald

# Timeout
TimeoutStartSec=30              # nếu không start trong 30s → fail
TimeoutStopSec=30               # nếu không stop trong 30s → SIGKILL

[Install]
WantedBy=multi-user.target
# multi-user.target = runlevel 3 (non-graphical multi-user)
# graphical.target = runlevel 5 (với GUI)
```

```bash
# Sau khi tạo/sửa unit file
sudo systemctl daemon-reload
sudo systemctl enable shoplite
sudo systemctl start shoplite
sudo systemctl status shoplite
journalctl -u shoplite -f
```

---

### 6. Package Management (Ubuntu/Debian)

#### 6.1 apt — cơ bản

```bash
# Cập nhật package list (KHÔNG upgrade gì cả)
sudo apt update
# Output giải thích:
# Hit  — package list chưa thay đổi
# Get  — download package list mới
# Ign  — bỏ qua (không quan trọng)

# Upgrade packages đã cài
sudo apt upgrade                # upgrade nhưng không xóa packages
sudo apt full-upgrade           # upgrade + xóa packages cũ nếu cần (như dist-upgrade)
sudo apt upgrade -y             # tự động yes (dùng trong script)

# Cài packages
sudo apt install nginx
sudo apt install -y nodejs npm postgresql redis-server    # -y: không hỏi
sudo apt install -y --no-install-recommends nginx        # không cài recommended packages
sudo apt install ./local-package.deb                     # cài từ file .deb local

# Xóa packages
sudo apt remove nginx           # xóa package, giữ config files
sudo apt purge nginx            # xóa package + config files
sudo apt autoremove             # xóa packages không còn cần thiết
sudo apt autoremove --purge     # autoremove + xóa config
sudo apt autoclean              # xóa file .deb cũ trong cache
sudo apt clean                  # xóa toàn bộ cache (giải phóng disk)
```

#### 6.2 apt — nâng cao

```bash
# Tìm kiếm và xem thông tin
apt-cache search keyword        # tìm package theo tên/mô tả
apt-cache show nginx            # thông tin chi tiết của package
apt-cache showpkg nginx         # dependencies
apt-cache policy nginx          # version có sẵn và đã cài

# apt search và apt show (modern syntax)
apt search nodejs
apt show nodejs

# Kiểm tra package đã cài
dpkg -l | grep nginx            # tìm package đã cài
dpkg -l | grep "^ii"            # chỉ package đang installed (ii = installed)
dpkg -L nginx                   # liệt kê files của package nginx
dpkg -S /usr/sbin/nginx         # tìm xem file này thuộc package nào
dpkg --get-selections           # tất cả packages đã cài

# Giữ version không cho upgrade
sudo apt-mark hold nginx
sudo apt-mark unhold nginx
apt-mark showhold               # xem packages đang hold
```

#### 6.3 Thêm third-party repository

```bash
# Phương pháp hiện đại (Ubuntu 22.04+) — dùng signed-by
# Ví dụ: thêm NodeSource repo cho Node.js 20
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg

echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" \
  | sudo tee /etc/apt/sources.list.d/nodesource.list

sudo apt update
sudo apt install -y nodejs

# Phương pháp cũ (deprecated nhưng còn gặp)
# curl -fsSL https://example.com/key.gpg | sudo apt-key add -    # DEPRECATED
# /etc/apt/trusted.gpg.d/ — cách mới

# Thêm PPA (Personal Package Archive — Ubuntu specific)
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.12

# Xóa PPA
sudo add-apt-repository --remove ppa:deadsnakes/ppa

# Liệt kê repositories
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/
```

#### 6.4 snap và các package manager khác

```bash
# snap — universal packages (sandboxed)
snap find nodejs
snap install node --classic     # --classic: bỏ sandbox
snap list                       # packages đã cài
snap remove node
snap refresh                    # update all snaps

# Khi nào dùng snap:
# - Package không có trong apt
# - Cần version mới nhất
# - Ứng dụng desktop

# flatpak — chủ yếu cho desktop apps
# pip — Python packages
pip3 install package            # cài globally (hạn chế)
pip3 install --user package     # cài cho user hiện tại
python3 -m venv venv && source venv/bin/activate  # virtualenv (khuyến nghị)
pip install package             # trong virtualenv

# npm — Node.js packages
npm install -g package          # global
npm install package             # local (trong project)
```

---

### 7. Text Processing Thực Chiến

#### 7.1 grep — tìm kiếm pattern

```bash
# Cơ bản
grep "ERROR" /var/log/syslog
grep -i "error" /var/log/syslog          # case-insensitive
grep -r "shoplite" /etc/                 # recursive trong thư mục
grep -v "DEBUG" app.log                  # invert — dòng KHÔNG chứa DEBUG
grep -n "error" app.log                  # hiện số dòng
grep -c "error" app.log                  # đếm số dòng match
grep -l "error" /var/log/*.log           # chỉ hiện tên file

# Context
grep -A 3 "ERROR" app.log               # 3 dòng After (sau) match
grep -B 2 "ERROR" app.log               # 2 dòng Before (trước) match
grep -C 5 "CRASH" app.log               # 5 dòng trước và sau

# Extended regex (-E hoặc egrep)
grep -E "error|warning|critical" app.log
grep -E "^[0-9]{4}-[0-9]{2}-[0-9]{2}" app.log  # dòng bắt đầu bằng ngày
grep -E "https?://" app.log             # http hoặc https

# Hữu ích trong DevOps
grep -r "password" /etc/ 2>/dev/null    # tìm password trong config (audit)
grep -rn "TODO\|FIXME" /app/shoplite/   # tìm TODOs trong code
grep -E "5[0-9]{2}" /var/log/nginx/access.log | head -20  # 5xx errors
journalctl -u shoplite | grep -E "error|exception|fatal" -i

# Fixed string (không interpret regex)
grep -F "1.2.3.4" /var/log/nginx/access.log
```

#### 7.2 sed — stream editor

```bash
# Substitute (thay thế)
sed 's/old/new/' file.txt               # thay thế lần đầu tiên trên mỗi dòng
sed 's/old/new/g' file.txt              # thay thế TẤT CẢ (g=global)
sed 's/old/new/gi' file.txt             # case-insensitive + global
sed -i 's/old/new/g' file.txt          # in-place (sửa file trực tiếp)
sed -i.bak 's/old/new/g' file.txt      # in-place + backup file.txt.bak

# Xóa dòng
sed '/pattern/d' file.txt              # xóa dòng chứa pattern
sed '/^#/d' config.txt                 # xóa comment lines
sed '/^$/d' file.txt                   # xóa dòng trống
sed '5d' file.txt                      # xóa dòng 5
sed '5,10d' file.txt                   # xóa dòng 5 đến 10

# In dòng cụ thể
sed -n '5p' file.txt                   # in dòng 5 (-n: không in tự động)
sed -n '5,10p' file.txt                # in dòng 5 đến 10
sed -n '/START/,/END/p' file.txt       # in từ dòng START đến END

# Thêm dòng
sed '/pattern/a\new line after' file.txt   # thêm dòng sau pattern
sed '/pattern/i\new line before' file.txt  # thêm dòng trước pattern
sed '1i\# This is a header' file.txt        # thêm header vào đầu file

# Ví dụ thực tế
# Cập nhật config nginx port
sed -i 's/listen 80/listen 8080/' /etc/nginx/sites-available/shoplite

# Xóa comments và dòng trống trong config
sed '/^#/d; /^$/d' /etc/nginx/nginx.conf

# Đổi database host trong .env
sed -i "s/DB_HOST=localhost/DB_HOST=10.0.1.5/" /app/shoplite/.env
```

#### 7.3 awk — data processing

```bash
# Cơ bản
awk '{print $1}' file.txt               # in cột 1 (space-separated)
awk '{print $1, $4}' file.txt           # in cột 1 và 4
awk '{print NR, $0}' file.txt           # in số dòng + toàn bộ dòng
awk 'NR==5' file.txt                    # in dòng số 5
awk 'NR>5 && NR<10' file.txt            # in dòng 6 đến 9

# Field separator
awk -F: '{print $1}' /etc/passwd        # separator là :
awk -F',' '{print $2}' data.csv         # separator là ,
awk -F'\t' '{print $3}' data.tsv        # separator là tab

# Điều kiện
awk '$3 > 100' file.txt                 # in dòng có cột 3 > 100
awk '/pattern/ {print $0}' file.txt     # in dòng match pattern (như grep)
awk '!/pattern/ {print}' file.txt       # in dòng KHÔNG match

# Tính toán
awk '{sum+=$1} END{print "Sum:", sum}' numbers.txt
awk '{sum+=$1} END{print "Average:", sum/NR}' numbers.txt
awk 'BEGIN{max=0} $1>max{max=$1} END{print "Max:", max}' numbers.txt

# Xử lý log — ví dụ thực tế
# Đếm HTTP status codes trong nginx access log
# Format: 192.168.1.1 - - [01/Jan/2024] "GET /api/users HTTP/1.1" 200 1234
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Tìm request chậm (>1000ms) — giả sử cột cuối là response time
awk '$NF > 1000 {print $7, $NF "ms"}' /var/log/nginx/access.log

# Tính tổng bandwidth
awk '{sum+=$10} END{print sum/1024/1024 " MB"}' /var/log/nginx/access.log

# Xử lý /etc/passwd
awk -F: '$3 >= 1000 {print $1, $6, $7}' /etc/passwd   # regular users

# BEGIN và END blocks
awk 'BEGIN{print "Start"} {print NR, $0} END{print "Total:", NR " lines"}' file.txt

# Multiple field separators
awk -F'[,;]' '{print $1}' file.txt     # separator là , hoặc ;
```

#### 7.4 Các lệnh text processing khác

```bash
# cut — cắt cột
cut -d: -f1 /etc/passwd                 # delimiter :, lấy field 1
cut -d, -f1,3 data.csv                  # field 1 và 3
cut -c1-10 file.txt                     # ký tự 1 đến 10

# sort — sắp xếp
sort file.txt                           # alphabetical
sort -n numbers.txt                     # numeric sort
sort -rn numbers.txt                    # reverse numeric
sort -k2 -n file.txt                    # sort by column 2, numeric
sort -k2 -k3 file.txt                   # sort by col 2, then col 3
sort -u file.txt                        # unique (bỏ duplicate)
sort -t: -k3 -n /etc/passwd             # sort by UID

# uniq — xử lý duplicate
sort file.txt | uniq                    # loại bỏ duplicate (phải sort trước)
sort file.txt | uniq -c                 # đếm số lần xuất hiện
sort file.txt | uniq -d                 # chỉ hiện duplicate lines
sort file.txt | uniq -u                 # chỉ hiện unique lines (không duplicate)

# wc — word count
wc -l file.txt                          # đếm lines
wc -w file.txt                          # đếm words
wc -c file.txt                          # đếm bytes
wc -m file.txt                          # đếm characters
ls /var/log/*.log | wc -l               # đếm số log files

# tr — translate/delete characters
echo "hello world" | tr a-z A-Z         # lowercase to uppercase
echo "hello world" | tr -d 'aeiou'      # xóa vowels
echo "hello   world" | tr -s ' '        # squeeze multiple spaces thành 1
echo "hello\nworld" | tr -d '\n'        # xóa newlines
cat file.txt | tr '\t' ' '              # đổi tab thành space

# head và tail
head -20 file.txt                       # 20 dòng đầu
head -c 1024 file.txt                   # 1024 bytes đầu
tail -20 file.txt                       # 20 dòng cuối
tail -f /var/log/nginx/error.log        # follow (real-time)
tail -F /var/log/nginx/error.log        # follow + reopen nếu file rotate
tail -n +5 file.txt                     # từ dòng 5 trở đi (bỏ 4 dòng đầu)

# diff và patch
diff file1.txt file2.txt
diff -u file1.txt file2.txt             # unified format (dễ đọc hơn)
diff -r dir1/ dir2/                     # so sánh 2 thư mục
diff -u original.conf new.conf > changes.patch
patch < changes.patch                   # áp dụng patch
```

#### 7.5 Pipe và Redirection

```bash
# Pipe — output của lệnh này là input của lệnh kia
command1 | command2 | command3

# Ví dụ thực tế
cat /var/log/nginx/access.log \
  | grep "$(date +%d/%b/%Y)" \          # chỉ logs hôm nay
  | grep " 5[0-9][0-9] " \             # 5xx status
  | awk '{print $1}' \                 # lấy IP
  | sort | uniq -c \                   # đếm theo IP
  | sort -rn \                         # sort giảm dần
  | head -10                           # top 10 IP gửi nhiều 5xx nhất

# Redirection
command > output.txt                   # stdout → file (ghi đè)
command >> output.txt                  # stdout → file (append)
command 2> error.txt                   # stderr → file
command 2>&1                           # stderr → stdout (merge)
command > output.txt 2>&1             # cả stdout và stderr → file
command &> output.txt                  # shorthand cho trên (bash only)
command > /dev/null                    # bỏ stdout
command > /dev/null 2>&1              # bỏ cả stdout và stderr

# Ví dụ trong cron/script
/app/shoplite/scripts/backup.sh >> /var/log/shoplite/backup.log 2>&1

# tee — ghi vào file VÀ vẫn in ra stdout
command | tee output.txt               # ghi + hiện
command | tee -a output.txt            # ghi (append) + hiện
command | tee output.txt | grep ERROR  # pipe tiếp

# Process substitution
diff <(sort file1.txt) <(sort file2.txt)   # sort rồi diff không cần file temp

# Here document — input nhiều dòng
cat << 'EOF' > /etc/nginx/sites-available/shoplite
server {
    listen 80;
    server_name shoplite.local;
    location / {
        proxy_pass http://localhost:3000;
    }
}
EOF
```

---

### 8. Bash Scripting Đầy Đủ

#### 8.1 Cấu trúc script và best practices

```bash
#!/bin/bash
# ^^^ Shebang — chỉ định interpreter

# set options — best practice cho production scripts
set -e          # exit ngay khi có lệnh lỗi (exit code != 0)
set -u          # lỗi nếu dùng biến chưa khai báo
set -o pipefail # pipeline fail nếu bất kỳ lệnh nào trong pipe fail
# Kết hợp:
set -euo pipefail

# Ví dụ tại sao quan trọng:
# Không có set -e: rm -rf $DIR/* — nếu $DIR rỗng → rm -rf /*
# Với set -u: lỗi ngay khi $DIR chưa được set

# Script header chuẩn
#!/bin/bash
set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/shoplite/${SCRIPT_NAME%.sh}.log"

# Logging function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$SCRIPT_NAME] $*" | tee -a "$LOG_FILE"
}

log_info()    { log "INFO:  $*"; }
log_warn()    { log "WARN:  $*"; }
log_error()   { log "ERROR: $*" >&2; }

# Cleanup function — chạy khi script exit (kể cả lỗi)
cleanup() {
    local exit_code=$?
    log_info "Script exiting with code: $exit_code"
    # Xóa temp files
    [[ -f "$LOCK_FILE" ]] && rm -f "$LOCK_FILE"
}
trap cleanup EXIT
trap 'log_error "Signal received, exiting..."; exit 1' INT TERM
```

#### 8.2 Biến và các phép toán

```bash
# Khai báo biến
VAR="hello world"               # không có space xung quanh =
READONLY_VAR="constant"
declare -r CONST="immutable"    # readonly
declare -i NUM=42               # integer
declare -a ARRAY=(a b c)        # array
declare -A HASH                 # associative array (bash 4+)

# Giá trị mặc định
${VAR:-default}                 # nếu VAR chưa set hoặc rỗng → dùng "default"
${VAR:=default}                 # nếu VAR chưa set → set VAR="default" và dùng
${VAR:+other}                   # nếu VAR đã set và không rỗng → dùng "other"
${VAR:?error message}           # nếu VAR chưa set → in error và exit

# Ví dụ thực tế
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
APP_ENV="${APP_ENV:?APP_ENV must be set (dev/staging/prod)}"

# String operations
str="Hello, World!"
echo ${#str}                    # độ dài: 13
echo ${str:7}                   # từ index 7: "World!"
echo ${str:7:5}                 # từ index 7, lấy 5 ký tự: "World"
echo ${str/World/Linux}         # replace first: "Hello, Linux!"
echo ${str//l/L}                # replace all: "HeLLo, WorLd!"
echo ${str#Hello, }             # xóa prefix: "World!"
echo ${str%!}                   # xóa suffix: "Hello, World"
echo ${str^^}                   # UPPERCASE: "HELLO, WORLD!"
echo ${str,,}                   # lowercase: "hello, world!"

# Arithmetic
a=10; b=3
echo $((a + b))                 # 13
echo $((a - b))                 # 7
echo $((a * b))                 # 30
echo $((a / b))                 # 3 (integer division)
echo $((a % b))                 # 1 (modulo)
echo $((a ** b))                # 1000 (exponentiation)
((a++))                         # increment (không echo)
((a += 5))                      # a = a + 5
let "result = a * b + 2"

# bc cho float
echo "scale=2; 10/3" | bc      # 3.33
```

#### 8.3 Điều kiện và kiểm tra

```bash
# if statement
if [ condition ]; then
    # ...
elif [ condition ]; then
    # ...
else
    # ...
fi

# Double bracket [[ ]] — modern, an toàn hơn với string
if [[ "$VAR" == "prod" ]]; then
    echo "Production environment"
fi

# Kiểm tra file/directory
[ -f "/etc/nginx/nginx.conf" ]  # file tồn tại và là regular file
[ -d "/app/shoplite" ]           # directory tồn tại
[ -e "/path/to/thing" ]          # tồn tại (bất kỳ loại)
[ -r "/file" ]                   # readable
[ -w "/file" ]                   # writable
[ -x "/script.sh" ]              # executable
[ -s "/file" ]                   # tồn tại và size > 0
[ -L "/symlink" ]                # là symbolic link
[ -z "$VAR" ]                    # string rỗng
[ -n "$VAR" ]                    # string không rỗng

# So sánh số
[ "$a" -eq "$b" ]                # equal
[ "$a" -ne "$b" ]                # not equal
[ "$a" -lt "$b" ]                # less than
[ "$a" -gt "$b" ]                # greater than
[ "$a" -le "$b" ]                # less or equal
[ "$a" -ge "$b" ]                # greater or equal

# So sánh string
[ "$a" = "$b" ]                  # equal (POSIX)
[[ "$a" == "$b" ]]               # equal (bash)
[[ "$a" != "$b" ]]               # not equal
[[ "$a" < "$b" ]]                # lexicographic less than
[[ "$str" == *pattern* ]]        # glob matching (chỉ [[ ]])
[[ "$str" =~ ^[0-9]+$ ]]         # regex matching (chỉ [[ ]])

# Logical operators
[ condition1 ] && [ condition2 ] # AND
[ condition1 ] || [ condition2 ] # OR
! [ condition ]                   # NOT
[[ condition1 && condition2 ]]    # AND trong double bracket
[[ condition1 || condition2 ]]    # OR trong double bracket

# Ví dụ thực tế
check_prerequisites() {
    local errors=0
    
    if ! command -v node &>/dev/null; then
        log_error "Node.js is not installed"
        ((errors++))
    fi
    
    if ! command -v psql &>/dev/null; then
        log_error "PostgreSQL client is not installed"
        ((errors++))
    fi
    
    if [[ ! -f "/app/shoplite/shared/.env" ]]; then
        log_error ".env file not found"
        ((errors++))
    fi
    
    if [[ "$errors" -gt 0 ]]; then
        log_error "Prerequisites check failed with $errors error(s)"
        return 1
    fi
    
    log_info "All prerequisites OK"
}
```

#### 8.4 Vòng lặp

```bash
# for loop — range
for i in {1..10}; do
    echo "Item $i"
done

for i in {0..100..10}; do       # bước nhảy 10
    echo "$i"
done

# for loop — C style
for ((i=0; i<10; i++)); do
    echo "$i"
done

# for loop — iterate over items
for service in nginx postgresql redis shoplite; do
    if systemctl is-active --quiet "$service"; then
        echo "$service: running"
    else
        echo "$service: STOPPED"
    fi
done

# for loop — files
for file in /var/log/*.log; do
    if [[ -f "$file" ]]; then
        size=$(du -sh "$file" | cut -f1)
        echo "$size  $file"
    fi
done

# for loop — command output
for user in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    echo "Regular user: $user"
done

# while loop
while [ condition ]; do
    # ...
done

# Đọc file từng dòng
while IFS= read -r line; do
    echo "Processing: $line"
done < /etc/hosts

# Chờ service sẵn sàng
max_attempts=30
attempt=0
while ! pg_isready -h localhost -p 5432 &>/dev/null; do
    ((attempt++))
    if [[ "$attempt" -ge "$max_attempts" ]]; then
        log_error "PostgreSQL not ready after $max_attempts attempts"
        exit 1
    fi
    log_info "Waiting for PostgreSQL... attempt $attempt/$max_attempts"
    sleep 2
done
log_info "PostgreSQL is ready"

# until loop
until pg_isready -h localhost; do
    echo "Waiting..."
    sleep 1
done

# break và continue
for i in {1..10}; do
    [[ "$i" -eq 5 ]] && break    # thoát vòng lặp
    [[ "$i" -eq 3 ]] && continue # bỏ qua iteration này
    echo "$i"
done
```

#### 8.5 Functions

```bash
# Khai báo function
function_name() {
    local param1="$1"           # local: biến local, không ảnh hưởng ra ngoài
    local param2="${2:-default}" # với default value
    
    # logic
    echo "Result"               # return string qua stdout
    return 0                    # return code (0=success, 1+=error)
}

# Gọi function
result=$(function_name arg1 arg2)   # capture output
function_name arg1 arg2             # chỉ chạy, không capture

# Ví dụ thực tế — check disk usage
check_disk() {
    local path="${1:-/}"
    local threshold="${2:-80}"
    
    local usage
    usage=$(df "$path" | awk 'NR==2 {print $5}' | tr -d '%')
    
    if [[ "$usage" -gt "$threshold" ]]; then
        log_warn "Disk usage on $path is ${usage}% (threshold: ${threshold}%)"
        return 1
    else
        log_info "Disk usage on $path is ${usage}% (OK)"
        return 0
    fi
}

# Ví dụ — check RAM
check_ram() {
    local threshold="${1:-90}"
    
    local total used usage
    total=$(free | awk '/^Mem:/ {print $2}')
    used=$(free | awk '/^Mem:/ {print $3}')
    usage=$(( used * 100 / total ))
    
    if [[ "$usage" -gt "$threshold" ]]; then
        log_warn "RAM usage is ${usage}% (threshold: ${threshold}%)"
        return 1
    fi
    log_info "RAM usage is ${usage}% (OK)"
}

# Ví dụ — check CPU load
check_cpu_load() {
    local threshold="${1:-2.0}"
    local ncpu
    ncpu=$(nproc)
    
    local load
    load=$(uptime | awk -F'load average:' '{print $2}' | awk -F, '{print $1}' | tr -d ' ')
    
    if (( $(echo "$load > $threshold" | bc -l) )); then
        log_warn "CPU load is $load (threshold: $threshold, CPUs: $ncpu)"
        return 1
    fi
    log_info "CPU load is $load (OK)"
}
```

#### 8.6 Arrays

```bash
# Indexed arrays
arr=("apple" "banana" "cherry")
arr[3]="date"                   # thêm phần tử
arr+=("elderberry")             # append

echo "${arr[0]}"                # truy cập phần tử (index từ 0)
echo "${arr[@]}"                # tất cả phần tử
echo "${arr[*]}"                # tất cả (khác nhau khi quote)
echo "${#arr[@]}"               # số phần tử
echo "${!arr[@]}"               # tất cả indices
unset arr[2]                    # xóa phần tử

# Iterate
for item in "${arr[@]}"; do
    echo "$item"
done

for i in "${!arr[@]}"; do
    echo "[$i] = ${arr[$i]}"
done

# Associative arrays (bash 4+)
declare -A config
config["host"]="localhost"
config["port"]="5432"
config["dbname"]="shoplite"

echo "${config[host]}"
for key in "${!config[@]}"; do
    echo "$key = ${config[$key]}"
done

# Ví dụ thực tế
declare -A SERVICE_PORTS
SERVICE_PORTS=(
    ["nginx"]="80 443"
    ["postgresql"]="5432"
    ["redis"]="6379"
    ["shoplite"]="3000"
)

for service in "${!SERVICE_PORTS[@]}"; do
    for port in ${SERVICE_PORTS[$service]}; do
        if ss -tlnp | grep -q ":$port "; then
            echo "$service (port $port): LISTENING"
        else
            echo "$service (port $port): NOT LISTENING"
        fi
    done
done
```

#### 8.7 Script kiểm tra system resources (production-ready)

```bash
#!/bin/bash
# /app/shoplite/scripts/health-check.sh
# Mục đích: Kiểm tra sức khỏe server và gửi cảnh báo

set -euo pipefail

# ===== CONFIGURATION =====
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/shoplite/health-check.log"
readonly LOCK_FILE="/tmp/shoplite-health-check.lock"
readonly ALERT_EMAIL="${ALERT_EMAIL:-}"  # set qua env var

# Ngưỡng cảnh báo (%)
readonly DISK_THRESHOLD="${DISK_THRESHOLD:-80}"
readonly RAM_THRESHOLD="${RAM_THRESHOLD:-85}"
readonly CPU_THRESHOLD="${CPU_THRESHOLD:-2.0}"
readonly ALERT_COOLDOWN=3600  # giây — không alert lại trong 1 giờ

# ===== FUNCTIONS =====
log() {
    local level="$1"; shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}
log_info()  { log "INFO " "$@"; }
log_warn()  { log "WARN " "$@"; }
log_error() { log "ERROR" "$@" >&2; }

# Lock để tránh chạy 2 instance cùng lúc
acquire_lock() {
    if [[ -f "$LOCK_FILE" ]]; then
        local lock_pid
        lock_pid=$(cat "$LOCK_FILE" 2>/dev/null || echo "0")
        if kill -0 "$lock_pid" 2>/dev/null; then
            log_error "Another instance is running (PID: $lock_pid)"
            exit 1
        fi
        log_warn "Removing stale lock file"
        rm -f "$LOCK_FILE"
    fi
    echo "$$" > "$LOCK_FILE"
}

cleanup() {
    rm -f "$LOCK_FILE"
}
trap cleanup EXIT

send_alert() {
    local subject="$1"
    local message="$2"
    local alert_flag="/tmp/shoplite-alert-$(echo "$subject" | md5sum | cut -c1-8).flag"
    
    # Cooldown — không alert liên tục
    if [[ -f "$alert_flag" ]]; then
        local last_alert
        last_alert=$(stat -c %Y "$alert_flag" 2>/dev/null || echo 0)
        local now
        now=$(date +%s)
        if (( now - last_alert < ALERT_COOLDOWN )); then
            log_info "Alert cooldown active for: $subject"
            return 0
        fi
    fi
    
    touch "$alert_flag"
    log_warn "ALERT: $subject"
    log_warn "       $message"
    
    if [[ -n "$ALERT_EMAIL" ]]; then
        echo "$message" | mail -s "[ShopLite Alert] $subject" "$ALERT_EMAIL" 2>/dev/null || true
    fi
}

check_disk() {
    log_info "Checking disk usage..."
    local failed=0
    
    while IFS= read -r line; do
        local usage mount
        usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
        mount=$(echo "$line" | awk '{print $6}')
        
        if [[ "$usage" -ge "$DISK_THRESHOLD" ]]; then
            send_alert "High Disk Usage: $mount" \
                "Disk usage on $mount is ${usage}% (threshold: ${DISK_THRESHOLD}%)"
            ((failed++))
        else
            log_info "  $mount: ${usage}% (OK)"
        fi
    done < <(df -h | awk 'NR>1 && $5+0 > 0 {print}')
    
    return "$failed"
}

check_ram() {
    log_info "Checking RAM usage..."
    
    local total used available usage
    total=$(free -m | awk '/^Mem:/ {print $2}')
    used=$(free -m | awk '/^Mem:/ {print $3}')
    available=$(free -m | awk '/^Mem:/ {print $7}')
    usage=$(( used * 100 / total ))
    
    log_info "  RAM: ${used}MB used / ${total}MB total (${usage}%), ${available}MB available"
    
    if [[ "$usage" -ge "$RAM_THRESHOLD" ]]; then
        send_alert "High RAM Usage" \
            "RAM usage is ${usage}% (${used}MB/${total}MB, threshold: ${RAM_THRESHOLD}%)"
        return 1
    fi
}

check_cpu_load() {
    log_info "Checking CPU load..."
    
    local ncpu load_1m load_5m load_15m
    ncpu=$(nproc)
    load_1m=$(uptime | awk -F'load average:' '{print $2}' | awk -F, '{print $1}' | xargs)
    load_5m=$(uptime | awk -F'load average:' '{print $2}' | awk -F, '{print $2}' | xargs)
    load_15m=$(uptime | awk -F'load average:' '{print $2}' | awk -F, '{print $3}' | xargs)
    
    log_info "  Load average: ${load_1m} (1m) ${load_5m} (5m) ${load_15m} (15m) | CPUs: $ncpu"
    
    if (( $(echo "$load_1m > $CPU_THRESHOLD" | bc -l) )); then
        send_alert "High CPU Load" \
            "CPU load average (1m): $load_1m (threshold: $CPU_THRESHOLD, CPUs: $ncpu)"
        return 1
    fi
}

check_services() {
    log_info "Checking services..."
    local failed=0
    local services=("nginx" "postgresql" "redis-server" "shoplite")
    
    for service in "${services[@]}"; do
        if systemctl is-active --quiet "$service" 2>/dev/null; then
            log_info "  $service: active"
        else
            log_warn "  $service: INACTIVE"
            send_alert "Service Down: $service" \
                "Service $service is not running on $(hostname)"
            ((failed++))
        fi
    done
    
    return "$failed"
}

# ===== MAIN =====
main() {
    mkdir -p "$(dirname "$LOG_FILE")"
    acquire_lock
    
    log_info "=== Health Check Started ==="
    log_info "Hostname: $(hostname)"
    log_info "Uptime: $(uptime -p)"
    
    local overall_status=0
    
    check_disk  || overall_status=1
    check_ram   || overall_status=1
    check_cpu_load || overall_status=1
    check_services || overall_status=1
    
    if [[ "$overall_status" -eq 0 ]]; then
        log_info "=== Health Check PASSED ==="
    else
        log_warn "=== Health Check FAILED — see warnings above ==="
    fi
    
    return "$overall_status"
}

main "$@"
```

---

### 9. Cron Jobs

#### 9.1 Cú pháp cron

```
# Cú pháp: 5 trường + command
# ┌───────── phút (0-59)
# │ ┌───────── giờ (0-23)
# │ │ ┌───────── ngày trong tháng (1-31)
# │ │ │ ┌───────── tháng (1-12)
# │ │ │ │ ┌───────── thứ trong tuần (0-7, 0 và 7 đều là Chủ nhật)
# │ │ │ │ │
# * * * * * command

# Ký hiệu đặc biệt:
# *   bất kỳ giá trị nào
# ,   liệt kê nhiều giá trị: 1,15,30
# -   khoảng: 1-5 (thứ 2 đến thứ 6)
# /   bước nhảy: */5 (mỗi 5 đơn vị)
```

```bash
# Ví dụ cú pháp
* * * * *           # mỗi phút
0 * * * *           # đầu mỗi giờ
0 2 * * *           # 2:00 AM mỗi ngày
0 2 * * 0           # 2:00 AM mỗi Chủ nhật
0 2 1 * *           # 2:00 AM ngày 1 mỗi tháng
0 2 1 1 *           # 2:00 AM ngày 1/1 mỗi năm
*/5 * * * *         # mỗi 5 phút
0 */6 * * *         # mỗi 6 giờ (0:00, 6:00, 12:00, 18:00)
0 2 * * 1-5         # 2:00 AM thứ 2 đến thứ 6 (weekdays)
30 8,12,17 * * *    # 8:30, 12:30, 17:30 mỗi ngày
0 0 1,15 * *        # midnight ngày 1 và 15 mỗi tháng

# Aliases (ít portable hơn nhưng dễ đọc)
@reboot             # chạy khi system boot
@yearly             # = 0 0 1 1 *
@monthly            # = 0 0 1 * *
@weekly             # = 0 0 * * 0
@daily              # = 0 0 * * *
@hourly             # = 0 * * * *
```

#### 9.2 Quản lý crontab

```bash
# User crontab
crontab -e                      # mở editor để sửa
crontab -l                      # xem crontab hiện tại
crontab -r                      # XÓA toàn bộ crontab (NGUY HIỂM — không hỏi lại)
crontab -l > ~/crontab.bak      # backup trước khi sửa
crontab -u username -l          # xem crontab của user khác (root only)
crontab -u username -e          # sửa crontab của user khác

# System-wide cron
# /etc/crontab — system crontab (có thêm cột USER)
# /etc/cron.d/ — cron files của packages
# /etc/cron.daily/ — scripts chạy hàng ngày (do run-parts)
# /etc/cron.weekly/ — scripts chạy hàng tuần
# /etc/cron.monthly/ — scripts chạy hàng tháng
# /etc/cron.hourly/ — scripts chạy hàng giờ

# Format /etc/crontab và /etc/cron.d/ (có cột user):
# * * * * * username command
```

#### 9.3 Cron jobs thực tế cho ShopLite

```bash
# /etc/cron.d/shoplite
# Chú ý: không có comments ngay sau MAILTO/các biến

SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=""

# Health check mỗi 5 phút
*/5 * * * * shoplite /app/shoplite/scripts/health-check.sh >> /var/log/shoplite/health-check.log 2>&1

# Backup database lúc 2:00 AM hàng ngày
0 2 * * * shoplite /app/shoplite/scripts/backup-db.sh >> /var/log/shoplite/backup.log 2>&1

# Xóa session cũ lúc 3:00 AM hàng ngày
0 3 * * * shoplite /app/shoplite/scripts/cleanup-sessions.sh >> /var/log/shoplite/cleanup.log 2>&1

# Rotate logs mỗi Chủ nhật lúc 1:00 AM
0 1 * * 0 shoplite /app/shoplite/scripts/rotate-logs.sh >> /var/log/shoplite/rotate.log 2>&1

# Report tổng hợp hàng tuần vào thứ 2 lúc 9:00 AM
0 9 * * 1 shoplite /app/shoplite/scripts/weekly-report.sh 2>&1 | mail -s "ShopLite Weekly Report" admin@example.com

# User crontab của user shoplite (crontab -e -u shoplite)
# Tạm thời tắt một job bằng cách thêm # ở đầu
```

#### 9.4 Script backup cho cron

```bash
#!/bin/bash
# /app/shoplite/scripts/backup-db.sh
set -euo pipefail

readonly BACKUP_DIR="/app/shoplite/shared/backups"
readonly DB_NAME="${DB_NAME:-shoplite}"
readonly DB_USER="${DB_USER:-shoplite}"
readonly RETENTION_DAYS="${RETENTION_DAYS:-7}"
readonly TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
readonly BACKUP_FILE="${BACKUP_DIR}/db_${DB_NAME}_${TIMESTAMP}.sql.gz"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }

# Tạo backup
log "Starting database backup: $BACKUP_FILE"
mkdir -p "$BACKUP_DIR"

pg_dump -U "$DB_USER" "$DB_NAME" | gzip > "$BACKUP_FILE"

# Kiểm tra backup thành công
if [[ -f "$BACKUP_FILE" ]] && [[ -s "$BACKUP_FILE" ]]; then
    local_size=$(du -sh "$BACKUP_FILE" | cut -f1)
    log "Backup successful: $BACKUP_FILE ($local_size)"
else
    log "ERROR: Backup file is empty or missing!"
    exit 1
fi

# Xóa backup cũ hơn RETENTION_DAYS ngày
log "Cleaning up backups older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -name "db_${DB_NAME}_*.sql.gz" \
    -mtime "+${RETENTION_DAYS}" -delete -print | while read -r f; do
    log "  Deleted: $f"
done

# Thống kê
backup_count=$(find "$BACKUP_DIR" -name "db_${DB_NAME}_*.sql.gz" | wc -l)
log "Current backup count: $backup_count"
log "Backup completed successfully"
```

---

### 10. Bộ 40+ Lệnh Quan Trọng

#### 10.1 System information

```bash
# Thông tin hệ thống
uname -a                        # kernel version, arch, hostname
uname -r                        # chỉ kernel version (dùng để check module path)
hostname                        # tên máy chủ
hostname -f                     # FQDN (fully qualified domain name)
hostnamectl                     # chi tiết + đổi hostname
hostnamectl set-hostname server01

uptime                          # thời gian chạy, users, load average
uptime -p                       # pretty format: "up 3 days, 4 hours"
uptime -s                       # thời điểm boot: "2024-01-15 08:00:00"

date                            # ngày giờ hiện tại
date '+%Y-%m-%d %H:%M:%S'       # format cụ thể
date -d "yesterday"             # ngày hôm qua
date -d "next monday"           # thứ 2 tới
date +%s                        # Unix timestamp

timedatectl                     # timezone, NTP status
timedatectl list-timezones | grep Asia
timedatectl set-timezone Asia/Ho_Chi_Minh
timedatectl set-ntp true        # bật NTP sync

# Hardware
lscpu                           # thông tin CPU chi tiết
lsmem                           # thông tin memory
lsblk                           # block devices (disks, partitions)
lsblk -f                        # thêm filesystem info
lspci                           # PCI devices (GPU, NIC, etc.)
lsusb                           # USB devices
lshw                            # full hardware list (cài: apt install lshw)
lshw -short                     # tóm tắt
dmidecode -t bios               # BIOS info (cần root)
dmidecode -t memory             # RAM slots info
```

#### 10.2 Disk và storage

```bash
# Disk space
df -h                           # disk usage, human-readable
df -hT                          # thêm filesystem type
df -h /app                      # chỉ filesystem chứa /app
df -i                           # inode usage (quan trọng khi "disk full" nhưng df -h OK)

# Directory size
du -sh /var/log                 # size của thư mục
du -sh /var/log/*               # size của từng sub-item
du -h --max-depth=1 /var        # depth 1
du -h --max-depth=2 / 2>/dev/null | sort -rh | head -20  # top 20 largest

# Block devices
lsblk                           # cây thiết bị
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
fdisk -l                        # partition table (cần root)
blkid                           # UUID và filesystem type của partitions
mount | column -t               # mounted filesystems

# Find large files
find / -type f -size +100M 2>/dev/null | head -20
find /var/log -type f -name "*.log" -size +50M

# inode check
ls -i /path/to/file             # xem inode number
stat filename                   # đầy đủ: size, inode, permissions, timestamps
```

#### 10.3 Network

```bash
# Network interfaces
ip addr                         # IP addresses (thay thế ifconfig)
ip addr show eth0               # chỉ interface eth0
ip link                         # link layer info
ip link set eth0 up             # bật interface
ip link set eth0 down           # tắt interface

# Routing
ip route                        # routing table
ip route show                   # tương tự
ip route get 8.8.8.8            # route đến IP cụ thể

# Connections và ports
ss -tulpn                       # tất cả listening sockets (thay thế netstat)
# t=TCP, u=UDP, l=listening, p=process, n=numeric (không resolve)
ss -tulpn | grep :80            # port 80 đang listen không
ss -tnp                         # established TCP connections
ss -s                           # statistics summary
netstat -tulpn                  # tương tự ss (cài: apt install net-tools)

# DNS
nslookup example.com
dig example.com                 # DNS lookup chi tiết
dig +short example.com          # chỉ IP
dig -x 8.8.8.8                  # reverse lookup
host example.com

# Connectivity
ping -c 4 8.8.8.8               # 4 packets
ping -i 0.2 8.8.8.8             # interval 0.2 giây
traceroute 8.8.8.8
mtr 8.8.8.8                     # traceroute + ping (interactive)
curl -I https://example.com     # HTTP headers only
curl -v https://example.com     # verbose (xem TLS handshake)
curl -o /dev/null -w "%{http_code}" https://example.com  # chỉ status code
wget -q --spider https://example.com  # kiểm tra URL không download

# Network statistics
iftop                           # bandwidth per connection (cài apt install iftop)
nload                           # bandwidth tổng
nethogs                         # bandwidth per process
```

#### 10.4 File operations nâng cao

```bash
# find — tìm file
find /path -name "*.log"                    # tìm theo tên
find /path -name "*.log" -type f            # chỉ files
find /path -type d                          # chỉ directories
find /path -user shoplite                  # thuộc về user
find /path -group www-data                 # thuộc về group
find /path -mtime +7                        # sửa đổi hơn 7 ngày trước
find /path -mtime -1                        # sửa đổi trong 1 ngày qua
find /path -size +100M                      # lớn hơn 100MB
find /path -perm 644                        # permission chính xác 644
find /path -perm -u+x                       # có executable bit cho owner
find /path -name "*.log" -mtime +7 -delete  # tìm và xóa

# find với exec
find /var/log -name "*.log" -exec ls -lh {} \;     # chạy ls cho từng file
find /tmp -name "*.tmp" -exec rm {} \;              # xóa từng file
find /app -name "*.js" -exec grep -l "TODO" {} \;  # find TODO trong .js files

# locate — tìm nhanh (cần updatedb)
locate filename
updatedb                        # cập nhật database
locate -i filename              # case-insensitive

# which và type
which node                      # đường dẫn executable
which -a python                 # tất cả occurrences trong PATH
type ls                         # xem ls là alias, builtin, hay binary
type -a ls                      # tất cả

# File content
file filename                   # xác định loại file (không dựa vào extension)
file /bin/bash                  # ELF 64-bit LSB pie executable
xxd file | head -4              # hex dump (xem magic bytes)

# Archive
tar -czvf archive.tar.gz /path/to/dir       # c=create, z=gzip, v=verbose, f=file
tar -cjvf archive.tar.bz2 /path/to/dir      # bzip2 (nhỏ hơn, chậm hơn)
tar -cJvf archive.tar.xz /path/to/dir       # xz (nhỏ nhất, chậm nhất)
tar -xzvf archive.tar.gz                     # x=extract
tar -xzvf archive.tar.gz -C /tmp/           # extract vào /tmp/
tar -tzvf archive.tar.gz                     # t=list (xem nội dung không extract)
tar -xzvf archive.tar.gz file.txt           # extract file cụ thể

zip -r archive.zip /path/                   # tạo zip
zip -r -e archive.zip /path/               # có password
unzip archive.zip                           # giải nén
unzip -l archive.zip                        # list nội dung
unzip archive.zip -d /tmp/                  # extract vào /tmp/
```

#### 10.5 Process monitoring nâng cao

```bash
# lsof — list open files
lsof -p 1234                    # files đang mở bởi PID 1234
lsof -u shoplite               # files đang mở bởi user
lsof -i :80                    # process dùng port 80
lsof -i TCP                    # tất cả TCP connections
lsof /var/log/app.log          # process nào đang mở file này
lsof +D /tmp/                  # files trong thư mục

# strace — trace system calls (debug)
strace -p 1234                  # attach vào process đang chạy
strace -p 1234 -e read,write    # chỉ trace read/write syscalls
strace -p 1234 -o strace.log    # ghi vào file
strace command                  # trace toàn bộ execution
strace -c command               # summary (đếm syscalls)

# Ứng dụng strace: debug tại sao app không đọc được file, port bị conflict...

# vmstat — virtual memory stats
vmstat                          # snapshot
vmstat 2 10                     # mỗi 2 giây, 10 lần
vmstat -s                       # summary statistics

# iostat — I/O statistics
iostat                          # CPU và disk I/O
iostat -x                       # extended disk stats
iostat 2                        # update mỗi 2 giây

# sar — system activity reporter
sar -u 1 5                      # CPU usage mỗi 1 giây, 5 lần
sar -r 1 5                      # RAM usage
sar -d 1 5                      # disk I/O
```

#### 10.6 Misc — Tiện ích hàng ngày

```bash
# alias — tạo shortcut
alias ll='ls -lah'
alias gs='git status'
alias ports='ss -tulpn'
alias myip='curl -s https://api.ipify.org'
alias df='df -h'
# Đặt trong ~/.bashrc để persistent

alias                           # xem tất cả alias
unalias ll                      # xóa alias

# history
history                         # xem lịch sử lệnh
history | tail -20              # 20 lệnh cuối
history | grep nginx            # tìm lệnh liên quan nginx
!!                              # chạy lại lệnh trước
!42                             # chạy lệnh số 42 trong history
!git                            # chạy lại lệnh git gần nhất
Ctrl+R                          # search history (incremental reverse search)
HISTSIZE=10000                  # số lệnh lưu trong memory
HISTFILESIZE=20000              # số lệnh lưu trong file ~/.bash_history
HISTCONTROL=ignoredups          # không lưu dòng duplicate liên tiếp

# Environment variables
env                             # xem tất cả env vars
export VAR="value"              # set và export (subprocesses thấy được)
VAR="value"                     # set nhưng không export (chỉ shell hiện tại)
printenv VAR                    # xem giá trị
unset VAR                       # xóa biến
source ~/.bashrc                # reload .bashrc (áp dụng thay đổi)
. ~/.bashrc                     # shorthand cho source

# Screen management
clear                           # xóa màn hình
reset                           # reset terminal (khi terminal bị lỗi)
Ctrl+L                          # clear (giống clear command)
tput cols                       # số cột terminal
tput lines                      # số dòng terminal

# Miscellaneous
cal                             # calendar
cal 2024                        # calendar cả năm
sleep 5                         # chờ 5 giây
time command                    # đo thời gian chạy lệnh
watch -n 2 "df -h"              # chạy lệnh mỗi 2 giây (như dashboard)
watch -d "ss -tulpn"            # highlight thay đổi
xargs                           # build và execute commands từ stdin
echo "file1 file2" | xargs rm  # xóa file1 và file2
find . -name "*.tmp" | xargs rm -f  # xóa tất cả .tmp files

# Xem file lớn
less file.txt                   # scroll, tìm kiếm
less +F file.txt                # như tail -f nhưng có thể scroll lên
less +G file.txt                # mở ở cuối file
less +/pattern file.txt         # mở và highlight pattern
# Trong less: / search, n next, N prev, q quit, G end, g start

# nl — number lines
nl file.txt                     # đánh số dòng
nl -ba file.txt                 # đánh số cả dòng trống

# tac — in file theo thứ tự ngược
tac file.txt                    # dòng cuối lên đầu

# rev — đảo ngược từng dòng
echo "hello" | rev              # "olleh"

# Column formatting
column -t file.tsv              # căn lề columns
ls -la | column -t

# bc — calculator
echo "2^32" | bc
echo "scale=4; 355/113" | bc    # pi approximation

# String tools
echo -n "hello" | md5sum        # MD5 hash
echo -n "hello" | sha256sum     # SHA256 hash
base64 file.txt                 # encode
base64 -d encoded.txt           # decode

# Processes và system
nproc                           # số CPU cores
free -h                         # RAM usage, human-readable
free -s 2                       # update mỗi 2 giây
sync                            # flush filesystem buffers
dmesg                           # kernel ring buffer (boot messages, hardware errors)
dmesg -T                        # với timestamp
dmesg | tail -20                # 20 dòng cuối
dmesg | grep -i "error\|warning\|fail"   # lọc lỗi
```

---

### 11. Tổng Kết Quick Reference

#### Cheat Sheet — Lệnh thường dùng nhất mỗi ngày

```bash
# === ĐIỀU HƯỚNG ===
pwd; ls -la; cd /path; cd -; cd ~

# === FILE ===
cp -r src dst; mv src dst; rm -rf dir; mkdir -p a/b/c
touch file; ln -s target link; find . -name "*.log" -mtime +7

# === QUYỀN ===
chmod 755 script.sh; chmod -R 750 /app; chown user:group file; chown -R u:g dir

# === PROCESS ===
ps aux | grep app; top; htop; kill -15 PID; kill -9 PID
pgrep -f pattern; pkill -f pattern; nohup cmd &

# === SERVICE ===
systemctl status nginx; systemctl restart nginx; systemctl enable nginx
journalctl -u nginx -f; journalctl -u nginx -n 100 --no-pager

# === PACKAGE ===
apt update && apt upgrade -y; apt install -y pkg; apt remove pkg; dpkg -l | grep pkg

# === LOG ANALYSIS ===
tail -f /var/log/app.log
grep -E "error|warn" /var/log/app.log | tail -50
cat access.log | awk '{print $9}' | sort | uniq -c | sort -rn | head -10

# === DISK ===
df -h; du -sh /var/log/*; du -h --max-depth=1 /var | sort -rh | head -10
find / -type f -size +100M 2>/dev/null | head -10

# === NETWORK ===
ip addr; ss -tulpn; ss -tulpn | grep :80; curl -I https://example.com

# === CRON ===
crontab -e; crontab -l; cat /etc/cron.d/
```

---

## Kỹ năng đạt được
- Quản trị Linux server cơ bản qua dòng lệnh.
- Đọc và lọc log bằng grep/awk.
- Viết script tự động hóa tác vụ và lập lịch bằng cron.

---

## Thực hành

**Môi trường:** Ubuntu Server (LTS) trên VM hoặc WSL2.

**Lab:**
- Cài đặt Ubuntu Server, cấu hình hostname, timezone.
- Cài các runtime mà ShopLite cần (Node.js / Python, PostgreSQL) **bằng tay** để hiểu phụ thuộc.
- Tạo user riêng cho ứng dụng, phân quyền thư mục.
- Viết script backup file log và lập lịch chạy bằng cron.

**Công cụ:** Ubuntu Server, systemd, cron, bash.

---

## Bài tập

- **Bắt buộc:** Viết script kiểm tra dung lượng đĩa, RAM, và in cảnh báo nếu vượt ngưỡng.
- **Nâng cao:** Script tự động tạo user mới với quyền chuẩn và thư mục home cấu hình sẵn.
- **Mô phỏng doanh nghiệp:** Viết "runbook" 1 trang hướng dẫn khởi động lại dịch vụ ShopLite chạy bằng systemd khi nó sập.

---

## Deliverable
- [ ] Một Ubuntu Server vận hành được, có user ứng dụng riêng.
- [ ] ShopLite chạy được thủ công trên server này (chưa container hóa).
- [ ] Thư mục `scripts/` chứa các bash script tiện ích.

---

## Tiêu chí hoàn thành
- [ ] Làm chủ ≥ 40 lệnh Linux thông dụng mà không cần tra cứu.
- [ ] Tự cài và chạy được ShopLite thủ công trên Linux.
- [ ] Có ít nhất 2 bash script hoạt động + 1 cron job.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
