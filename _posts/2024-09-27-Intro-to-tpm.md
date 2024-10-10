---
title: "TPM 101: A Beginner's Guide to Trusted Platform Modules"
date: "2024-09-27 04:20PM"
categories: ["TPM", "Security"]
tags: ["kernel", "security", "attestation", "tpm", "tpm2-tools"]
---

During my extended absence from posting, I’ve been immersing myself in some
fascinating tech, particularly those related to `Trusted Platform Modules (TPM)`.
While there are countless resources available online that discuss TPM in depth,
this could be considered as a very basic elementary guide to highlight the impressive
capabilities of TPM and explore its significance in the context of OpenBMC.

## Root of Trust (RoT)

We often hear the term **"Core Root of Trust"** when talking about security. It 
refers to the foundational layer of security upon which all other security mechanisms
are built. To understand this better, let’s break it down a bit :

1. **Root of Trust (RoT)**: 
   The "Root of Trust" is a secure and trusted component in a system that
   performs critical security operations, such as cryptographic key generation,
   authentication, or validation. It's considered the starting point of trust in
   the security architecture because every other security mechanism depends on
   the trustworthiness of this root.

2. **Core Root of Trust**: 
   In many secure systems, the "Core Root of Trust" refers to the most fundamental
   piece of hardware or firmware that is responsible for initializing the chain
   of trust when the system boots up. This is usually immutable and tightly
   protected from tampering. 

In practical terms, this means that the **Core Root of Trust** is the first
piece of code or hardware that is executed and trusted when a system starts. If 
the Core Root of Trust is secure, it ensures the integrity and security of
everything else in the system as it boots, layer by layer.

## Trusted Platform Module

A Trusted Platform Module (TPM) is a small chip on your `computer's motherboard`
that helps protect your data and ensure security.

The TPM helps with tasks like:
- **Encrypting files** so that only you can open them.
- **Checking that your computer's software** hasn't been tampered with when you start it.
- **Storing sensitive info**, like login credentials, securely.

Basically, it’s a security guard that protects your data and makes sure your
computer is safe and trustworthy.

## What does RoT mean in the OpenBMC Context 

OpenBMC is the open-source firmware used for managing systems (like servers) 
at a low level, often in data centers. In this context, the **Core Root of Trust**
is essential for establishing a secure boot process. It ensures that the BMC
(Baseboard Management Controller) firmware is not tampered with and starts up in
a trusted state.

For example, a Trusted Platform Module (TPM) or other hardware security modules 
can be part of the Core Root of Trust in an OpenBMC system. These modules will
perform cryptographic checks, ensuring that only authentic and trusted firmware 
is loaded during the boot process, which prevents malicious firmware from 
compromising the system.

## Types of TPM's

In many cases, especially in modern systems, the TPM is either soldered directly
onto the motherboard as a dedicated chip or built into the chipset. There are two
main types of TPM implementations:

1. **Discrete TPM** : A physically separate TPM chip installed on the motherboard.
A discrete TPM chip is typically connected to a system using one of the following
interfaces, depending on the design of the hardware and platform:
   - **LPC (Low Pin Count) Bus**:
Most common in traditional systems: The TPM is often connected via the LPC bus,
which is a simple and widely-used low-bandwidth bus for attaching peripherals to
the motherboard.
   - **SPI (Serial Peripheral Interface)**:
Increasingly popular in modern systems: Many newer systems, including some modern
server and embedded platforms, use SPI to connect the TPM. SPI is faster than LPC
and supports more modern device integration.
   - **I²C (Inter-Integrated Circuit)**:
Some embedded systems use the I²C interface for TPMs.
   - **PCIe (Peripheral Component Interconnect Express)**:
Specialized or discrete TPMs: Some TPM chips come as PCIe devices, typically as
discrete, add-in cards. These are often used in high-security environments where
the TPM is implemented as a separate physical card, as opposed to being integrated
into the motherboard.

2. **Firmware TPM (fTPM)** : A TPM that exists in firmware, running as part of
the system’s main processor, often within the trusted execution environment (TEE)
or ARM's TrustZone. While not a separate chip, it serves the same function as a
hardware TPM but is more integrated into the system architecture.

Finally gist is that, `TPM is like a safe in your house`. Inside this safe, you
can store important keys (like passwords or encryption keys) that only the TPM
can use. Even if someone breaks into your computer, they can't access what's
inside the safe (unless some one spends a billion dollars & and use sophisticated
hardware tools to break it)

