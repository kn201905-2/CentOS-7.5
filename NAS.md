# パーティション  
* 今回（2019-10-22）は、以下のように割り振った  
（実際のところ、swap は不要であると思う。Ramdisk に大きくメモリを使用するつもりであるため、念の為 disk 上に swap 領域を設定した）  

```
/boot/efi　256m  
/　16g  
swap　4g  
/home　残り全部  
```

---
# 初期設定
* ホスト名の設定　# hostnamectl set-hostname NUC-SMB
* ネットワークデバイス名の確認　# nmcli device
* TUI を利用して、デバイスに IPアドレスを設定　# nmtui edit enp2s0  
この設定を終えた後、ネットワークケーブルを接続すると自動的に接続が実行される。
* ネットワーク接続の確認　# ping 8.8.8.8
* この時点で、SSH で root 接続できるようになる。
* SELinux の設定状況を確認　# getenforce
SELinux が解除されていたら「Disabled」が表示される。
* SELinux の無効化  
設定ファイルを書き換える　# vi /etc/selinux/config  
SELINUX=enforcing を disabled に書き換える。  
reboot して、無効化されたことを確認する　# getenforce
* 電源ボタンによってシャットダウンができる設定であることの確認  
\# vi /etc/systemd/logind.conf  
上記のファイルの設定項目で、以下を確認  
HandlePowerKey=poweroff  

（参考）yum update は、インストール済みのパッケージの更新であるため、apt 系と異なることに注意すること。  
yum update とすると、インストールされているカーネルが更新され、OSのバージョンが上がるため注意が必要である。  
必要なパッケージのみを「yum update パッケージ名」とした方がよい。

---
# 設定の補足（必要ある場合実行する）
* ネットワークデバイスが「切断済み」になっている場合、以下のコマンドを試みる。  
\# nmcli c add type ethernet ifname enp1s0 con-name enp1s0  
\# nmcli c add type wifi ifname wlp2s0 con-name wlp2s0  
\# nmcli c up enp1s0  

add することにより、以下のフォルダに設定ファイル ifcfg-enp1s0 が作成される。  
/etc/sysconfig/network-scripts  

* デバイスが自動接続になっていない場合　# nmcli c m enp1s0 connection.autoconnect "yes"  
一時的にリンクアップする場合　# nmcli c up enp1s0  

* 開いているポートを調べる場合、nmap をインストール　# yum -y install nmap  
nmap でポートスキャンを実行　# nmap 192.168.0.110
* routeコマンドなどを利用したいとき　# yum -y install net-tools
* DNSなどが利用できるようになったかどうかを調べるための nslookup 等のインストール　# yum -y install bind-utils  
yum を利用するためには、TCP 80, 443 / UDP 53 が開かれていること。

---
# grub メニューの変更（待機時間を短くする）
* 次の設定ファイルを変更　# vi /etc/default/grub
```
GRUB_TIMEOUT=1
```
* GRUB を再構築　# grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg  
起動が BIOS である場合は、# grub2-mkconfig -o /boot/grub2/grub.cfg

（参考）Ubuntu の場合は、# update-grub を利用する。

---
# 起動しなくなったときの対処法
* FAT32でフォーマットされている「EFIパーティション」に「ファイルを１つコピーする」だけで起動できるようになる。  
EFIパーティションの「/EFI/centos/grubx64.efi」を「/EFI/BOOT」フォルダにコピーするだけでOK。

（補足）fstab の設定を変更した後、起動できなくなるようである。  
FAT32のパーティションを操作するだけのため、Windowsでも操作可能である。

* JetFlash USB を用いて、CentOS デスクトップを起動。  
ユーザー名は「user-k」で、パスワードは「Sky-Supervisor」と同じ。  
USBメモリを挿し込んで、端末を起動し、「lsblk」を用いてどれが「EFIパーティション」を確認する。恐らく「sdc1」がEFIパーティションになっていると思う。

「su -」で root に移り、sdc1 をどこかにマウントする。  
今は、/work フォルダを作って、そこにマウントするものとする。  
```
# mkdir /work
# mount -t vfat /dev/sdc1 /work

次に、ファイルを１つコピーすればOK
# cd /work/EFI/BOOT
# cp /work/EFI/centos/grubx64.efi grubx64.efi
```
以上で起動が可能になるはず。

---
# 設定の追加
* vim のインストール　# yum install -y vim
* コマンドプロンプトと vim の起動モードの変更
```
# cd
# vim .bashrc

以下の２行を追記（'' の最後に半角スペースを入れておくこと）
export PS1='[\u@\h \w]\$ '
alias vim='vim -c start'
```

