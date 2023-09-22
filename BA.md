---
theme: anhembi
title: "Bachelor Thesis: Protected Host Signing Keys for OpenSSH"
_class: lead
paginate: true
# backgroundColor: #fff
# backgroundImage: url('https://marp.app/assets/hero-background.svg')
marp: true
footer: "Finn Callies - 21.09.2023 - Bachelor Thesis: Protected Host Signing Keys for OpenSSH"
---

<!-- _paginate: false -->

# Bachelor Thesis:

## Protected Host Signing Keys for OpenSSH

Finn Callies

---

# Problem Scenario

- Cloud Provider buys IBM Z
- Owns LPARs
- Provides to Clients Hosted VMs
- Client runs a Secure Execution Guest
- Cryptographic Operations without CEX


---

# Problem Scenario

- Cloud Provider buys IBM Z
- Owns LPARs
- Provides to Clients Hosted VMs
- __Client runs a Secure Execution Guest__
    - IBMs Confidential Computing Solution
    - Secures Data in Use
    - Implements a TEE
    &rarr; Secures Guest from Hypervisor
- Securely Execute Cryptographic Operations without CEX


---

# CEX

- IBM HSM Solution
- FIPS 140-2 Level 4 Certified
- Performs or Accelerates certain Cryptographic Operations
    - Accelerator
    - CCA
    - EP11
- stores Master Keys

![bg contain right:50%](architecture.drawio.png)


---

# Cryptographic Keys on IBM Z

| Secure Key                    | Protected Key                 |
|-------------------------------|-------------------------------|
| encrypted by CEX              | encrypted by CPACF            |
| slow cryptographic operations | fast cryptographic operations |
| operations executed on CEX    | operations executed by CPACF  |
| persistent                    | transient                     |
| MK identified by MKVP         | VSMK identified by WKVP       |


---

# OpenSSH

- SSH protocol defined in RFCs
- Successor to RSH Protocol
- OpenSSH implements Standard
- Provides a CLI for Remote Host Management


---

# OpenSSH - Host Authentication

- Host Authentication during Key Exchange
- Signing of Known Data 
- Signing key is one of the host private keys


---

# OpenSSH - Host Key Management

- stored in /etc/ssh/
- private key: ssh_host_\<algorithm\>_key
- public key: ssh_host_\<algorithm\>_key.pub

Example:

    finn@PC-Finn:/etc/ssh$ ls
    ssh_config    ssh_host_dsa_key      ssh_host_ecdsa_key      ssh_host_ed25519_key      ssh_host_rsa_key
    ssh_config.d  ssh_host_dsa_key.pub  ssh_host_ecdsa_key.pub  ssh_host_ed25519_key.pub  ssh_host_rsa_key.pub


---

<!-- _footer: ""-->
# OpenSSH KEX

![bg 80%](kex2.drawio.png)

<!--
PK_OK == Public Key OK
-->


---

<!-- _footer: ""-->
# OpenSSH KEX

![bg 80%](kex.drawio.png)


---

# OpenSSH - Host Key Lifecycle

- Read and Parse Host Key from File
- sshkey_sk Functions 
- sshkey_shield_private
- Sign Verify Check
- __optional__: Parse Public Key from Private Key File
- accumulate_host_timing_secret
- SSH Fingerprint Calculation

<!--
Used for FIDO
-->


---

# OpenSSH - Host Key during KEX

- Private and Public Host Key are loaded from Internal Structure
- Algorithm Negotiation
- Hash is generated which authenticates server
- Signing of calculated Hash

---

# Development - Concept

- Parsing of Private Key
- Skip Functions Irrelevant for Protected Key
    - sshkey_sk
    - sshkey_shield_private
    - Sign Verify Check
    - Parse Public from Private
    - accumulate_host_timing_secret
- Signature Calculation


---

# OpenSSH - Signed Data

| Shared/Agreed Data            | Shared / Agreed                       |
|-------------------------------|---------------------------------------|
| client SSH version            | shared during SSH Version Exchange    |
| server SSH version            | shared during SSH Version Exchange    |
| client random number          | shared during SSH_MSG_KEXINIT         |
| server random number          | shared during SSH_MSG_KEXINIT         |
| client public key             | shared during SSH_MSG_KEXDH_INIT      |
| server public host key        | shared during SSH_MSG_KEXDH_REPLY     |
| shared secret                 | agreed on                             |

