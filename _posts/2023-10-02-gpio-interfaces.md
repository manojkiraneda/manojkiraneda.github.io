---
title: "GPIO Interfaces"
date: "2023-10-02 8:21AM"
categories: ["GPIO subsystem"]
tags: ["device drivers", "kernel interface", "Linux", "GPIO"]
---

## What is GPIO ?

A “General Purpose Input/Output” (GPIO) is a flexible software-controlled digital
signal. Each GPIO represents a bit connected to a particular pin. Board schematics
show which external hardware connects to which GPIOs.

System-on-Chip (SOC) processors heavily rely on GPIOs. In some cases, every
non-dedicated pin can be configured as a GPIO; and most chips have at least several
dozen of them. Programmable logic devices (like FPGAs) can easily provide GPIOs;
multifunction chips like power managers, and audio codecs often have a few such
pins to help with pin scarcity on SOCs; and there are also “GPIO Expander” chips
that connect using the I2C or SPI serial buses.

The exact capabilities of GPIOs vary between systems. Common options:

* Output values are writable (high=1, low=0). Some chips also have options about
how that value is driven, so that for example only one value might be driven,
supporting `wire-OR` and similar schemes for the other value (notably, `open drain`
signaling).
* Input values are likewise readable (1, 0). Some chips support readback of pins
configured as `output`, which is very useful in such `wire-OR` cases (to support
bidirectional signaling). GPIO controllers may have input de-glitch/debounce logic,
sometimes with software controls.
* Inputs can often be used as IRQ signals, often edge triggered but sometimes
level triggered. Such IRQs may be configurable as system wakeup events, to wake
the system from a low power state.
* Usually a GPIO will be configurable as either input or output, as needed by
different product boards; single direction ones exist too.
* Most GPIOs can be accessed while holding spinlocks, but those accessed through
a serial bus normally can’t. Some systems support both types.

On a given board each GPIO is used for one specific purpose like monitoring MMC/SD
card insertion/removal, detecting card write-protect status, driving a LED,
configuring a transceiver, bit-banging a serial bus, poking a hardware watchdog,
sensing a switch, and so on.

## Common GPIO Properties

### Active-High and Active-Low

It is natural to assume that a GPIO is “active” when its output signal is 
1 (“high”), and inactive when it is 0 (“low”). However in practice the signal of
a GPIO may be inverted before is reaches its destination, or a device could decide
to have different conventions about what “active” means. Such decisions should be
transparent to device drivers, therefore it is possible to define a GPIO as being
either active-high (“1” means “active”, the default) or active-low (“0” means
“active”) so that drivers only need to worry about the logical signal and not
about what happens at the line level.

### Open Drain and Open Source

Sometimes shared signals need to use `open drain` (where only the low signal level
is actually driven), or “open source” (where only the high signal level is driven)
signaling. That term applies to CMOS transistors; “open collector” is used for TTL.
A pullup or pulldown resistor causes the high or low signal level. This is sometimes
called a `wire-AND`; or more practically, from the negative logic (low=true)
perspective this is a `wire-OR`.

## GPIO Interface

GPIO stands for General-Purpose Input/Output and is one of the most commonly used
peripherals in an embedded Linux system.

Internally, the Linux kernel implements the access to GPIOs via a producer/consumer
model. There are drivers that produce GPIO lines (GPIO controllers drivers) and
drivers that consume GPIO lines (keyboard, touchscreen, sensors, etc).

To manage the GPIO registration and allocation there is a framework inside the
Linux kernel called gpiolib. This framework provides an API to both device drivers
running in kernel space and user space applications.


## The old way: sysfs interface

Until Linux version 4.7, the interface to manage GPIO lines in user space has
always been in sysfs via files exported at /sys/class/gpio. So for example, if I
want to set a GPIO, I would have to:

 - Identify the number of the GPIO line.
 - Export the GPIO writing its number to /sys/class/gpio/export.
 - Configure the GPIO line as output writing out to /sys/class/gpio/gpioX/direction.
 - Set the GPIO writing 1 to /sys/class/gpio/gpioX/value.

As a practical example, to set GPIO 504 from user space, we would have to execute
the following commands:

```bash
# echo 504 > /sys/class/gpio/export
# echo out > /sys/class/gpio/gpio504/direction
# echo 1   > /sys/class/gpio/gpio504/value
```

This interface is very simple and works pretty well, but has some deficiencies:
 - The allocation of the GPIO is not tied to any process, so if the process using
   a GPIO ends its execution or crashes, the GPIO line may remain exported.
 - We can have multiple processes accessing the same GPIO line, so concurrency
   could be a problem.
 - Writing to multiple pins would require open()/read()/write()/close() operations
   to a lot of files (export, direction, value, etc).
 - The polling process to catch events (interrupts from GPIO lines) is not reliable.
 - There is no interface to configure the GPIO line (open-source, open-drain, etc).
 - The numbers assigned to GPIO lines are not stable.

## The new way: chardev interface

Since Linux version 4.8 the GPIO sysfs interface is deprecated, and now we have
a new API based on character devices to access GPIO lines from user space.

Every GPIO controller (gpiochip) will have a character device in /dev and we can
use file operations (open(), read(), write(), ioctl(), poll(), close()) to manage
and interact with GPIO lines:

```bash
# ls /dev/gpiochip*
/dev/gpiochip0  /dev/gpiochip2  /dev/gpiochip4  /dev/gpiochip6
/dev/gpiochip1  /dev/gpiochip3  /dev/gpiochip5  /dev/gpiochip7
```

Although this new char device interface prevents manipulating GPIO with standard
command-line tools like echo and cat, it has some advantages when compared to the
sysfs interface, including:

 - The allocation of the GPIO is tied to the process that it is using it, improving control over which GPIO lines are been used by user space processes.
 - It is possible to read or write to multiple GPIO lines at once.
 - It is possible to find GPIO controllers and GPIO lines by name.
 - It is possible to configure the state of the pin (open-source, open-drain, etc).
 - The polling process to catch events (interrupts from GPIO lines) is reliable.
