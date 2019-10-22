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
# 種々の設定
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
\# cat /etc/systemd/logind.conf  
上記のファイルの設定項目で、以下を確認  
HandlePowerKey=poweroff  

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
# 起動しなくなったときの対処法
