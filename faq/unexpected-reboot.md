### 一、现象：
```text
客户反馈：下午119这台机器好像被重启了3次，但是不是客户主动操作的reboot
```

### 二、排查
1、先进BMC查看客户所说是否属实，排查是否是电源或者其他原因造成的重启

<img width="2065" height="194" alt="image" src="https://github.com/user-attachments/assets/0522c675-df56-4dbf-8372-8507a3225629" />

以截图为例，说明设备并不是外部原因造成的关机重启，而是从系统内操作的reboot

2、执行：history ，但是没有找到reboot命令，客户反馈自己也没有操作reboot命令

3、网络和系统同事排查，没有其他权限或者异常IP登录服务器进行操作

### 定位
1、查看系统文件
```text
cat /var/log/syslog
```
从BMC中锁定设备Set ACPI power state successfully 的时间，但是并没有找到线索，文件甚至一切正常

2、查看/var/lib/systemd/pstore/
```text
pstore 可以简单理解成：Linux 专门用来保存“系统崩溃前最后日志”的持久化区域。

正常情况下，服务器突然 Kernel Panic、硬件 Fatal Error、掉电或自动重启时，普通 dmesg 和内存里的日志可能来不及写入磁盘。pstore 的作用，就是借助 BIOS/UEFI 提供的持久化存储，比如 ERST、EFI 变量，把崩溃前的关键内核日志先保存下来。机器重新启动后，systemd-pstore 再把这些内容整理到：

/var/lib/systemd/pstore/
```

果然看到
```text
root@RTX6000D-4:/# ll /var/lib/systemd/pstore/
total 24
drwxr-xr-x  6 root root 4096 Sep 23 20:41 ./
drwxr-xr-x 11 root root 4096 Jul 18 19:54 ../
drwxr-xr-x  2 root root 4096 Sep 23 11:27 7688566173708/
drwxr-xr-x  2 root root 4096 Sep 23 15:05 7688622450665/
drwxr-xr-x  2 root root 4096 Sep 23 17:39 7688662063148/
drwxr-xr-x  2 root root 4096 Sep 23 20:41 7688708848227/
```

与设备重启时间吻合

每个pstore下的子文件夹结构如下：
```text
root@RTX6000D-4:/# ll /var/lib/systemd/pstore/7688566173708
total 120
drwxr-xr-x 2 root root  4096 Sep 23 11:27 ./
drwxr-xr-x 6 root root  4096 Sep 23 20:41 ../
-rw------- 1 root root 17696 Sep 23 11:23 dmesg-erst-7688566173708845057
-rw------- 1 root root 17747 Sep 23 11:23 dmesg-erst-7688566173708845058
-rw------- 1 root root 17698 Sep 23 11:23 dmesg-erst-7688566173708845059
-rw-r----- 1 root root 53237 Sep 23 11:27 dmesg.txt
root@RTX6000D-4:/#
```
执行 ```text  cat /var/lib/systemd/pstore/7688566173708/dmesg.txt  ```
其中error部分如下：

<img width="2553" height="1320" alt="image" src="https://github.com/user-attachments/assets/87ee4918-f228-47d3-8c8e-214edd47f6d6" />

这里报错指向了
```text
<0>[238848.773572] {1}[Hardware Error]:   device_id: 0000:13:00.0
<0>[238848.773573] {1}[Hardware Error]:   slot: 71
```
<img width="1626" height="406" alt="image" src="https://github.com/user-attachments/assets/7b295173-509a-4594-bd09-46c370cce368" />

这么来看是CX7网卡导致的，但是如果把/var/lib/systemd/pstore下的文件都看一遍发现每次报错的网卡都不同，可以说是4张网卡挨个依次都造成过重启。
那么问题就从某一张网卡有问题变成了
1、要么是主板的问题，如果是板子的问题分为硬件故障和固件版本的原因，前面在BMC中我们看了日志应该不是硬件故障，可能会是固件版本的原因
2、CX7网卡的固件版本或者驱动版本的原因

### 检查
更倾向于CX7网卡的驱动版本和固件版本不同
查询驱动版本
```text
root@RTX6000D-4:~# ofed_info -s
MLNX_OFED_LINUX-24.10-3.2.5.0:
```

查询固件版本
```text
root@RTX6000D-4:~# mlxfwmanager --query
Device #13:
----------

  Device Type:      ConnectX7
  Part Number:      MCX755106AS-HEA_Ax
  Description:      NVIDIA ConnectX-7 HHHL Adapter Card; 200GbE (default mode) / NDR200 IB; Dual-port QSFP112; PCIe 5.0 x16 with x16 PCIe extension option; Crypto Disabled; Secure Boot Enabled
  PSID:             MT_0000000834
  PCI Device Name:  /dev/mst/mt4129_pciconf0
  Base MAC:         605e65fd5b72
  Versions:         Current        Available     
     FW             28.44.1036     N/A           
     PXE            3.7.0500       N/A           
     UEFI           14.37.0014     N/A           

  Status:           No matching image found
```
nvidia官网上说明：MLNX_OFED_LINUX-24.10-3.2.5.0匹配的固件版本是：28.43.3608
<img width="2559" height="1365" alt="image" src="https://github.com/user-attachments/assets/76412b22-138a-4470-8cd7-727b89021928" />
但是这个也不能绝对的只看nvidia，还要看看厂商的意见


