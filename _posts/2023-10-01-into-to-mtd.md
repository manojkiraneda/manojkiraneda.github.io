---
title: "Introduction to mtd interface"
date: "2023-10-01 6:45AM"
categories: ["mtd"]
tags: ["device drivers", "kernel interface", "Linux", "mtd", "flash driver"]
---

`Memory Technology Device` (MTD) is the name of the Linux subsystem that handles
most raw flash devices, such as `NOR`, `NAND`, `dataflash`, and `SPI flash`. It
provides both character and block access to these devices, as well as a number
of specialized filesystems.

## Device support

The following devices are supported by the MTD subsystem:
 - NAND Flash
 - NOR Flash
 - One NAND Flash
 - Atmel Data Flash
 - SPI Flash

The following devices are `NOT` handled by the MTD subsystem, even though they are
commonly referred to as `flash`. Many of these include Flash Translation Layer (FTL)
hardware that takes care of many of the concerns surrounding flash usage,
such as wear-leveling and bad blocks.

 - USB Sticks — handled by USB Host and SCSI subsystems
 - Compact Flash — handled by the PC Card/IDE subsystems, depending on the
   implementation
 - EEPROMs — handled by either the SPI EEPROM driver, or I2C EEPROM driver
 - MMC/SD Cards — handled by the MMC subsystem

## Overview 

The MTD subsystem provides a number of mechanisms for interacting with raw flash
chips. It consists of a number of generic drivers for classes of chips (NAND, NOR),
as well drivers for individual flash controllers. It also includes support for
accessing these chips as either block or character devices, as well as the ability
to divide a single chip into multiple, smaller partitions. Many features of the MTD
subsystem deal with flash-centric issues such as wear-leveling, bad block detection
and handling, and out-of-band (OOB) data. There are a number of filesystems
available specifically for the MTD subsystem.

### Wear Leveling

Wear-leveling is an important consideration when dealing with flash chips. A flash
chip is composed of a large number of memory cells, or gates. Each of these gates
are rated for a limited number of erase/program cycles. This means that they will
eventually wear down, and become "bad blocks." Without wear leveling, some blocks
will wear down much faster than others. For example, consider a file that keeps
track of the total operational time of the board. If we were to update this file
once a minute, and always use the same block when storing the file, that block
would wear down after around 70 days on flash rated for 100,000 erase/program
cycles.
 
Wear-leveling ensures that this burden is spread around to multiple blocks, which
increases the overall life of the chip. MTD itself doesn't perform wear-leveling,
but many of the flash filesystems do.

### Bad Blocks 

One distinguishing feature of NAND flash is that it is allowed to contain bad
blocks straight from the factory. These bad blocks are tracked on the chip so that
software can skip them when necessary. This allows the manufacturers to achieve
a much larger yield, and therefore decrease the costs of the chips. Most (if not
all) NAND flash manufacturers will also guarantee that the first block is good,
allowing you to use that space for a bootloader.

Linux MTD also has facilities for detecting and tracking bad blocks on NAND chips.
Linux generates a bad block table (bbt) and stores this information in the last
two good blocks of the chip. Most Linux-capable bootloaders also treat NAND in
this way. When using a flash-friendly filesystem, or userspace MTD utilities, bad
blocks are automatically handled.


### Is an MTD device a block device or a char device?

First off, an MTD is a "Memory Technology Device", so it's just "MTD". Unix
traditionally only knew block devices and character devices. Character devices
were things like keyboards or mice, that you could read current data from, but
couldn't be seek-ed and didn't have a size. Block devices had a fixed size and
could be seek-ed. They also happened to be organized in blocks of multiple bytes,
usually 512.

Flash doesn't match the description of either block or character devices. They
behave similar to block device, but have differences. For example, block devices
don't distinguish between write and erase operations. Therefore, a special device
type to match flash characteristics was created: `MTD`.

So MTD is neither a block nor a char device. There are translations to use them,
as if they were. But those translations are nowhere near the original, just like
translated Chinese poems. 

