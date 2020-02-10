# ipv6の無効化
* \# vim /etc/default/grub
```
GRUB_CMDLINE_LINUX="ipv6.disable=1"
```
* \# grub2-mkconfig -o /boot/grub2/grub.cfg
* \# reboot
* \# vim /etc/sysctl.conf
```
net.ipv6.conf.all.disable_ipv6 = 1
```
* \# sysctl -p
* inet6 が消えていることを確認　# ip a
