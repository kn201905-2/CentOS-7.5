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
