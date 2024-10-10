---
title: "Understanding Attestation: The Role of Endorsement and Attestation Keys"
date: "2024-09-30 14:20PM"
categories: ["TPM", "Security", "OpenBMC", "attestation"]
tags: ["kernel", "security", "attestation", "tpm", "tpm2-tools", "measurements"]
---

In an increasingly interconnected digital world, ensuring the authenticity and
integrity of devices is crucial for maintaining trust and security. One of the
key mechanisms that enable this assurance is **attestation**.

In this article we will explore the concept of attestation, why it is essential,
and the importance of Attestation Keys (ask) in this process, particularly in
relation to the Endorsement Key (EK).

> All the commands mentioned in this article are on OpenBMC software running on 
an actual BMC (ast2600) connected to a `Nuvuton NPCT 75xx series 2.0 TPM` via I2C.
The commands are not tested on any other versions other than `OpenSSL 3.3.2`, 
`tpm2-openssl_1.2.0`, `tpm2-tss_4.0.1`, `tpm2-tools_5.5`, `yocto (scarthgap)`
{: .prompt-info} 


## What is Attestation?

Attestation is the process by which a device proves its authenticity and the 
integrity of its state to another party, often referred to as the verifier. This
process allows the verifier to ensure that the device has not been tampered with
and is operating as intended. Attestation is especially important in environments
where security is paramount, such as in IoT devices, enterprise networks, and
secure communications.

Attestation typically involves the use of cryptographic techniques. The device
generates a report of its current state, which includes values from its Platform
Configuration Registers (PCRs), and signs this report using a private key. The
corresponding public key is then used by the verifier to validate the signature
and confirm the device's identity and state.

## Why Do We Need Attestation?

Attestation serves several critical purposes:

1. **Trust Establishment**: It enables trust between devices and systems. By 
verifying the identity and integrity of devices, organizations can confidently
interact with them.

2. **Security Assurance**: Attestation helps ensure that devices are free from
malware or unauthorized modifications, which is essential for maintaining the
security of systems that depend on these devices.

3. **Compliance**: Many industries have regulatory requirements that necessitate
the use of attestation mechanisms to demonstrate compliance with security
standards.

4. **Remote Verification**: Attestation allows remote parties to verify the state
of a device without needing physical access, which is especially valuable in
distributed environments.

## The Role of Endorsement Keys (EK)

The Endorsement Key (EK) is a unique cryptographic key pair generated within a
Trusted Platform Module (TPM) during manufacturing. The EK serves as the foundation
for attestation, as it provides a way to uniquely identify the TPM and, by
extension, the device it protects. The public part of the EK is often embedded
in an EK certificate, which can be shared with verifiers to establish trust.

While the EK is critical for device identification, it is primarily concerned
with establishing a trust anchor. However, using the EK directly for all
attestation purposes can pose challenges related to privacy and security.

### Why Not Use EK for Attestation?

1. **Privacy Concerns**:
   - **Traceability**: The EK is a permanent identifier tied to a specific TPM and device. Using the EK directly for attestation would allow third parties to trace all attestation interactions back to the specific TPM, potentially compromising user privacy.
   - **Lack of Anonymity**: The EK, if used for attestation, would expose the identity of the TPM in every attestation process. This could lead to unwanted tracking and profiling of devices, undermining privacy.

2. **Static Nature of EK**:
   - The EK is designed to be a static, long-lived key that uniquely identifies the TPM throughout its lifecycle. Using a static key for all attestation processes could lead to security vulnerabilities if the key is ever compromised.
   - In contrast, Attestation Keys (ask) can be generated for specific sessions or applications, allowing for more dynamic and flexible attestation processes.

3. **Key Management**:
   - **Revocation Challenges**: If the EK were compromised, the implications would be severe and complex to manage, as it is a permanent identifier. Revoking or replacing an EK would be difficult, whereas ask can be easily revoked and regenerated without affecting the underlying EK.
   - **Multiple Use Cases**: Different applications might have varying security and privacy needs. Using ask allows for tailored cryptographic operations specific to those needs, improving overall security.

4. **Separation of Roles**:
   - The EK serves as a trust anchor for the device, primarily used for identification. On the other hand, ask are specifically designed for the attestation process, enabling devices to prove their integrity without exposing their identity.
   - This separation allows organizations to adopt best practices in security and cryptographic operations.

5. **Enhanced Security**:
   - Using ask for attestation reduces the exposure of the EK. This minimizes the risk of compromise associated with exposing the device’s unique identifier during attestation processes.
   - ask can be used to sign attestation reports while keeping the EK secure and isolated, ensuring that the integrity and identity of the device remain intact.

## The Importance of Attestation Keys (AK)