<!--
this data is hashed with SHA512 and then SHA256 and then signed
-->


--- 

# Development - SSH Host Key File Format

    finn@PC-Finn:/etc/ssh$ cat ssh_host_ecdsa_key
    -----BEGIN OPENSSH PRIVATE KEY-----
    b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAaAAAABNlY2RzYS
    1zaGEyLW5pc3RwMjU2AAAACG5pc3RwMjU2AAAAQQSjQMKkrqkc+OOYK/C+X+9FeCXP+lhX
    br5ZnytSCYYFRemXvvgd3TarljOxyiyt+MHw6/M31BW+HnwltfCi7o73AAAAqJj8ZrCY/G
    awAAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBKNAwqSuqRz445gr
    8L5f70V4Jc/6WFduvlmfK1IJhgVF6Ze++B3dNquWM7HKLK34wfDr8zfUFb4efCW18KLujv
    cAAAAgfEUJTJoGJWxl39bRnz6P1iCkEiQdePQJ/F8SZaeuFYoAAAAMcm9vdEBQQy1GaW5u
    AQIDBA==
    -----END OPENSSH PRIVATE KEY-----


---

# Development - Detection of Key Format

OpenSSH File Format: 

    -----BEGIN OPENSSH PRIVATE KEY-----
    -----END OPENSSH PRIVATE KEY-----

PEM File Format:

    -----BEGIN ECDSA PRIVATE KEY-----
    -----END ECDSA PRIVATE KEY-----

Protected Key File Format:

    -----BEGIN PROTECTED PRIVATE KEY-----
    -----END PROTECTED PRIVATE KEY-----


---

# Development - Protected Key File Format

Protected Key Object consists of:
- Bitstream of Wrapped Clear Key 
- Wrapping Key Verification Pattern (32 Bytes)

Structure: 

    [root@f37se etc]# cat ssh_host_ecdsa_key.pkey
    -----BEGIN PROTECTED PRIVATE KEY-----
    ( base64 encoded protected key object )
    -----END PROTECTED PRIVATE KEY-----


---

# Development - Parsing Protected Key

## sshkey.h

    struct sshkey {
        ...
        /* KEY_ECDSA and KEY_ECDSA_SK */
        int ecdsa_nid;
        EC_KEY *ecdsa;
        ...
        /* PROTECTED KEY */
        uint8_t *prot_key;
        uint8_t *wkvp;
    };
    #define WKVP_LEN 32


---

# Development - Skip Irrelevant Functions

    if ( sshkey->wkvp == NULL ) {
        // Function to be skipped
    }


---

# Development - ECDSA Protected Key Signature

## ssh-ecdsa.c/ssh-ecdsa-sign(...)

    if ( ( esig = ECDSA_do_sign(digest, hlen, key->ecdsa) ) == NULL ) {
        ret = SSH_ERR_LIBCRYPTO_ERROR;
        goto out;
    }


--- 

# Development - ECDSA Protected Key Signature

## ssh-ecdsa.c/ssh-ecdsa-sign(...)

    if ( key->wkvp != NULL ) {
        if ( ( esig = ecdsa_protected_key_sign(digest, hlen, key->prot_key,
                                        key->wkvp, key->ecdsa_nid) ) == NULL ) {
            ret = SSH_ERR_INTERNAL_ERROR;
            goto out;
        }
    } else if ( ( esig = ECDSA_do_sign(digest, hlen, key->ecdsa) ) == NULL ) {
        ret = SSH_ERR_LIBCRYPTO_ERROR;
        goto out;
    }


--- 

# Development - ECDSA Protected Key Signature

![](param.drawio.png)

<!--
RIBM == Reserved for IBM 
    stores Status and Control Information
CSB stores intermediate results for partial completion
-->


---
<!-- _class: lead -->
<!-- _backgroundImage: "" -->
<!-- _paginate: false -->

# DEMO


---
<!-- _class: lead -->
<!-- _backgroundImage: "" -->
<!-- _paginate: false -->

# Thank you for your attention

## Questions?
