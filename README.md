## 前置き
このBorn2beRootという課題では仮想マシン上でサーバーを構築します。　　
この課題を通してITインフラの基礎を学びましょう！

## 0. VirtualBox ホスト側設定と事前準備

| 設定項目 | 内容 |
|---|---|
| **表示言語** | VirtualBox を日本語表示に変更 |
| **ホストキー** | 「左 Ctrl + 左 Win（⌘）」へ変更 |
| **既定 VM フォルダ** | `/sgoinfre` |
| **ISO 取得** | Debian 公式サイトから `debian-<version>-amd64-netinst.iso` をダウンロード |

---

## 1. 仮想マシン新規作成

### 1.1 基本情報  
1. **名前**: 任意（初使用の名称）  
2. **保存場所**: `/sgoinfre`  
3. **ISO**: `debian-…netinst.iso` を指定  
4. **自動インストール**: *オフ*（チェックを外す）

### 1.2 ハードウェア  
| 項目 | 推奨値 |
|---|---|
| メモリ / CPU | デフォルト（後で変更可） |
| 仮想 HDD | 30 GB (VDI・可変) |

> 作成後すぐ **起動** → 画面上部 **表示 › スケールモード** へ切替

---

## 2. Debian 12 インストール

### 2.1 インストーラ設定 (BIOS mode)
| 画面 | 選択値 |
|---|---|
| Installer Menu | **Install** |
| Language | English |
| Country | United States |
| Keymap | American English |
| Hostname | `ikawamuk42` |
| Domain | （空欄） |
| Root PW / User | それぞれ設定 |
| Time zone | Eastern |

### 2.2 パーティション（Manual）

1. **ディスク**: `VBOX HARDDISK` → *Yes*  
2. **基本領域**  
   - `sda1`  … `/boot`  
3. **暗号化 + LVM**  
   - `sda5` を *Use as: physical volume for encryption*  
   - **Configure LVM** ⇒ *Yes*  
   - **VG 名**: `LVMGroup`  
   - **Logical Volume (7 個例)**  
     | 名前 | マウントポイント | サイズ |
     |---|---|---|
     | root | / | 10 GiB |
     | … | … | … |
4. **暗号化ボリューム**: *Yes*（全 LV を選択）  
5. **ファイルシステム**: ext4（`swap` は swap area）  
6. **ブートローダ**: GRUB を `/dev/sda`（MBR）へ  
7. 追加メディア: *No*  
8. ミラー: Country = US, Mirror = `deb.debian.org`, Proxy = 空  
9. パッケージ選択: *全て外す*  
10. 再起動確認: **Continue**

---

## 3. 初回起動後の確認

```bash
# root でログイン
dpkg -l                   # インストール済み一覧
dpkg -s <pkg>             # 個別パッケージ情報
apt update && apt upgrade # システム更新
systemctl status apparmor # AppArmor 起動確認
timedatectl set-timezone Asia/Tokyo
date                      # 日本時刻へ変更済みか確認
```

---

## 4. 基本パッケージ導入

### 4.1 UFW

```bash
apt install ufw
ufw status verbose        # 状態確認
ufw enable                # 有効化
```

### 4.2 sudo

```bash
apt install sudo
usermod -aG sudo ikawamuk
groups ikawamuk           # sudo 参加確認
visudo -f /etc/sudoers.d/b2b_rules
```

`/etc/sudoers.d/b2b_rules` 例:

```
Defaults passwd_tries=3
Defaults badpass_message="That's unique password! but it's still wrong..."
Defaults log_input,log_output
Defaults iolog_dir="/var/log/sudo"
Defaults requiretty
Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

### 4.3 OpenSSH Server

```bash
apt install openssh-server
systemctl status sshd      # active (running)
```

---

## 5. リモートログイン構成

### 5.1 ホストオンリーネットワーク

1. VM をシャットダウン  
2. VirtualBox › ツール › ネットワークでホストオンリーアダプタ作成  
3. VM 設定 › ネットワーク › **アダプタ 2** = ホストオンリーアダプタ

### 5.2 ゲスト側静的 IP

`/etc/network/interfaces` 追記:

```
# Host-only interface
allow-hotplug enp0s8
iface enp0s8 inet static
    address 192.168.60.2
    netmask 255.255.255.0
```

### 5.3 SSH 設定変更

`/etc/ssh/sshd_config`:

```
Port 4242
PermitRootLogin no
```

再読み込み:

```bash
systemctl restart sshd
```

クライアント側 `~/.ssh/config`:

```
Host ikawamuk42
    HostName 192.168.60.2
    User ikawamuk
    Port 4242