---
# ipatables の設定
* まず、firewalld を無効にする。
```
# systemctl status firewalld
# systemctl stop firewalld
# systemctl disable firewalld
```

* iptables のインストールと起動設定
```
# yum install -y iptables-services
# systemctl start iptables
# systemctl enable iptables
# systemctl status iptables
```
（注意）systemctl start ip6tables なども必要になるかも？？

* iptables の設定  
```
# sh ipt_NUC_SMB.sh  
# service iptables save
# iptables -nL
```

* iptables のログを分離する  
iptables のログ出力は syslog が担っているため、syslog の出力にフィルタを掛ける。  
/etc/rsyslog.d/ にフィルタを記述したファイルを置けば、フィルタが掛かるようになる。  
```
# touch /etc/rsyslog.d/iptables.conf
# vim /etc/rsyslog.d/iptables.conf

以下の２行を書き込んでおく
:msg,contains,"Drp_" -/var/log/iptables.log
& ~
```

ファイル指定の前の「-」は、ログ出力時のディスクとの同期の抑制を示す。（パフォーマンス向上が狙える）  
「& ~」は、対象としているログを廃棄する、ということを示す。

・rsyslog の再起動法　# systemctl restart rsyslog  

参考URL：  
http://www.geocities.jp/yasasikukaitou/rsyslog-filter.html  
https://vogel.at.webry.info/201311/article_4.html  

---
# ip フォワーディングが必要な場合
* ip フォワーディングの設定の確認　# sysctl net.ipv4.ip_forward  
* ip フォワーディングの設定　# sysctl -w net.ipv4.ip_forward=1  
「-w」は、sysctl の設定を変更する、という意味  

* 再起動時に設定が戻ってしまうため、設定ファイルを書き換える。
```
# vim /etc/sysctl.conf

sysctl.conf の最下行に、以下の１行を追加すればOK
net.ipv4.ip_forward = 1
```

---
# Swap を無効化したい場合
* 現在のメモリ状況の確認　# free -h  
swapを無効化　# swapoff --all  

・再起動しても swapが無効になるように設定ファイルを編集する。
```
# vim /etc/fstab

上記の操作で開いたファイルの、「swap」という単語が記されている行（１行のみ）をコメントアウトすればOK。
```
・swap がなくなったことの確認　# free

---
# Ramdisk の設定
* fstab の書き換え　# vim /etc/fstab  
以下を追記  

```
# /tmp, /var/tmp -> RAMディスク
tmpfs /tmp tmpfs defaults,size=256m,noatime,mode=1777 0 0
tmpfs /var/tmp tmpfs defaults,size=256m,noatime,mode=1777 0 0

# /var/log -> RAMディスク
tmpfs /var/log tmpfs defaults,size=512m,noatime,mode=0755 0 0
```
（参考）NUC-SMB は、１回の使用時間が短いため、/var/log のサイズも size=256m とした  

noatime： アクセスした際に、アクセス時のタイムスタンプの変更をしない  
mode： アクセス権。先頭の数字はスティッキービット  
５番目の数字： バックアップを作る時を決定するために dump ユーティリティによって使われる。通常は０でよい  
６番目の数字： ファイルシステムをチェックする順番を決めるために fsck によって使われる。RamDiskは０でよい  

* 起動時にRAMディスクにディレクトリを作成するように設定する。一部のサービスは、サービス起動時にワーキングディレクトリの存在が必要となる。  
\# vim /etc/rc.d/rc.local  
以下を追記する  

```
#!/bin/bash

mkdir -p /var/log/anaconda
mkdir -p /var/log/audit
mkdir -p /var/log/chrony
mkdir -p /var/log/fsck
mkdir -p /var/log/rhsm
mkdir -p /var/log/samba
mkdir -p /var/log/tuned

chown root.adm /var/log/samba

# Create Lastlog, wtmp, btmp
touch /var/log/lastlog
touch /var/log/wtmp
touch /var/log/btmp

chown root.utmp /var/log/lastlog
chown root.utmp /var/log/wtmp
chown root.utmp /var/log/btmp
```

* 上記の rc.local に、実行権限を与える　# chmod +x /etc/rc.d/rc.local  

