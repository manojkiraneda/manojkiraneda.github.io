---
title: "Story of eSPI"
date: "2023-09-26 6:35AM"
categories: ["protocols"]
tags: ["hardware protocols", "hardware spec"]
---

eSPI stands for enhanced serial peripheral interface. The [eSPI base spec](https://www.intel.com/content/dam/support/us/en/documents/software/chipset-software/327432-004_espi_base_specification_rev1.0_cb.pdf) talks about the architectural
details of the eSPI bus interface for both client & server platforms.

The devices that can be supported over the eSPI interface includes but not
necessary limited to Embedded Controller (EC), Baseboard Management Controller (BMC),
Super-I/O (SIO) and Port-80 debug card.

Prior to this specification, Embedded Controller (EC), Baseboard Management
Controller (BMC) and Super I/O (SIO) are connected to the chipset through the Low
Pin Count (LPC) bus.

### eSPI Topology

eSPI hardware protocol supports 4 channels:

1. Pheripheral channel  
2. Virtual wire channel
3. Out-of-band channel
4. Flash Access channel

### Virtual wire channel

This channel allows the eSPI master to claim the General-Purpose I/O (GPIO) pins
physically resided on the eSPI slave side as part of its own virtual I/O pins.

If the Virtual GPIO is configured as an output pin, the eSPI master tunnels the
state of the Virtual GPIO pin through in-band messaging and the eSPI slave, upon
receiving the message, reflects the state on the GPIO pin physically located on
the slave side.

So this interface could leverage the kernel's GPIO subsystem to toggle the GPIO's
which are physically connected on the BMC side(eSPI Slave).

### Out of band Channel (for tunneled SMBus) Messages

The SMBus packets can be tunneled through eSPI as Out-Of-Band(OOB) messages. The
whole SMBus packet is embedded inside the eSPI OOB message as data. So the host
could send SMBus Commands to BMC (eSPI Slave) via the oob channel.

Only SMBus block writes are tunneled through the eSPI OOB message. These `include`
the SMBus Management Component Transport Protocol (MCTP) packets which are
based on the SMBus block write protocol.

Master Attached Flash Sharing (MAFS) :

Master Attached Flash Sharing refers to the scheme where flash components are
attached to the eSPI master such as the chipset. eSPI slaves are allowed to access to
the shared flash component through the Flash Access channel. The flash components
may be on an independent SPI interface, or on a shared SPI/eSPI interface depending
on the system configuration

Slave Attached Flash Sharing
Slave Attached Flash Sharing scheme is defined only for server platforms and it is not
included in the eSPI base specification.
The detail of the Slave Attached Flash Sharing is described in the eSPI addendum for
server platforms.
