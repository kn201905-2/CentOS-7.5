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
```

* iptables のログを分離する