> [TPM Genie](https://github.com/nccgroup/TPMGenie) is designed to aid in
vulnerability research of Trusted Platform Modules. As a serial bus interposer,
TPM Genie is capable of intercepting and modifying all traffic that is sent
across the I2C channel (this works only when the discrete TPM chip is connected
via the I2C bus) between the host machine and a discrete TPM chip.
{: .prompt-info} 

## Why do we need TPM's ? 

In the current day tech, TPM's are being used in various places to solve various
kinds of problems. But majority of the use-cases are basically driven from these
3 primary problems.

##### **Easier to steal sensitive data**
Without a TPM, if someone gets access to your computer, they could potentially
steal important information like your passwords or encryption keys. This is like
someone getting into your house and easily finding your hidden spare key (no
matter how much secretly you store it)

> The TPM securely locks these important keys and sensitive data inside a special
chip. Even if someone breaks into your computer, they can’t access that data
without the TPM unlocking it. It’s like storing the spare key in a safe that
only you can open.
{: .prompt-tip }

##### **Harder to detect tampered software**
When you boot up a computer without a TPM, it might not detect if someone has
tampered with your operating system or installed malware. This can lead to hidden
attacks that run unnoticed.

> A TPM checks the integrity of your system when you turn it on. If it detects any
unauthorized changes (like tampered software or malware), it can stop the
computer from booting or warn you. It’s like having an alarm system that tells
you if someone messed with your house while you were away.
{: .prompt-tip }

##### **Encryption is less secure**
Without a TPM, encryption (which scrambles your data to protect it) relies on
software or your password. If a hacker gets access to your computer, they could
potentially bypass this and decrypt your files.

> The TPM holds the encryption keys safely inside its chip, making it much harder
for anyone to decrypt your files without permission. Even if they take your hard
drive, without the TPM, they can’t unlock the encrypted data. It’s like locking
a treasure chest and hiding the key inside an unbreakable vault.
{: .prompt-tip }

Okay , so with all the above context - Lets try and under how a presence of TPM
can be detected from userspace linux.

## Enabling TPM on Desktop software

Now a days, every desktop/Laptop comes with a TPM, it can be either a discrete
one or a Firmware TPM. To use TPM 2.0 on a desktop software with x86 CPU hardware,
the first step is to enable it in your PC's firmware settings (commonly known as
UEFI, which has replaced traditional BIOS). The exact procedure for doing this
will vary depending on your motherboard, BIOS version, and the specific TPM module
you're using, so it's best to refer to your motherboard's manual for detailed
instructions.

## Enabling TPM on OpenBMC software

OpenBMC runs on linux, and we need to ensure that the kernel config to enable the
TPM and probing the TPM hardware is selected.

```bash
root@openbmc:~# dmesg | grep tpm
[    1.580176] tpm_tis_i2c 12-002e: 2.0 TPM (device-id 0xFC, rev-id 1)
```

This log output from `dmesg` indicates that the system has successfully detected
a TPM (Trusted Platform Module) device connected via the **I²C bus**.
Here's a breakdown of the key components in the above message:

1. **`tpm_tis_i2c 12-002e`**:
   - This indicates that the **TPM** is being accessed using the **TPM TIS I²C driver**.
     The **TPM TIS** (TPM Interface Specification) is a standard protocol used for communication between the operating system and TPM.
   - **`12-002e`**: Refers to the address of the TPM device on the I²C bus. I²C devices are addressed with 7-bit addresses, and `0x2e` is the hexadecimal address where the TPM is located on bus number `12`.

2. **`2.0 TPM`**:
   - This shows that the detected TPM module supports **TPM 2.0**, which is the latest version of the Trusted Platform Module specification.

3. **`(device-id 0xFC, rev-id 1)`**:
   - **`device-id 0xFC`**: This is the **device identifier** of the TPM module. It can vary based on the manufacturer and specific TPM model.
   - **`rev-id 1`**: This indicates the **revision ID** of the TPM hardware. It's a version number that helps to identify the specific revision of the TPM module.

```bash
root@openbmc:~# ls -l /dev/tpm*
crw-rw----    1 root     root       10, 224 Sep 26 03:46 /dev/tpm0
crw-rw----    1 root     root      251, 65536 Sep 26 03:46 /dev/tpmrm0
```

This output from your **OpenBMC system** shows two **TPM-related device files**:
 `/dev/tpm0` and `/dev/tpmrm0`. Here's what each of these device nodes represents:

