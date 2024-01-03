---
title: "KGDB on OpenBMC Kernel"
date: "2023-12-21 05:58PM"
categories: ["Kernel Debugging"]
tags: ["kernel", "debugging", "kgdb", "gdb-tui", "tftp debugging"]
---

It's the most serene period of the year — devoid of notifications, calls, or
disruptions. Everyone in the vicinity appears content, and I, too wanted to
experience the joy in successfully concluding the testing of my espi kernel
device driver. Confidence surged within me, this time with a palpable certainty
that I am on the verge of submitting my inaugural kernel patch and poised to
conclude the year with a flourish. However, my ordeal commenced with the initial
dry run.

## The Problem

I was trying to model the `espi flash channel` as an `mtd device` on BMC. I see the
module is built successfully, the driver is probed succesfully.But the mtd device
is not populated in the `/sys` file system.

## What is /sys filesystem anyway ?

`sysfs` is a pseudo file system provided by the Linux kernel that exports
information about various kernel subsystems, hardware devices, and associated
device drivers from the kernel's device model to user space through virtual files.

In addition to providing information about various devices and kernel subsystems,
exported virtual files are also used for their configuration. sysfs provides
functionality similar to the sysctl mechanism found in BSD operating systems, with
the difference that sysfs is implemented as a virtual file system instead of being
a purpose-built kernel mechanism, and that, in Linux, sysctl configuration
parameters are made available at /proc/sys/ as part of procfs, not sysfs which
is mounted at /sys/.

## /sys filesystem - an example

Here is the sample output from my bmc box :

```bash
root@sbp1:/sys# ls
block     bus       class     dev       devices   firmware  fs        kernel
module    power
root@sbp1:/sys# cd class/
root@sbp1:/sys/class# ls
bdi           devlink       gpio          i2c-dev       mdio_bus      mtd
ptp           scsi_device   spi_master    video4linux   zram-control
block         drm           hidg          input         mem           net
regulator     scsi_disk     tty           wakeup        bsg           fsi-master
hwmon         leds          misc          pps           rtc           scsi_host
udc           watchdog
```

Most of the directories here are self explanatory here, for example:
`/sys/class/mtd` - folder contains all the devices of mtd class
`/sys/class/gpio` - folder contains all the devices of gpio class

so , if you create an `mtd device`, that its expected to be seen in the `/sys/class/mtd`
folder, with all vritual files populated to provide its configuration.

## Hope of figuring out the issue using KGDB

I hate debugging with traces & printks.

I have heard that GDB in client - server mode could actually work on live kernel,
so my expectation was that i could use a GDB client and connect to the live kernel
and stop at my device driver probe function, and walk though the code in step mode.
And in the process was expecting the route cause of the problem to be revealed.

## Ingredients needed to get KGDB working

1. Hardware configuration
2. Enabling KGDB support in kernel & Enabling the kernel debug symbols
3. Boot kernel with KGDB support
4. [Optional] TUI support for GDB
5. Client GDB in Host

### Hardware configuration

Two machines are required for using kgdb:

* **Host/Development Machine**: Runs gdb client against the vmlinux file which
    contains the symbols and performs debugging
* **Target Machine**: Runs `kgdb` and is the machine to be debugged (BMC)

```text
        ------------------                              --------------------
        |       Host      |                             |       Target     |
        |                 |                             |                  |
        |  -------------  |                             |   ------------   |
        | |     gdb     | |<--------------------------->|  |    kgdb    |  |
        | |             | |             Serial or       |  |            |  |
        | --------------  |             Ethernet        |  -------------   |
        |       |         |             Connection      |        |         |
        |  -------------- |                             |  --------------  |
        | | Kernel image ||                             |  |Linux Kernel | |
        | | with debug   ||                             |  |(zImage)     | |
        | | symbols      ||                             |  |             | |
        | | (vmlinux)    ||                             |  --------------- |
        | ----------------|                             |                  |
        -------------------                             --------------------
```

KGDB is a GDB Server implementation integrated in the Linux Kernel, It supports:

* Serial port communication (available in the mainline kernel) and
* Network communication (patch required)

It enables full control over kernel execution on target, including memory read
and write, step-by-step execution and even breakpoints in interrupt handler.

In my case , i have small laptop(`/dev/ttyUSB1`) connected to the serial console
of the BMC(`/dev/ttyS0`),so that would be my `Host`(as per the above picture) and
BMC is my `Target`.

### Enabling KGDB & Debug symbols while building the kernel

Here are some kernel configuration that i had to enable before building the linux
kernel, to get KGDB working:

* **CONFIG_VT=y**
  * This will get support for terminal devices with display and keyboard
    devices.These are called "virtual" because you can run several virtual
    terminals (also called virtual consoles) on one physical terminal

