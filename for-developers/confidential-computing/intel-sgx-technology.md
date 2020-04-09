# Overview

**Confidential Computing** \(or **Trusted Execution Environments -** **'TEE'**\) ensures computation confidentiality through mechanisms of memory encryption at the hardware level. Applications being executed and data being processed are safeguarded against even the most privileged access levels \(OS, Hypervisor...\). Only authorized code can run inside this protected area and manipulate its data.

In some cases, ensuring that code runs correctly without any third party altering the execution, is even more important than hiding the computation's data. This concept is called **Trusted Computing.**

These guarantees are critical for a decentralized cloud where code is being executed on a remote machine, that is not controlled by the requester. They are also required to prevent leakage while monetizing data sets.

## Intel® Software Guard Extension \(Intel® SGX\)

[Intel® SGX](https://software.intel.com/en-us/sgx) is a technology that enables **Trusted** and **Confidential Computing**. At its core, it relies on the creation of a special zone in the memory called an “enclave”. This enclave can be considered as a vault, to which only the CPU can have access. Neither privileged access-levels such as root, nor the operating system itself is capable of inspecting the content of this region. The code, as well as the data inside the protected zone, is totally unreadable and unalterable from the outside. This guarantees non-disclosure of data as well as tamper-proof execution of the code.

An application's code can be separated into "trusted" and "untrusted" parts where sensitive data is manipulated inside the protected area.

### Enclave

In confidential computing jargon, an "enclave" is the special memory zone protected by the CPU. For simplicity's sake, we can refer to private regions of memory defined by Intel® SGX application as "enclaves".

### Remote attestation

As explained by [Intel](https://software.intel.com/en-us/sgx/attestation-services), the remote attestation is the process that happens before any exchange between a remote provider and an enclave. It allows the provider to verify that the expected software is running in an Intel® SGX-protected way, as well as getting other details about the application being attested. If the attestation is successful, a secure communication channel is established between the provider and the enclave, and secrets can safely land in the latter.

## Confidential Computing with iExec

At iExec we do not develop Intel® SGX frameworks, but rather we integrate state of the art solutions developed by other specialists. Currently, the SCONE framework has been integrated into our platform. We are actively working on supporting [Graphene](https://grapheneproject.io/).

### SCONE Framework

Kernel services and system calls are not available from an Intel® SGX enclave as the OS is not a part of the trusted computing base \(TCB\) in Intel® SGX. This can be limiting as your application will not be able to use sockets or the file system directly from code running inside the enclave. To resolve this, and reduce the burden of porting your application to Intel® SGX,  we can use use the [SCONE](https://scontain.com/) framework.

At a high level SCONE provides a C standard library interface to container processes. System calls are executed outside of the enclave, but they are shielded by transparently encrypting/decrypting application data: files stored outside of the enclave are therefore encrypted, and network communication is protected by transport layer security \(TLS\). With SCONE you can make your application compatible with Intel® SGX without modifying the source code. You need just to prepare your application's docker image based on on of the [curated list](https://sconedocs.github.io/SCONE_Curated_Images/) of images provided by SCONE.

### Terminology

#### MrEnclave: \(In short; It is the id of an enclave\)

The [MrEnclave](https://sconedocs.github.io/MrEnclave/) is a hash value that identifies every enclave. It is obtained from the content of memory pages and access rights. After you build your SCONE app, you will get its fingerprint, the MrEnclave is the last part of that fingerprint.

{% hint style="info" %}
Please note that some of SCONE's [environment variables](https://sconedocs.github.io/SCONE_ENV/) such as **SCONE\_HEAP** can affect the value of the MrEnclave.
{% endhint %}

#### FSPF \(File System Protection File\)

In addition to identifying the code, SCONE, also, takes a snapshot of the file system state. This guarantees that we cannot alter the enclave by modifying its files' state. To do this, scone uses two parameters the [FSPF\_KEY](https://sconedocs.github.io/SCONE_Fileshield/#file-system-protection-file) and [FSPF\_TAG](https://sconedocs.github.io/SCONE_Fileshield/#file-system-protection-file).

#### Application's fingerprint

It is the concatenation of the MrEncalve, the FSPF\_KEY and the FSPF\_TAG separated by a "\|". You should use this when deploying your application on the Blockchain. The SMS uses this as a reference to evaluate the state of client enclaves and whether they should get secrets or not.

### Secret Management Service \(SMS\)

With the integration of SCONE in iExec, you do not need to worry about [remote attestation](intel-sgx-technology.md#remote-attestation). We do that for you, we guarantee that the code is running inside an enclave. But that is not all, we also verify that the enclave asking for secrets is authorized to do so. Hence, we implemented a component to handle the permission management for those secrets. You guessed it, it is the SMS! The SMS queries the blockchain an determines, for each task, the required secrets and provisions them on the fly.

Unquestionably, the SMS is a critical component. That's why its runs inside an enclave.

### How does it work?

{% hint style="info" %}
For more information about SCONE, please refer to their documentation at [https://sconedocs.github.io](https://sconedocs.github.io/).
{% endhint %}

We explain the process of how to make your trusted application using iExec in details in the [next chapter](create-your-first-sgx-app.md). Here is a quick general overview:

**Trusted application:** First things first, choose a base docker image for your use case. We provide a template Dockerfile so you would, just, add your specific requirements and dependencies, then build you image. Push you docker image somewhere accessible and deploy your application on the blockchain with the correct image URI and fingerprint.

**Confidential dataset:** To make your dataset available on iExec, you should first encrypt it using the SDK, before making the encrypted file publicly available. Deploy your dataset on the blockchain, then, push the encryption key into the SMS where it is securely saved \(protected by an enclave, which means even us we cannot access it\). Only applications you have authorised can get this key.