1. **`/dev/tpm0`**:
   - This is the main **TPM device file**. It represents the standard interface
    for communicating with the **TPM hardware** on your system. Applications or 
    drivers use this device to interact with the TPM, send commands, and retrieve
    responses from the chip.
   - **Major number 10, minor number 224**:
     - The numbers (`10, 224`) represent the **major** and **minor device numbers**. These are used by the Linux kernel to map the device file to the actual driver and hardware interface.
   - **Permissions**: `crw-rw----`:
     - This indicates it's a **character device** (`c`) with **read/write permissions** for **root** and **members of the root group**.

2. **`/dev/tpmrm0`**:
   - This device file represents a **TPM resource manager** interface (`tpmrm0`).
   The TPM Resource Manager is responsible for managing multiple applications or
   processes that may want to use the TPM concurrently. The TPM itself is typically
   a single-threaded device, so it cannot handle multiple requests at the same time.
   The resource manager helps virtualize access to the TPM by managing contexts and
   session handles.

In most modern Linux systems, applications will use `/dev/tpmrm0` to avoid
conflicts, as it provides a more robust and scalable way to handle TPM 
interactions.

> [tpm2-abrmd](https://github.com/tpm2-software/tpm2-abrmd) : TPM2 Access Broker
and Resource Manager was a user-space daemon designed to manage concurrent
access to the TPM 2.0 by different applications. It acted as a broker and resource
manager, ensuring that multiple applications could use the TPM without conflicts.
However, **`tpm2-abrmd` has been deprecated** because the in-kernel resource manager
(`/dev/tpmrm0`) offers similar functionality with better performance and lower overhead.
Applications should now use the kernel's built-in TPM resource manager via `/dev/tpmrm0`
for managing TPM 2.0 sessions and concurrent access, instead of relying on `tpm2-abrmd`.
{: .prompt-danger} 

```bash
root@openbmc:~# cat /sys/class/tpm/tpm0/tpm_version_major
2
```
The output of the command `cat /sys/class/tpm/tpm0/tpm_version_major` shows the
major version of the TPM (Trusted Platform Module) installed on your system. In
this case, it indicates that the system is using **TPM 2.0**, which is the latest
major version of the TPM specification.

- **TPM 2.0**: This version provides enhanced security features, broader 
cryptographic algorithm support, and improved functionality compared to TPM 1.2.
It is widely used in modern systems for tasks like secure boot, disk encryption,
and attestation.

## What is the TPM Software Stack (TSS)

The TPM Software Stack (TSS) is a set of software libraries and interfaces that
enable communication with the Trusted Platform Module (TPM 2.0). The TSS is based
on the Trusted Computing Group (TCG) specifications and provides developers with
a consistent, high-level API for interacting with TPMs. It defines how software
components can access and manage TPM hardware, and it abstracts many low-level
details, allowing developers to focus on building secure applications without
worrying about direct TPM communication.

The TSS stack is divided into multiple layers:
- `Enhanced System API (ESAPI)`: High-level API for managing TPM commands.
- `System API (SAPI)`: Lower-level API for interacting with the TPM.
- `TPM Command Transmission Interface (TCTI)`: Handles the communication between the system and the TPM.
- `Resource Manager`: Manages multiple applications' concurrent access to the TPM.

## Available OpenSource TSS Stacks

1. [**IBM TPM 2.0 TSS**](https://github.com/kgoldman/ibmtss) :
IBM’s TPM 2.0 TSS is an implementation of the Trusted Computing Group (TCG) TPM
2.0 Software Stack. This stack is provided by IBM and offers a complete
implementation of the TCG's TSS for TPM 2.0, including SAPI (System API), ESAPI
(Enhanced System API), and TCTI (Transport Layer).

2. [**tpm2-tss**](https://github.com/tpm2-software/tpm2-tss) :
This is the official TCG-compliant implementation of the TPM 2.0 Software Stack
(TSS), maintained by the tpm2-software community. It provides a full set of C
libraries that follow the TCG standards for TPM interaction. It offers APIs that
developers can use to manage TPM resources, keys, and cryptographic functions.

> I tried using both the above stacks & I personally liked the tools that are
provided in the [tpm2-tools](https://github.com/tpm2-software/tpm2-tools) repository
are more intuitive & easy to use, than the ones that are present as part of the
`ibm-tss`. So all my articles would only use `tpm2-tool` commands going forward.
{: .prompt-tip} 

## Add necessary userspace TPM support to OpenBMC rootfs
[openbmc/openbmc](https://github.com/openbmc/openbmc/tree/master/meta-security/meta-tpm)
already has all the recipes defined for packages like `tpm2-tss`, `tpm2-tools` 
e.t.c & they are all part of the `meta-security/meta-tpm` layer. All we need to 
do is just pull them into the image by adding below statement in the 
`build/<MACHINE>/conf/local.conf` file & we are good to go

```bash
root@openbmc:~# . setup <machine>
root@openbmc:~# vim build/<machine>/conf/local.conf

## add below line
IMAGE_INSTALL:append = " tpm2-tools tpm2-openssl tpm2-tss libtss2-tcti-device"

root@openbmc:~# bitbake obmc-phosphor-image
```

## Fun with TPM

Now comes the most interesting part! , how do we make TPM do some thing ?

Here are some basic **`tpm2-tools`** commands and explanations to help you
understand the TPM, its manufacturer, capabilities, and more:

##### **Check TPM Manufacturer Information**
   - **Command**: `tpm2_getcap properties-fixed`

This command retrieves fixed properties of the TPM, which include the manufacturer
ID, firmware version, and other details. This is useful for understanding the vendor
and specific version of your TPM.
```bash
root@openbmc:~# tpm2_getcap properties-fixed
TPM2_PT_FAMILY_INDICATOR:
  raw: 0x322E3000
  value: "2.0"
TPM2_PT_MANUFACTURER:
  raw: 0x4E544300
  value: "NTC"
TPM2_PT_VENDOR_STRING_1:
  raw: 0x4E504354
  value: "NPCT"
TPM2_PT_VENDOR_STRING_2:
  raw: 0x37357800
  value: "75x"
......
```
Above output shows that i am using a `Nuvoton NPCT 75xx series` TPM `2.0` chip.

##### **List All TPM Capabilities**
   - **Command**: `tpm2_getcap -l`

This command lists all the categories of capabilities that the TPM can provide information on. You can use these categories with `tpm2_getcap` to dig deeper into specific aspects of your TPM.

```bash
root@openbmc:~# tpm2_getcap -l
- algorithms
- commands
- pcrs
- properties-fixed
- properties-variable
- ecc-curves
- handles-transient
- handles-persistent
- handles-permanent
- handles-pcr
- handles-nv-index
- handles-loaded-session
- handles-saved-session
- vendor
```

##### **View Supported Algorithms**
   - **Command**: `tpm2_getcap algorithms`

This lists all the cryptographic algorithms supported by the TPM, such as RSA, ECC, SHA-256, and more. This is helpful when you need to understand which algorithms your TPM can use.

```bash
root@openbmc:~# tpm2_getcap algorithms
rsa:
  value:      0x1
  asymmetric: 1
  symmetric:  0
  hash:       0
  object:     1
  reserved:   0x0
  signing:    0
  encrypting: 0
  method:     0
sha1:
  value:      0x4
  asymmetric: 0
  symmetric:  0
  hash:       1
  object:     0
  reserved:   0x0
  signing:    0
  encrypting: 0
  method:     0
sha256:
  value:      0xB
  asymmetric: 0
  symmetric:  0
  hash:       1
  object:     0
  reserved:   0x0
  signing:    0
  encrypting: 0
  method:     0
aes:
  value:      0x6
  asymmetric: 0
  symmetric:  1
  hash:       0
  object:     0
  reserved:   0x0
  signing:    0
  encrypting: 0
  method:     0
```
     
##### **Generate a Random Number**
   - **Command**: `tpm2_getrandom 8`

This generates 8 bytes of random data from the TPM's hardware-based random number
generator. The TPM is often used to generate secure random numbers, useful for
cryptographic operations.
```bash
root@openbmc:~# tpm2_getrandom -o random.out 8
root@openbmc:~# hexdump -C random.out
00000000  7d 66 5f f2 5a 8d 4b 82                           |}f_.Z.K.|
00000008
```

The fact that a command like `tpm2_getrandom` taps directly into the 
**TPM's hardware-based random number generator** is quite fascinating. Unlike
software-based random number generation, which can be predictable or influenced
by external factors, the **TPM** uses specialized circuitry designed to generate
truly unpredictable, high-entropy numbers. This ensures a much higher level of
security, as the random numbers produced by the TPM are inherently resistant to
many forms of attack.

> **Source of true randomness**: A TPM2.0 has at least one internal source of entropy.
These sources can include noise, clock variations, air movement amongst other events.
Look at [this article](https://developers.tpm.dev/posts/random-number-generator-tpm2-12528972)
to understand how tpm2.0 hardware number generator works.
{: .prompt-tip}

## The End 

It’s remarkable how seamlessly we can interact with such powerful hardware through
simple commands, leveraging **cutting-edge cryptography** to enhance the security
of modern systems. The ability to access **hardware-backed randomness** with 
ease—directly from a **Trusted Platform Module (TPM)**—is a testament to how far
we've come from where we have began.

Isn't it pretty cool !!
