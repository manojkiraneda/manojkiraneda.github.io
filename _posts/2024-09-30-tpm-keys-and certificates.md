---
title: "Unlocking the Secrets: Keys and Certificates Inside a TPM "
date: "2024-09-30 10:20AM"
categories: ["TPM", "Security", "OpenBMC"]
tags: ["kernel", "security", "attestation", "tpm", "tpm2-tools", "measurements"]
---

Have you ever wondered what makes a Trusted Platform Module (TPM) tick? Today,
we’re diving into the fascinating world of keys and certificates nestled within
this tiny security powerhouse. But before we embark on this techy adventure,
let’s take a moment to acknowledge the real MVP behind our productivity: mom's
tea.

That's right! It’s not just caffeine that fuels our late-night coding sessions 
and security research—it's the warm, comforting brew that keeps us alert and,
dare I say, a bit more brilliant. So, grab your cup of mom’s finest, and let’s 
unlock the secrets of the TPM together!


> All the commands mentioned in this article are on OpenBMC software running on 
an actual BMC (ast2600) connected to a `Nuvuton NPCT 75xx series 2.0 TPM` via I2C.
The commands are not tested on any other versions other than `OpenSSL 3.3.2`, 
`tpm2-openssl_1.2.0`, `tpm2-tss_4.0.1`, `tpm2-tools_5.5`, `yocto (scarthgap)`
{: .prompt-info}

## Understanding Keys Inside the TPM: A Layman's Guide

The Trusted Platform Module (TPM) is like a security vault inside your computer,
designed to keep sensitive information safe. One of the main things the TPM deals
with are keys — not the kind that unlock doors, but cryptographic keys that help
secure your data and ensure the integrity of your device.

## TPM security architecture
When a key pair, is generated within the TPM, the **`private key remains securely
isolated inside the chip`**, ensuring it cannot be extracted, copied, or accessed
externally. **`Only the public key can be extracted and shared`**, allowing for secure
authentication and verification processes. 

This design not only enhances trust in the cryptographic operations performed by
the TPM but also protects sensitive keys from unauthorized access, even if the 
device itself is compromised. Ultimately, this robust isolation and management 
of keys reinforce the TPM's role as a critical component in secure computing 
environments.

When we talk about trusted computing, most often we hear about two keys :
1. EK (Endorsement key)
2. AK (Attestation key)

## Endorsement Key

The `Endorsement Key (EK)` is one of the most important concepts in Trusted
Computing. It is a cryptographic key that is unique to every device and plays a
key role in establishing trust between a computer or device and other systems.
Its essentially a unique, secret key pair that is embedded into  the device’s
Trusted Platform Module (TPM) when the device is manufactured. It serves as a
proof of the device’s identity and authenticity, kind of like a birth certificate
for the computer's hardware.

Following are the traits for EK :
- **`Unique`**: Every TPM has its own EK, which means no two devices have the same EK.
- **`Permanent`**: Once created, the EK doesn’t change for the lifetime of the device.
- **`Stored Securely`**: The EK is stored deep inside the TPM, where it can’t be accessed or tampered with by normal users or even most software.

> The **Endorsement Key (EK)** is often referred to in two contexts, which can
lead to some confusion between a **key** and a **certificate**.
1. **Endorsement Key (EK) as a Key**:
   - At its core, the EK is a cryptographic key pair, consisting of a **public**
    and a **private key**. This key pair is generated within the TPM and is
    unique to each TPM device. The private part of the EK never leaves the TPM,
    ensuring its security.
2. **EK Certificate**:
   - The **EK certificate** is a **X.509 certificate** that contains the 
    **public portion** of the EK. This certificate is issued by the TPM
    manufacturer and attests to the validity and origin of the EK. The EK 
    certificate links the EK to a trusted authority, ensuring that the public key
    corresponds to a genuine TPM.
The reason the terms "EK" and "EK certificate" are sometimes used interchangeably
is that in many TPM-related operations, the **public EK** (within the certificate)
is what's shared and validated, leading to the conflation of the two. However, it
is important to note that the EK refers to the cryptographic key pair, while the
EK certificate is simply the certificate containing the public key of the EK.
{: .prompt-tip}