To address these challenges, we generate additional key pair called Attestation
Keys (ask). Here are the steps to create an AK :
1. Create a Primary Key
2. Create the Attestation Key (AK)
3. Load the Attestation Key into the TPM
4. Evict Control of the Attestation Key

## Steps to Create an Attestation Key (AK)

### 1. Create a Primary Key

The first step is to create a primary key within the TPM:

```bash
root@openbmc:~# tpm2_createprimary -C e -g sha256 -G rsa -c primary.ctx
name-alg:
  value: sha256
  raw: 0xb
attributes:
  value: fixedtpm|fixedparent|sensitivedataorigin|userwithauth|restricted|decrypt
  raw: 0x30072
type:
  value: rsa
  raw: 0x1
exponent: 65537
bits: 2048
scheme:
  value: null
  raw: 0x10
scheme-halg:
  value: (null)
  raw: 0x0
sym-alg:
  value: aes
  raw: 0x6
sym-mode:
  value: cfb
  raw: 0x43
sym-keybits: 128
rsa: c24903fe60462c7aa2689b0a3c5470a8d838d757616f1ddf43b6a713773da58f137b008fcfb69ca6fa31e1dc7b48fca20093869978f95eced082bb8ce200b6370d14792cf0db87052219f9a7bc0bd450bab66a0a02678e74274a442f2a97f1c0b91232020d09f667a34e0ddaff8a71b59593713e02ead1b796c39cd492ac6243d2ecf85822862372b22643b4cb4e401d73f3025605db235580cfb84febf215b8dc3335092ddcd15904e5ad3565d265ff38dc6ab79bbd7ec46f25d39c9a46a8df92cbb4fd6adcd005c257d64c4784594264cae69b2f15b5a2d7c0d6ab4b73c9937dc2cb743ba20fa0ed79e0bc5f3ff768891d0d3e3f4771892b4592655b930643

```
- **Description**: This command generates a primary key, which serves as a secure anchor for subsequent key creation. 
- **Options**:
  - `-C e`: Specifies that the key will be created under the Endorsement hierarchy.
  - `-g sha256`: Uses SHA-256 as the hash algorithm for key creation.
  - `-G rsa`: Defines the key type as RSA.
  - `-c primary.ctx`: Saves the context of the primary key to a file named `primary.ctx` for future reference.

### 2. Create the Attestation Key (AK)

Next, we create the Attestation Key:

```bash
root@openbmc:~# tpm2_create -C primary.ctx -G rsa -u aik.pub -r aik.priv
name-alg:
  value: sha256
  raw: 0xb
attributes:
  value: fixedtpm|fixedparent|sensitivedataorigin|userwithauth|decrypt|sign
  raw: 0x60072
type:
  value: rsa
  raw: 0x1
exponent: 65537
bits: 2048
scheme:
  value: null
  raw: 0x10
scheme-halg:
  value: (null)
  raw: 0x0
sym-alg:
  value: null
  raw: 0x10
sym-mode:
  value: (null)
  raw: 0x0
sym-keybits: 0
rsa: b03c6b9ed2cd9046ca999ec7829e5aff2b4ea9ab06d9c33d232c7a74efebff796ada23b4fca87c051e8f0840d7a759fef649b8a266dae478d072787fc38d9206120207bc3268ab02d08f9520a8941bf2789aa3e29cacac075a1f474647cc13a64b408732032697276534b52dd66f1d79674080f43c0190ab10c0b851f8076a75a123bbeb6f010a9d90e3b7a633858b9b627dd83fa89e23aef9c14e4f4844e5981ea83b12d501f842b07676a26a94bbde8915fce0069cca6391fc89bf776fe5bd224bdcf44aa8af488962ac4ab1cf287b498b16befb8bb78c1d2177d697b9637428fbab2fbfc83f8e95742c601d5b1fce8656148517e5154fb939791acaa4338f
```

- **Description**: This command generates a new RSA key pair, designated as the Attestation Key (AK), under the previously created primary key.
- **Options**:
  - `-C primary.ctx`: Uses the context of the primary key to establish a secure environment for creating the AK.
  - `-G rsa`: Specifies that the key type is RSA.
  - `-u aik.pub`: Outputs the public portion of the AK to a file named `aik.pub`.
  - `-r aik.priv`: Outputs the private portion of the AK to a file named `aik.priv`.

### 3. Load the Attestation Key into the TPM

The next step involves loading the AK into the TPM:

```bash
root@openbmc:~# tpm2_load -C primary.ctx -u aik.pub -r aik.priv -c aik.ctx
name: 000bc01cd2da1b7fbf94e0284b1e37b289d87befa8f7aa2d20e5867595e93f675575
```