```

### 5.4 UFW ポート開放

```bash
ufw allow 4242/tcp
ufw status verbose
```

---

## 6. モニタリングスクリプト

### 6.1 本体 `/usr/local/bin/monitoring_sh`

```bash
#!/bin/bash
until who | grep -q .; do sleep 1; done

BANNER=$(figlet "System Monitor")
ARCH=$(uname -a)
CPU_PHYS=$(lscpu | awk '/Socket\(s\)/{print $2}')
VCPU=$(nproc)
MEM_T=$(free -m | awk '/^Mem:/{print $2}')
MEM_U=$(free -m | awk '/^Mem:/{print $3}')
MEM_P=$(awk "BEGIN{printf \"%.2f\",$MEM_U/$MEM_T*100}")
DISK_U=$(df -Bm --total | awk '/^\/dev/{u+=$3}END{print u}')
DISK_T=$(df -Bm --total | awk '/^\/dev/{t+=$2}END{print t}')
DISK_P=$(awk "BEGIN{printf \"%.0f\",$DISK_U/$DISK_T*100}")
CPU_L=$(top -bn1 | awk '/load average/{sub(",","",$10);print $10}')
LAST_B=$(who -b | awk '{print $3,$4,$5}')
LVM=$(lsblk | grep -q lvm && echo yes || echo no)
TCP=$(ss -t | grep ESTAB | wc -l)
USR=$(who | cut -d' ' -f1 | sort -u | wc -l)
IP=$(hostname -I | awk '{print $1}')
MAC=$(ip link show | awk '/ether/{print $2;exit}')
SUDO=$(journalctl _COMM=sudo | grep COMMAND | wc -l)

wall <<EOF
$BANNER
#Architecture: $ARCH
#CPU physical: $CPU_PHYS
#vCPU: $VCPU
#Memory Usage: ${MEM_U}/${MEM_T}MB (${MEM_P}%)
#Disk Usage: ${DISK_U}/${DISK_T}MB (${DISK_P}%)
#CPU load: $CPU_L%
#Last boot: $LAST_B
#LVM use: $LVM
#Connections TCP: $TCP ESTABLISHED
#User log: $USR
#Network: IP $IP ($MAC)
#Sudo: $SUDO cmd
EOF
```

```bash
chmod 700 /usr/local/bin/monitoring_sh
apt install figlet
```

### 6.2 cron 例

```
@reboot  /usr/local/bin/monitoring_sh
*/10 * * * * /usr/local/bin/monitoring_sh
```

### 6.3 systemd Timer

`/etc/systemd/system/monitoring.service`

```
[Unit]
Description=Run monitoring_sh once
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/monitoring_sh
```

`/etc/systemd/system/monitoring.timer`

```
[Unit]
Description=Run monitoring_sh every 10 minutes

[Timer]
OnBootSec=0
OnUnitActiveSec=10min
Unit=monitoring.service

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now monitoring.timer
systemctl status monitoring.timer
```

---

## 7. パスワードポリシー (PAM)

```bash
apt install libpam-pwquality
```

`/etc/security/pwquality.conf` 例:

```
difok       = 7
minlen      = 10
dcredit     = -1
ucredit     = -1
lcredit     = -1
maxrepeat   = 3
enforcing   = 1
enforce_for_root
```

`/etc/pam.d/common-password` に既存行:

```
password requisite pam_pwquality.so retry=3
```

---

## 8. 追加構築

### 8.1 WordPress + lighttpd

```bash
apt install php-cgi mariadb-server lighttpd wget php-mysql
lighty-enable-mod fastcgi fastcgi-php
systemctl restart lighttpd
```

```bash
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar xf latest.tar.gz
rm -rf /var/www/html/*
mv wordpress/* /var/www/html/
chown -R www-data:www-data /var/www/html
```

MariaDB 設定:

```sql
CREATE DATABASE wp_db;
CREATE USER 'ikawamuk'@'localhost' IDENTIFIED BY 'itou';
GRANT ALL PRIVILEGES ON wp_db.* TO 'ikawamuk'@'localhost';
FLUSH PRIVILEGES;
```

ブラウザで `http://192.168.60.2` にアクセスし WordPress 初期設定を完了。

### 8.2 Git server (SSH)

```bash
apt install git
adduser git
chsh -s /usr/bin/git-shell git
mkdir -p /srv/git
chown git:git /srv/git
sudo -u git git init --bare /srv/git/myproject.git
```

クライアント操作例:

```bash
git clone ssh://git@192.168.60.2:4242/srv/git/myproject.git
```