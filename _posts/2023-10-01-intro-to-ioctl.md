---
title: "Introduction to ioctl()"
date: "2023-10-01 6:15AM"
categories: ["ioctl"]
tags: ["device drivers", "kernel interface", "Linux"]
---

POSIX standard presents to the user space a very clever abstraction of the
underlying hardware: everything can be manipulated as a `file`, using file
descriptors and the standard system calls (`open`, `close`, `read`, `write`, etc.).
It allows you to send a job to a printer, read from the console and connect to
a remote server (using a socket); all using the same interface.

However, some devices have additional control functions that doesn't translate
well to a file-like interface. For example, you have a thumb drive connected to
an USB port, how can you access the port itself if all you can do is to read
and write to the connected disk ?

To help handle this kind of situation, the ioctl system call was created. It
handles non-specific interactions with devices without special-purpose system
calls. It works sending a special request code to the device driver to set or
get specific parameters on the device. The system call accepts the following
arguments:

- the file descriptor of the device.
- the device request code (the complete list of registered request codes can be
   retrieved calling ioctl_list).
- an optional pointer to a memory buffer, to receive the requested data or to
   send data to the device.

The code below shows how to open a CD drive tray using ioctl call:

```python
import fcntl
import os

CD_DEVICE = '/dev/cdrom'
CD_EJECT = 0x5309 

# From https://github.com/torvalds/linux/blob/master/include/uapi/linux/cdrom.h#L65

# You need to open the device, not the link
if os.path.islink(CD_DEVICE):
    device_path = os.readlink(CD_DEVICE)
    if not device_path.startswith('/'):
        device_path = os.path.dirname(CD_DEVICE) + '/' + device_path
else:
    device_path = CD_DEVICE

device_fd = os.open(device_path, os.O_RDONLY | os.O_NONBLOCK )
fcntl.ioctl(device_fd, CD_EJECT, 0)
close(device_fd)
```

Although very simple to implement and use, it has a number of downsides:

- Since the interface requires the user program to manipulate device-driver
   specific internal structures, accomplishing more sophisticate jobs using the
   interface can become very complex. Moreover, the interface is not necessarily
   the same between two different device drivers of the same device type (for
   example, checking the ink level might require a different request for different
   printer vendors)

- There is no fixed behavior for the interface, each distinct request code behaves
   in a different way and produces a different result. This make auditing security
   vulnerabilities harder than conventional system calls. Besides, it allows the
   user program to pass arbitrary data to the kernel space, a potential vector
   for exploiting vulnerabilities in the device driver.

As the name implies, ioctl is specific for I/O devices. There are equivalent system
calls for files and sockets: `fcntl` and `getsockopt`/`setsockopt`, respectively.

## What's uAPI ?

In the above example, the CDROM IOCTL number `0x5309` is defined as an include
as part of the `uAPI`/(User Space API). The [`uapi`](https://github.com/torvalds/linux/tree/master/include/uapi/linux)
folder is supposed to contain the user space API of the kernel. Then upon kernel
installation, the uapi include files become the top level /usr/include/linux/ files.

The other headers in theory are then private to the kernel. This allow clean
separation of the user-visible and kernel-only structures which previously were
intermingled in a single header file.