* Ramdisk に移行するフォルダのファイルを空にしておく
```
# find /tmp -type f
# find /tmp -type f -exec cp -f /dev/null {} \;

# find /var/tmp -type f
# find /var/tmp -type f -exec cp -f /dev/null {} \;

# find /var/log -type f
# find /var/log -type f -exec cp -f /dev/null {} \;
```
（参考）ネット上の情報では「# find /var/log/ -type f -name * -exec cp -f /dev/null {} \;」となってたが、-name オプションは不要と判断した

* 再起動　# reboot
* Ramdisk の確認
```
# df -h
# ls -la /var/log
```

---
# Samba テンポラリ用に、大きな ramdisk を用意
* マウント先の用意
```
# mkdir /home/shared
# chown user-k.user-k /home/shared

# mkdir /home/shared/ramdisk
# chown user-k.user-k /home/shared/ramdisk
```

* fstab の設定　# vim /etc/fstab  
以下を追記
```
tmpfs /home/shared/ramdisk tmpfs async,rw,size=12g,noatime,mode=0700,uid=user-k,gid=user-k 0 0
```

---
# NTP の設定
* 設定ファイルの書き換え　# vim /etc/chrony.conf
```
# Please consider joining the pool　の下に、
server ntp.nict.jp iburst
と記述すればOK。サーバーは適宜選べばよい。 
```

* 再起動　# systemctl restart chronyd  
* 接続先の確認　# chronyc sources

---
# ログローテーション の設定
* logrotate は cron によって起動されるため、cron の状態を確認　# systemctl status crond  
* /etc/logrotate.d に設定ファイルを作成することにより、ローテートを行う  
2019-10-22　ローテーションの設定ファイルは、「/etc/logrotate.d/syslog」を参考にして作成した。  
設定ファイルの書き方に関するネットでの情報は、Linux のディストリビューションや、バージョンごとの差異があり不明な点が多かったため、現時点では、現OSが走っている設定ファイルを真似することにした。

```
# touch /etc/logrotate.d/iptables
# vim /etc/logrotate.d/iptables
```
以下を記述
```
/var/log/iptables.log
{
  missingok
  notifempty
  daily
  rotate 15
  create 766
  dateext
  dateformat _%Y-%m-%d
  postrotate
    /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
  endscript
  （su syslog adm が必要になるかも。Ubuntu 18.04 では必要であった）
}
```

* 設定ファイルの動作確認　# logrotate -d /etc/logrotate.d/iptables  
「-d」は dry run を表す。実際には実行せずに、動作確認だけを行う。

（参考URL）  
https://qiita.com/zom/items/c72c7bac63462225971b  
https://oxynotes.com/?p=6493  
https://open-groove.net/linux/logrotate-test/  

---
# SD カードの設定
* 現在のブロックデバイスの状況を確認　# lsblk -f
* フォーマット　# mkfs -t ext4 /dev/mmcblk0p1  
* ラベルを付与　# e2label /dev/mmcblk0p1 SD-Card-32G  
（xfs フォーマットの場合は「# xfs_admin -L "SD-Card-32G" /dev/mmcblk0p1」）  
* 状況の確認　# lsblk -f  
* 起動時に SDカードが mount されるように設定する
```
# mkdir /home/shared/SD-Card-32G
# chown user-k.user-k /home/shared/SD-Card-32G
```

fstab を利用した方法では、うまく mount できなかったため、rc.local を利用することにした。  
fstab が利用できないのは、SDカードのドライバのロードが遅いのかもしれない。  
```
# vim /etc/rc.d/rc.local

以下の１行を追記
mount -o async,rw,noatime -L SD-Card-32G /home/shared/SD-Card-32G
```

---
# Samba の設定
* インストールの確認　# samba -V
* Samba のインストール　# yum install -y samba
* 設定ファイルの書き換え　#vi /etc/samba/smb.conf  
```
[global]
server string = SAMBA SERVER Version %v
wins support = no
wins proxy = no
dns proxy = no（不必要かも？？）

dos charset = CP932
load printers = no
; disable spoolss = yes（不要そうであるため、今は外している）
; syslog = 0　　syslog は deprecated らしい。log は、全て var/log/samba に書き出すものとする
disable netbios = yes（netbios は使用しない）

security = user
（passdb backend = tdbsam はデフォルトで設定されていた）
usershare allow guests = no（ユーザー定義共有という新しい機能らしい）
browseable = yes（no にすると、エクスプローラ上での表示がされなくなった）

以下のセクションを追加
[NUC-SMB]（[]の中は好きな文字列でよい。Windowsでアクセスしたときに表示されるフォルダ名となる）
comment = NUC-SMB
path = /home/shared
browseable = yes
read only = no
valid users = user-k
guest ok = no
writable = yes
create mode = 0600
directory mode = 0700

以下のセクションをすべてコメントアウト（プリンタサーバ機能は利用しない）
[printers]
[print$]
```

