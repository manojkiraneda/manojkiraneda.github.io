---
title: "OpenBMC kernel patching over u-boot tftp protocol"
date: "2023-12-03 05:58PM"
categories: ["Kernel Patching"]
tags: ["kernel", "patching", "netboot", "u-boot", "tftp"]
---

## The Problem

While working on kernel device drivers, we often stumble up-on situations where
the kernel panics and crashes. But you cannot reflash the system remotely as the
system goes off the network ; since we could not laod the user space networking
stack that is responsible for configuring the network.

Recently , i have hit the same problem with OpenBMC. I was working on a system
which is located around the globe, so the only way that i could access it is via
`serial console` (or) through `ssh`. It was pretty irritating to see that when
ever i flash an openbmc image with some custom kernel device driver changes,
kernel panics and the system goes of the grid. So a person has to manually go to
the lab to manually reprogram the flash using tools like `dediprog`.

In search of ways to *NOT* brick the system, i started looking for ways to
recover the system even after i crash it. I found 2 ways to do this :

1. Switch sides of BMC flash in uboot
2. Kernel live patching via u-boot tftp
3. Use simulators like qemu, simics

## Switch sides of BMC flash in uboot

ast2600 supports two flash banks, so one of them is considered as the active
or the golden side, the other side is called the non-active side. We could put
openbmc image in both of the sides.

So when we put a bad image(assume that it's the one that has got the custom
kernel changes that bricks the system), we can always change the boot side for
next boot in u-boot, so that we can always boot from the golden side. And from
there we can re-flash the non-active side and recover the system.

But unfortunately the system that i am working on does not have the support for
side switching the bmc code in u-boot. So i cannot use it.

## Kernel patching via u-boot tftp

While U-Boot is used to load and execute the OS after initial programming, it
can also be used to program the OS images to the local flash. This page describes
the process of using U-Boot to load Linux kernel and filesystem images from a
TFTP server and save them to the local flash for use during the boot process.

## Use simulators like qemu, simics

We could also load the custom linux image on an emulator like qemu/simics, but
we should have qemu/simics hardware models written to be able to test the software
side of things, for example : if we are planning to test espi protocol, we need
to have espi hardware modelled in qemu/simics. Unfortunately the system that i am
working on - does not even have the hardware models in qemu/simics. So i am left with
just option #2.

### Requirements