### Read and Print EK Certificate

Let's walk through the steps to extract the EK certificate out of TPM:

1. **List the handles** of certificates stored in the TPM.
2. **Read the certificate** data from these handles.
3. **Convert the certificate** into a human-readable format using `openssl`.

#### 1. List Installed Certificate Handles in TPM

First, you’ll use the `tpm2_getcap` command to list the NV index handles that
are available in the TPM. These handles might correspond to the location of the
EK certificate or other data stored in the TPM.

```bash
root@openbmc:~# tpm2_getcap handles-nv-index
- 0x1C00002
- 0x1C00016
```

These handles represent memory locations in the TPM where data (such as
certificates) are stored.

#### 2. Read Data from the NV Index Handle

Once you’ve identified the handle (in this case, `0x01c00002`), you can read the
data from that handle using `tpm2_nvread`. However, the output will be in raw 
binary format, as `tpm2-tools` deals with raw data.

```bash
root@openbmc:~# tpm2_nvread 0x01c00002 -o cert.der
```

This command reads the raw data from the handle `0x01c00002` and stores it in a
file called `cert.der`. The certificate is usually stored in 
**DER (Distinguished Encoding Rules)** format, which is a binary format for
X.509 certificates.

#### 3. Read & Convert and Display the Certificate Using OpenSSL

Now that you have the raw certificate data, you need to convert it into a 
human-readable format. `openssl` can easily handle this conversion. You’ll use
the `x509` command from `openssl` to decode the DER file and print the certificate
in text format:

```bash
root@openbmc:~# openssl x509 -inform der -in cert.der -text -noout

# or else we can merge both the read & print commands together with something
# like this
root@openbmc:~# tpm2_nvread 0x01c00002 | openssl x509 -inform der -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            01:23:45:67:89:ab:cd:ef:00:00:00:00:00:01
        Signature Algorithm: sha256WithRSAEncryption
         Issuer: CN=NuvotonTPMRootCA, O=Nuvoton Technology Corporation, C=TW
        Validity
            Not Before: Sep 30 12:00:00 2024 GMT
            Not After : Sep 30 12:00:00 2034 GMT
        Subject: CN = EK Cert
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus (2048 bit):
                    00:cf:1a:2e:5f:1a:1c:99:25:7d:09:2d:5a:38:
                    2c:9b:eb:22:68:22:e1:00:9c:14:25:14:8c:9c:
                    6d:79:fd:94:4e:1c:07:9f:29:67:45:58:98:21:
                    f7:11:6b:7c:00:09:0b:cd:07:14:11:a0:a5:7c:
                    ...
        X509v3 extensions:
            X509v3 Key Usage: critical
                Key Encipherment
        Signature Algorithm: sha256WithRSAEncryption
             ...
```

Looking at the various fields in the certificate, the key things are :
- **Serial Number** Serial number of the device
- **Issuer:** CN=NuvotonTPMRootCA (Issued by the Nuvuton CA)
- **Key Usage:** Key Encipherment
     - This means the key is intended to be used for encrypting other keys.

## EK uniquely proves the TPM identity 
> The Endorsement Key (EK) is derived from a unique random seed value that is 
permanently embedded within the Trusted Platform Module (TPM) during the
manufacturing process. This seed value is etched into the chip by fusing logic
gates, ensuring its immutability. As a result, no matter how many times the EK 
is generated or utilized, it will consistently yield the same key, which is 
inherently tied to the specific TPM. This uniqueness guarantees that each TPM 
possesses its own distinct EK, reinforcing the foundation of trust and security
in hardware-based cryptographic operations. The design of the EK thus serves as
a crucial element in establishing a reliable and secure identity for the TPM,
enabling robust authentication and attestation mechanisms throughout its 
lifecycle.
{: .prompt-tip}


Well, That's it. Thank you for joining me today.

In my next article, we will delve into the importance of `Attestation Keys (ask)`
and explore their necessity even in the presence of Endorsement Keys (EKs). 

Until then, take care and see you soon!