* **CONFIG_VT_CONSOLE=y**
  * The system console is the device which receives all kernel messages and
    warnings and which allows logins in single user mode. This this will
    enable virtual terminal (the device used to interact with a physical terminal)
    can be used as system console.

* **CONFIG_KGDB=y**
  * This will enable the support for remotely debugging the kernel using gdb.

* **CONFIG_MAGIC_SYSRQ=y**
  * This will have some control over the system even if the system crashes for
    example during kernel debugging (e.g., you will be able to flush the buffer
    cache to disk, reboot the system immediately or dump some status information).
    This is accomplished by pressing various keys while holding SysRq (Alt+PrintScreen).
    It also works on a serial console (on PC hardware at least), if you send a
    BREAK and then within 5 seconds a command keypress. The keys are documented
    in [Documentation/admin-guide/sysrq.rst](https://www.kernel.org/doc/Documentation/admin-guide/sysrq.rst)

* **CONFIG_DEBUG_INFO=y**
  * This adds debug symbols to the kernel and modules (gcc -g), and is needed
    if you intend to use kernel crashdump or binary object tools like crash,
    kgdb, LKCD, gdb, etc on the kernel.

* **CONFIG_KGDB_SERIAL_CONSOLE=y**
  * Serial console support for KGDB

**Optional kernel configuration:**

* **CONFIG_FRAME_POINTER=y**
  * Allows you to more accurately construct stack back traces while debugging
    the kernel
* **CONFIG_DEBUG_RODATA=n**
  * If the architecture supports this option, you should consider turning this
    off, as this prevents the use of software breakpoints because it marks the
    certain regions of the kernel memory space as read-only. Or you can use
    hardware break points in gdb, if you still want this option to be enabled.

### Boot kernel with KGDB support on Target

After your kernel is built with KGDB support, it needs be enabled and configured.
In general, KGDB is enabled by passing a command line switch to the kernel via
the kernel command line. If KGDB support is compiled into the kernel but not
enabled via a command line switch, it does nothing.

When KGDB is enabled, the kernel stops at a KGDB-enabled breakpoint very early
in the boot cycle to allow you to connect to the target using gdb

Append the following command line to kernel command line in grub:

```bash
kgdboc=ttyS0,115200 kgdbwait
```

* **kgdbdoc**: kgdb over console
  * This is a KGDB I/O driver and we are supplying two arguments one informing
    the kernel to use ttyS0 serial port and the other 115200 baud rate.
* **kgdwait**: Informs the kernel to wait until the debugger is attached

So, observe the serial console while the bmc boots, and stop at the u-boot prompt
as shown below:

```bash
root@mybox:~# reboot -f
Rebooting.
[50884.891740] reboot: Restarting system

U-Boot SPL 2019.04 (Jul 24 2023 - 12:31:15 +0000)
already initialized, Trying to boot from RAM

U-Boot 2019.04 (Jul 24 2023 - 12:31:15 +0000)

SOC: AST2600-A3
RST: WDT1 SOC
PCI RST: #1 #2
eSPI Mode: SIO:Enable : SuperIO-2e
Eth: MAC0: RMII/NCSI, MAC1: RMII/NCSI, MAC2: RGMII, MAC3: RGMII
Model: MY BOX
DRAM:  already initialized, 496 MiB (capacity:512 MiB, VGA:16 MiB, ECC:off)
Loading Environment from SPI Flash... SF: Detected w25q512jv with page size 256 Bytes, erase size 4 KiB, total 64 MiB
OK
In:    serial@1e783000
Out:   serial@1e783000
Err:   serial@1e783000
Model: IBM SBP1
Net:   eth0: ftgmac@1e670000, eth1: ftgmac@1e690000
Hit any key to stop autoboot:  0
ast#
```

Display the bootargs using `printenv` command in boot:

```bash
bootargs=console=ttyS0,115200n8 root=/dev/ram rw earlycon ignore_loglevel
```

Now append kgdboc parameters to `bootargs` property using `setenv`:

```bash
bootargs=console=ttyS0,115200n8 root=/dev/ram rw earlycon ignore_loglevel kgdboc=ttyS0,115200 kgdbwait
```

> Note that the kgdboc=ttyS0 points to the serial console on the bmc side.
{: .prompt-tip }
> Additional step : If you have custom changes in kernel code, you could even [patch the linux kernel using tftp](https://manojkiraneda.github.io/posts/kernel-patching-over-uboot-tftp/)
{: .prompt-info}

Now, boot the kernel using `bootm` command.

You should now see that the kernel would just boot enough and hit a sync point
and wait's till a GDB client is connected over the serial console. It should look
like this :

```bash
[    9.947575] KGDB: Registered I/O driver kgdboc
[    9.952644] KGDB: Waiting for connection from remote gdb...
```

### GDB client in host

There are two steps in configuring the GDB client:

1. Agent Proxy
2. Configuring the GDB client on host to connect to KGDB on target

### Agent Proxy

If you have only one serial port and want to use for both console and KGDB then
you have to go with agent proxy. `agent-proxy` is nothing more then a tty → tcp
connection mux that can allow you to connect more than one client application
to a tty. By using it, we can run the serial port client and gdb process simultaneously.

Here is the following syntax to run the agent-proxy command :

```bash
agent-proxy CONSOLE_PORT^DEBUG_PORT IP_ADDR PORT
```

```text
              ------------------
              |Target system   |
              |with serial port|
              ------------------
                     |
                     V
             -------------------
             |  agent-proxy    |
             -------------------
              /              \
             /                \
            /                  \
        ----------------    ---------
        |  For          |   |       |
        |console access |   |gdb    |
        |<port1>        |   |<port2>|
        ----------------    ---------
```

#### Building a Poxy Agent

Below are the steps to build and run a proxy agent:

```bash
git clone http://git.kernel.org/pub/scm/utils/kernel/kgdb/agent-proxy.git
cd agent-proxy ; make
./agent-proxy 5550^5551 0 /dev/ttyUSB1,115200
```

`/dev/ttyUSB1` if you are using USB to serial device node.
Now, you have `5550 port for console` and `5551 port for kgdb`

#### To get the console logs

You could use telnet to get to console logs:

```bash
telnet localhost 5550
```

### Configuring the GDB client on host to connect to KGDB on target

Source the yocto built sdk, so that we could use the GDB cross compiled for arm:

```bash
Austin_Team:manoj_test$ echo $GDB
arm-openbmc-linux-gnueabi-gdb
```

Run the gdb with the generated vmlinux binary that we have obtained after compiling
the kernel with `CONFIG_DEBUG_INFO=y` as mentioned above.

```bash
Austin_Team:manoj_test$ $GDB vmlinux
GNU gdb (GDB) 13.2
Copyright (C) 2023 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-oesdk-linux --target=arm-openbmc-linux-gnueabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from vmlinux...

warning: could not convert 'main' from the host encoding (ANSI_X3.4-1968) to UTF-32.
This normally should not happen, please file a bug report.
(gdb) set serial baud 115200
(gdb) set substitute-path /usr/src/kernel /home/Austin_Team/manoj_test/kernel-source
(gdb) target remote localhost:5551
Remote debugging using localhost:5551
warning: multi-threaded target stopped without sending a thread-id, using first non-exited thread
[Switching to Thread 4294967294]
arch_kgdb_breakpoint () at /usr/src/kernel/kernel/debug/debug_core.c:1218
1218  arch_kgdb_breakpoint();
(gdb) bt
#0  arch_kgdb_breakpoint () at /usr/src/kernel/kernel/debug/debug_core.c:1218
#1  kgdb_breakpoint () at /usr/src/kernel/kernel/debug/debug_core.c:1218
#2  0x801b5bdc in kgdb_initial_breakpoint () at /usr/src/kernel/kernel/debug/debug_core.c:1020
#3  kgdb_register_io_module (new_dbg_io_ops=0x80d86780 <kgdboc_io_ops>) at /usr/src/kernel/kernel/debug/debug_core.c:1157
#4  0x804c1f08 in configure_kgdboc () at /usr/src/kernel/drivers/tty/serial/kgdboc.c:221
#5  0x804c1fec in kgdboc_probe (pdev=0x80de55b8 <kgdb_setting_breakpoint>) at /usr/src/kernel/drivers/tty/serial/kgdboc.c:248
#6  0x8050ea3c in class_dev_iter_init (iter=0x8187f410, class=0x0, start=0x0, type=0x0) at /usr/src/kernel/drivers/base/class.c:312
#7  0x80d867c0 in kgdboc_platform_driver ()
```

> Note that if you want to single step through the kernel and also look at the
current executing line visually, you would need to start $GDB with `--tui` option
{: .prompt-tip }

Vola !! :dancer: Now you can use all GDB commands , dump registers/variables, observer the
kernel stack frames & do much more cool stuff.

### Problem for another day

So, now back to our problem of why does my mtd device not populated in `/sys/class/mtd` ?
To answer this, i need to put a break point in the probe function of my driver
and run the GDB.

For some reason my driver is being probed way earlier than the KGDB waiting for
the client GDB.

```bash
[    9.306516] aspeed-espi-ctrl 1e6ee000.espi-ctrl: module loaded
......
[    9.947575] KGDB: Registered I/O driver kgdboc
[    9.952644] KGDB: Waiting for connection from remote gdb...
```

Hence, there may be a necessity to re-probe the driver following the connection
of the GDB client. Currently, openbmc lacks support for probing or re-probing
due to the integration of all kernel modules into the kernel.

Nevertheless, the KGDB functionality is quite fascinating!

We can address the initial issue on a different occasion.

Cheers! Wishing you a Happy New Year :)