1. TFTP server
2. Serial console/minicom connected to the BMC
3. u-boot should have the networking(PHY's) configured.

### TFTP server

Ensure that you have got a working TFTP server that is in the local network of
the BMC. If the `tftp` server is not present in the local network and is
connected via switches and VLAN's we might encounter some wierd issues where you
will not be able to send files bigger than a certain range based on the MTU values
configured in your network between your tftp server and the BMC.

```bash
Austin_Team:tftpboot$ sudo systemctl status tftp
[sudo] password for Austin_Team:
● tftp.service - Tftp Server
   Loaded: loaded (/etc/systemd/system/tftp.service; indirect; vendor preset: disabled)
   Active: active (running) since Mon 2023-12-04 10:08:50 EST; 1min 46s ago
     Docs: man:in.tftpd
 Main PID: 864720 (in.tftpd)
    Tasks: 1 (limit: 49561)
   Memory: 188.0K
   CGroup: /system.slice/tftp.service
           └─864720 /usr/sbin/in.tftpd -B 16268 -s /var/lib/tftpboot

Dec 04 10:08:50 li-62ed7d4c-2bcf-11b2-a85c-b954d3743c4c.ibm.com systemd[1]: Started Tftp Server.
Austin_Team:tftpboot$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s31f6: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether c8:5b:76:af:30:ad brd ff:ff:ff:ff:ff:ff
3: wlp4s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether f0:d5:bf:f9:d8:a0 brd ff:ff:ff:ff:ff:ff
    inet 5.6.7.8/23 brd 5.6.7.255 scope global dynamic noprefixroute wlp4s0
       valid_lft 9156sec preferred_lft 9156sec
    inet6 2620:1f7:817:c5a::32:2ce/128 scope global dynamic noprefixroute
       valid_lft 37476sec preferred_lft 21276sec
    inet6 fe80::d08f:a07:139b:2a5b/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
5: virbr1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 00:16:3e:4f:e2:42 brd ff:ff:ff:ff:ff:ff
    inet 192.168.123.1/24 brd 192.168.123.255 scope global virbr1
       valid_lft forever preferred_lft forever
6: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether 00:16:3e:77:93:c2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
```

### u-boot network configuration

In my case both the ethernet nodes of ast2600 are manually diabled in u-boot
device tree. So i had to enable them. Here are the changes i made in the u-boot
device tree to enable the network in uboot. In my case the BMC has a dedicated
ethernet connect and *does NOT* have *NCSI*.

```bash
+++ b/meta-ibm/meta-sbp1/recipes-bsp/u-boot/u-boot-aspeed-sdk/0001-board-ibm-sbp1-Add-sbp1-board.patch
@@ -79,6 +79,13 @@ index 0000000000..87afa2cd9a
 +&mac1 {
 +      status = "disabled";
 +};
++&mac2 {
++      status = "okay";
++};
++
++&mac3 {
++      status = "okay";
++};
 +
 +&gpio0 {
 +      status = "okay";
```

With the above changes in the u-boot device tree, i was able to see the network
adapters being detected in the serial console of the BMC:

```bash
U-Boot SPL 2019.04 (Jul 24 2023 - 12:31:15 +0000)
already initialized, Trying to boot from RAM
U-Boot 2019.04 (Jul 24 2023 - 12:31:15 +0000)
SOC: AST2600-A3
RST: WDT1 SOC
PCI RST: #1 #2
eSPI Mode: SIO:Enable : SuperIO-2e
Eth: MAC0: RGMII, MAC1: RGMII, MAC2: RGMII, MAC3: RGMII
Model: IBM SBP1
DRAM:  already initialized, 496 MiB (capacity:512 MiB, VGA:16 MiB, ECC:off)
Loading Environment from SPI Flash... SF: Detected w25q512jv with page size 256 Bytes, erase size 4 KiB, total 64 MiB
OK
In:    serial@1e783000
Out:   serial@1e783000
Err:   serial@1e783000
Model: IBM SBP1
Net:   eth0: ftgmac@1e670000, eth1: ftgmac@1e690000
Hit any key to stop autoboot:  0
```

Notice the `Net:   eth0: ftgmac@1e670000, eth1: ftgmac@1e690000` line above.

Now lets gather certain information from user space before configuring the u-boot
to connect to the tftp server:

1. ip address of the BMC

```bash
root@sbp1:~# ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether fe:02:03:04:fe:c4 brd ff:ff:ff:ff:ff:ff
    inet 1.2.3.4/24 brd 1.2.3.255 scope global dynamic eth0
       valid_lft 34382sec preferred_lft 34382sec
    inet6 fe80::fc02:3ff:fe04:fec4/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether fe:02:03:04:fe:c5 brd ff:ff:ff:ff:ff:ff
```

Grab the ip from eth0  : `1.2.3.4`
2. Default gateway through which BMC is connected to the network

```bash
root@sbp1:~# ip route show
default via 1.2.3.4 dev eth0  src 1.2.3.0  metric 1024
```

Grab the default gateway : `1.2.3.0`

3.MAC address

```bash
root@sbp1:~# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether fe:02:03:04:fe:c4 brd ff:ff:ff:ff:ff:ff
3: eth1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether fe:02:03:04:fe:c5 brd ff:ff:ff:ff:ff:ff
```

Grab the eth0 MAC : `fe:02:03:04:fe:c4`

## Procedure

In u-boot prompt, lets set a couple of things to establish the connection to the
tftp server:

1.tftp server ip address

```bash
ast# setenv serverip 5.6.7.8
```

2.bmc ipaddress

Set the BMC ip address from the collected information in userspace.

```bash
ast#  setenv ipaddr 1.2.3.4
```

3.gateway ip address

Set the gateway ip address from the collected information in userspace

```bash
ast# setenv gatewayip 1.2.3.0
```

4.Double check , if you have set the information correct

```bash

ast# printenv
baudrate=115200
bootargs=console=ttyS0,115200n8 root=/dev/ram rw earlycon
bootcmd=run bootspi
bootdelay=2
bootfile=all.bin
bootspi=fdt addr 20100000 && fdt header get fitsize totalsize && cp.b 20100000 ${loadaddr} ${fitsize} && bootm; echo Error loading kernel FIT image
eth1addr=fe:02:03:04:fe:c5
ethact=ftgmac@1e670000
ethaddr=fe:02:03:04:fe:c4
fdtcontroladdr=9cf90188
gatewayip=1.2.3.0
ipaddr=1.2.3.4
loadaddr=0x83000000
netmask=255.255.255.0
serverip=5.6.7.8
stderr=serial@1e783000
stdin=serial@1e783000
stdout=serial@1e783000
verify=yes
```

5.Save the uboot properties to persistent flash

```bash
ast# saveenv
Saving Environment to SPI Flash... SF: Detected w25q512jv with page size 256 Bytes, erase size 4 KiB, total 64 MiB
Erasing SPI flash...Writing to SPI flash...done
OK
```

6.Now lets ping the `tftp` server `(5.6.7.8)`

Make sure your BMC u-boot can ping the `tftp` server

```bash
ast# ping 5.6.7.8
ftgmac@1e670000: link up, 100 Mbps full-duplex mac:fe:02:03:04:fe:c4
Using ftgmac@1e670000 device
host 5.6.7.8 is alive
```

7.Place the openbmc build generated fit image in the tftp server directory

Grab the `fitImage-obmc-phosphor-initramfs-sbp1--6.5.4+git0+93ab0de67a-r0-sbp1-20231129090329.bin`
file and place it in the tftp server directory.

```bash
manojeda@gfwa825:~/sbp1_code/working_copy/openbmc/build/sbp1/tmp/deploy/images/sbp1$ ls -l | grep fit | grep init
-rw-r--r-- 2 manojeda 1056555     2372 Nov 29 03:10 fitImage-its-obmc-phosphor-initramfs-sbp1--6.5.4+git0+93ab0de67a-r0-sbp1-20231129090329.its
lrwxrwxrwx 2 manojeda 1056555       91 Nov 29 03:10 fitImage-its-obmc-phosphor-initramfs-sbp1-sbp1 -> fitImage-its-obmc-phosphor-initramfs-sbp1--6.5.4+git0+93ab0de67a-r0-sbp1-20231129090329.its
-rw-r--r-- 2 manojeda 1056555  4766103 Nov 29 03:10 fitImage-obmc-phosphor-initramfs-sbp1--6.5.4+git0+93ab0de67a-r0-sbp1-20231129090329.bin
lrwxrwxrwx 2 manojeda 1056555       87 Nov 29 03:10 fitImage-obmc-phosphor-initramfs-sbp1-sbp1 -> fitImage-obmc-phosphor-initramfs-sbp1--6.5.4+git0+93ab0de67a-r0-sbp1-20231129090329.bin
lrwxrwxrwx 2 manojeda 1056555       42 Nov 30 08:50 image-kernel -> fitImage-obmc-phosphor-initramfs-sbp1-sbp1

manojeda@gfwa825:~/sbp1_code/working_copy/openbmc/build/sbp1/tmp/deploy/images/sbp1$ scp fitImage-obmc-phosphor-initramfs-sbp1--6.1.15+git0+c4c0b57522-r0-sbp1-20231121071040.bin Austin_Team@9.57.217.0:/var/lib/tftpboot/fit

Austin_Team:tftpboot$ ls
fit
```

8.Now load the fit image from the tftp server from the uboot tftp client

```bash
ast# tftp 5.6.7.8:fit
ftgmac@1e670000: link up, 100 Mbps full-duplex mac:fe:02:03:04:fe:c4
Using ftgmac@1e670000 device
TFTP from server 5.6.7.8; our IP address is 1.2.3.4; sending through gateway 1.2.3.0
Filename 'fit'.
Load address: 0x83000000
Loading: #################################################################
         #################################################################
         #################################################################
         #################################################################
         ######################################################
         689.5 KiB/s
done
Bytes transferred = 4605607 (4646a7 hex)
ast#
```

9.Boot the fit image obtained from tftp server.

```bash
ast# bootm
## Loading kernel from FIT Image at 83000000 ...
   Using 'conf-aspeed-bmc-ibm-sbp1.dtb' configuration
   Verifying Hash Integrity ... OK
   Trying 'kernel-1' kernel subimage
     Description:  Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x83000114
     Data Size:    3408376 Bytes = 3.3 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x80001000
     Entry Point:  0x80001000
     Hash algo:    sha256
```

Vola!!, you have booted the custom fit image using u-boot via tftp.