- **Description**: This command loads the AK into the TPM, making it available for attestation operations.
- **Options**:
  - `-C primary.ctx`: References the context of the primary key under which the AK is being loaded.
  - `-u aik.pub`: Supplies the public part of the AK for loading.
  - `-r aik.priv`: Supplies the private part of the AK for loading.
  - `-c aik.ctx`: Saves the context of the loaded AK to a file named `aik.ctx`.

### 4. Evict Control of the Attestation Key

To manage the AK, we evict its control:

```bash
root@openbmc:~# tpm2_evictcontrol -C o -c aik.ctx 0x81000008
persistent-handle: 0x81000008
action: persisted
```

- **Description**: This command allows the AK to be controlled outside the TPM, facilitating its use in various applications.
- **Options**:
  - `-C o`: Specifies that the operation is performed under the Owner hierarchy.
  - `-c aik.ctx`: References the context of the AK being evicted.
  - `0x81000008`: Assigns a persistent handle to the AK, allowing it to be used in future operations.


## Generate a Quote of Measurements

As discussed in the [measurements article](https://manojkiraneda.github.io/posts/tpm-measurements-in-openbmc/),
U-Boot records measurements in the SHA256 bank of Platform Configuration Registers
(PCRs), specifically PCRs 0, 1, 8, and 9. Utilizing the Attestation Key (AK), we
will now create a measurements file and sign it, a process known as generating
a Quote.

```bash
root@openbmc:~# tpm2_quote -c aik.ctx -l sha256:0,1,8,9 -q abc123 -m quote.msg -s quote.sig -o quote.pcrs -g sha256
quoted: ff54434780180022000b0d322a7f0856e203f19521a6f47162bba4d7bf3bd35d00e6092dd5120e1d0d340003abc1230000000629d262b6000001610000000001000700020003000000000001000b03030300002021b047d228a2bd09f3a986280f75131b3d56904841d2bd1fcd8e5ebce1f11027
signature:
  alg: rsassa
  sig: 982daac0b909400c5d1f38fa0c739303b4ec4192bd187d411603a0daad6cab0054a3fd6ab6f266cbd878010573e1e7eb88cb4798f800fe8c3c436bfa2306725cfedcbb559cf436b286f537ccf6f9c6115b8be5a58f1de11210c2d9a92f846188265350b15b2ae44525e34ae511455f2b846f9cf39cb15cee02fc1aee4082289b738acea95af1e918aa440d32ea4c521742dc416c48f7f860a5d758705994a722d282c8c6bd6f5b0abd919b97e51c83e1a99615df6a5a1b81169e15b9b0bbc083cb220c793294a09aa966e131573e659b132ccaac3a46de809024dfd5b584cfba78696451561bbf65d94f39b8013c898dd75d5ba04dd8e9ca2e225d8ba3e5f5cc
pcrs:
  sha256:
    0 : 0x5579389080DA827D061B08067E85134E084272DFEBC1A16DC34767E72A5AFEC9
    1 : 0x91E4E7BF88FF38DF16FDB83E7DAE1E8E705615B46A4A29C5FCAE27C8F7B7C5A8
    8 : 0x952F09E0B212048A5C868256A39D792ADD79A220552DDDCE23E002A13D265AF7
    9 : 0x75CD0299D004D06FF04053D20C1126DA196C20CFCA202A8EFE70482ECB557C5F
calcDigest: 21b047d228a2bd09f3a986280f75131b3d56904841d2bd1fcd8e5ebce1f11027

```

- **Description**: This command creates a cryptographic quote containing measurements from specified Platform Configuration Registers (PCRs), demonstrating the integrity of the device's state.
- **Options**:
  - `-c aik.ctx`: References the context of the AK used to sign the quote.
  - `-l sha256:0,1,8,9`: Specifies the PCRs to include in the quote.
  - `-q abc123`: Provides a nonce (random value) for the quote to ensure freshness.
  - `-m quote.msg`: Saves the signed quote message to a file named `quote.msg`.
  - `-s quote.sig`: Saves the signature of the quote to a file named `quote.sig`.
  - `-o quote.pcrs`: Outputs the PCR values used in the quote to a file named `quote.pcrs`.
  - `-g sha256`: Specifies the hash algorithm used (SHA-256).

## Verify the Quote

Finally, we verify the generated quote (this usually happens in a different
machine where the the signed quote along with the public key & the signature along
with the random nonce gets copied to):

```bash
root@openbmc:~# tpm2_checkquote -u aik.pub -m quote.msg -s quote.sig -f quote.pcrs -g sha256 -q abc123
pcrs:
  sha256:
    0 : 0x5579389080DA827D061B08067E85134E084272DFEBC1A16DC34767E72A5AFEC9
    1 : 0x91E4E7BF88FF38DF16FDB83E7DAE1E8E705615B46A4A29C5FCAE27C8F7B7C5A8
    8 : 0x952F09E0B212048A5C868256A39D792ADD79A220552DDDCE23E002A13D265AF7
    9 : 0x75CD0299D004D06FF04053D20C1126DA196C20CFCA202A8EFE70482ECB557C5F
sig: 982daac0b909400c5d1f38fa0c739303b4ec4192bd187d411603a0daad6cab0054a3fd6ab6f266cbd878010573e1e7eb88cb4798f800fe8c3c436bfa2306725cfedcbb559cf436b286f537ccf6f9c6115b8be5a58f1de11210c2d9a92f846188265350b15b2ae44525e34ae511455f2b846f9cf39cb15cee02fc1aee4082289b738acea95af1e918aa440d32ea4c521742dc416c48f7f860a5d758705994a722d282c8c6bd6f5b0abd919b97e51c83e1a99615df6a5a1b81169e15b9b0bbc083cb220c793294a09aa966e131573e659b132ccaac3a46de809024dfd5b584cfba78696451561bbf65d94f39b8013c898dd75d5ba04dd8e9ca2e225d8ba3e5f5cc

## Lets give a different nonce & observe that the verification fails
root@openbmc:~# tpm2_checkquote -u aik.pub -m quote.msg -s quote.sig -f quote.pcrs -g sha256 -q abc12345
pcrs:
  sha256:
    0 : 0x5579389080DA827D061B08067E85134E084272DFEBC1A16DC34767E72A5AFEC9
    1 : 0x91E4E7BF88FF38DF16FDB83E7DAE1E8E705615B46A4A29C5FCAE27C8F7B7C5A8
    8 : 0x952F09E0B212048A5C868256A39D792ADD79A220552DDDCE23E002A13D265AF7
    9 : 0x75CD0299D004D06FF04053D20C1126DA196C20CFCA202A8EFE70482ECB557C5F
sig: 982daac0b909400c5d1f38fa0c739303b4ec4192bd187d411603a0daad6cab0054a3fd6ab6f266cbd878010573e1e7eb88cb4798f800fe8c3c436bfa2306725cfedcbb559cf436b286f537ccf6f9c6115b8be5a58f1de11210c2d9a92f846188265350b15b2ae44525e34ae511455f2b846f9cf39cb15cee02fc1aee4082289b738acea95af1e918aa440d32ea4c521742dc416c48f7f860a5d758705994a722d282c8c6bd6f5b0abd919b97e51c83e1a99615df6a5a1b81169e15b9b0bbc083cb220c793294a09aa966e131573e659b132ccaac3a46de809024dfd5b584cfba78696451561bbf65d94f39b8013c898dd75d5ba04dd8e9ca2e225d8ba3e5f5cc
ERROR: Error validating nonce from quote
ERROR: Verify signature failed!
ERROR: Unable to run tpm2_checkquote

```

- **Description**: This command verifies the integrity of the quote against the expected values and ensures it was generated by the AK.
- **Options**:
  - `-u aik.pem`: Uses the public portion of the AK, which must be in PEM format, for verification.
  - `-m quote.msg`: Specifies the file containing the signed quote message to verify.
  - `-s quote.sig`: Specifies the file containing the signature of the quote.
  - `-f quote.pcrs`: References the file containing the PCR values used in the quote.
  - `-g sha256`: Specifies the hash algorithm for verification (SHA-256).
  - `-q abc123`: Supplies the nonce used during the quote generation for freshness.

Once the quote verification is successful, then we can safely assure that the
device mesaurements that are reported by the TPM under test are indeed generated
by the device itself and is not really fabricated by eavesdropper.


## Comparing Quoted PCR Values to the Reference Manifest

After generating a Quote with the measurements from the Platform Configuration
Registers (PCRs), the next step is to compare these values to a reference manifest.
This helps determine if the system is trustworthy.

1. **Verify Quoted PCR Values**: First, ensure that the PCR values from the Quote
are legitimate and have not been altered. This step confirms that the values come
from the Attestation Key and are authentic.

2. **Reference Manifest**: This is a trusted document that lists the expected PCR
values for a secure system. It serves as a benchmark for comparison.

3. **Comparison**: Check each quoted PCR value against the values in the reference manifest.
Any differences can indicate potential issues or unauthorized changes in the system.

4. **Decision Making**:ß
   - **If they match**: The system is considered secure and can continue operating normally.
   - **If they don’t match**: This suggests that something might be wrong. It could mean the system has been compromised, and further investigation is needed. In such cases, actions like stopping operations or alerting security teams may be necessary.