* 設定ファイルの構文チェック　# testparm
* Samba のユーザー登録。Linux に存在するユーザーで登録する必要がある　# pdbedit -a user-k  
Linux と Samba では暗号化方式が異なるため、同じパスワードでも両方に登録が必要になる。  

CentOS にユーザーを追加する場合
```
・CentOS ユーザーの追加　# useradd -m ユーザー名
　-m ホームディレクトリを作成
・CentOS ユーザーパスワードの設定　# passwd [username]
```

* グループを作成したい場合
```
# groupadd group-k

グループにユーザーを追加
# gpasswd -a user-k group-k

グループがディレクトリにアクセスできるように権限の設定
# chmod 774 /home/shared
# chgrp group-k /home/shared（所有グループの変更を忘れずに）

グループの登録状況のチェック
# cat /etc/group
```

* /var/log を ramdisk にしているため、smb の起動順序を制限する  
\# vim /usr/lib/systemd/system/smb.service
```
[Unit] の After を以下のように変更する

After=network.target rc-local.service
（nmbd と winbind は利用しないため、削除してよい。winbind は security=domain などの場合に利用されるデーモン）
```

* 起動　# systemctl start smb  
（netbios を使用しないため、nmb の設定は必要ない）
* 自動起動の設定　# systemctl enable smb

* reboot 後に、起動状態の確認　#systemctl status smb

---
# USB 外付けHDD の自動マウント  
* ネットに掲載されている情報のように、udev から直接 mount コマンドを起動させてもデバイスの準備が間に合っていないらしくてマウントできなかった。そのため、systemd のサービスを用いて mount コマンドを実行させることにする。

* smartclt をインストールしておく
```
# yum list installed | grep smartmontools
# yum install -y smartmontools
```

* 外付けHDD がマウントされるマウントポイントを作成する　# mkdir /home/shared/APPZ_01
* udev に systemd のサービスを start させる rule を設定する  
\# vim /etc/udev/rules.d/99-local.rules

```
ACTION=="add", ENV{DEVTYPE}=="partition", ENV{ID_FS_LABEL}=="APPZ_01", RUN+="/bin/systemctl start hdd-automount@%k.service"
```
上記において、%k は KERNEL を表し、sdb1 などのデバイス名が設定される。  
systemctl start hdd-automount@sdb1.service とすると、hdd-automount@.service が、%i = sdb1 として start される。（%i はインスタンス名という意味らしい）

* systemd に、バッチファイルを起動させるサービスを登録する  
\# vim /etc/systemd/system/hdd-automount@.service

```
[Unit]
Description = hdd-auto-mount on %i

[Service]
ExecStart = /home/shared/hdd-automount.sh %i
RemainAfterExit = yes
Type = simple

[Install]
WantedBy = multi-user.target
```

* systemd から実行されるバッチファイルを作成する  
\#vim /home/shared/hdd-automount.sh
```
#!/bin/bash

DEVBASE=$1
DEVICE="/dev/${DEVBASE}"

# ID_FS_LABEL に値を設定させる
eval $(/sbin/blkid -o udev ${DEVICE})

LABEL=${ID_FS_LABEL}
if [[ -z "${LABEL}" ]]; then
  exit 1
fi

MOUNT_POINT="/home/shared/${LABEL}"
/bin/mount -o async,rw,noatime ${DEVICE} ${MOUNT_POINT}


BASH_ON_START="{ echo; echo '-----------------------------------------------'; date; smartctl -A ${DEVICE/%?}; } >> ${MOUNT_POINT}/smartctl_on_start.log"
/bin/bash -c "${BASH_ON_START}"

BASH_OPR="while true; do sleep 540s; { echo; date; smartctl -A ${DEVICE/%?} | grep -e '1 Raw' -e '5 Real' -e '197 Cur'; } >> ${MOUNT_POINT}/smartctl.log; done"
/bin/bash -c "${BASH_OPR}" &
```

* hdd-automount.sh に実行権限を与える　# chmod +x hdd-automount.sh

---
# 自動マウントを考える際、有益となるコマンド
* udevadm info --query=all -n /dev/sdb1
* udevadm control --reload
* systemctl daemon-reload

---
