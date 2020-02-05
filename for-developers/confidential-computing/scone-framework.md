# Confidential Computing with iExec

## SCONE Framework

Kernel services and system calls are not available from an Intel® SGX enclave. This is often severely limiting as your application will not be able to use sockets or the file system directly from code running inside the enclave. One solution to get around this and reduce the burden of porting your application to Intel® SGX is to rely on SCONE. At a high level SCONE provides a C standard library interface to container processes. System calls are executed outside of the enclave, but they are shielded by transparently encrypting/decrypting application data. Files stored outside of the enclave are therefore encrypted, and network communication is protected by transport layer security \(TLS\). With SCONE you can make your application compatible with Intel® SGX without modifying the source code. You would, still, re-build it though, but this is already a huge step forward. They provide a [curated list](https://sconedocs.github.io/SCONE_Curated_Images/) of base docker images that you can use according to your requirements.

## Terminology

#### MrEnclave: \(TLDR; It is the id of an enclave\)

The [MrEnclave](https://sconedocs.github.io/MrEnclave/) is a hash value that identifies every enclave. It is obtained from the content of memory pages and access rights. After you build your SCONE app, you will get a fingerprint, the MrEnclave is the last part of the fingerprint.

{% hint style="info" %}
Please note that some of SCONE's [environment variables](https://sconedocs.github.io/SCONE_ENV/) such as **SCONE\_HEAP** can affect the value of the MrEnclave.
{% endhint %}

#### FSPF \(File System Protection File\)

In addition to identifying the code, SCONE, also, takes a snapshot of the file system state. This guarantees that no one can alter the enclave by modifying its files' state. To do this, scone uses two parameters the [FSPF\_KEY](https://sconedocs.github.io/SCONE_Fileshield/#file-system-protection-file) and [FSPF\_TAG](https://sconedocs.github.io/SCONE_Fileshield/#file-system-protection-file).

#### Application's fingerprint

It is the concatenation of the MrEncalve, the FSPF\_KEY and the FSPF\_TAG seperated by a "\|". You should use this when deploying your application on the Blockchain. The SMS \(see below\) uses this as a reference to evaluate the state of client enclaves and whether they should get secrets or not.

## Secret Management Service \(SMS\)

With the integration of SCONE in iExec, you do not need to worry about [remote attestation](intel-sgx-technology.md#remote-attestation). All of that is done for you, it guarantees that the code is running inside an enclave. But that is not all, verification of whether the enclave asking for secrets is authorized or not is also handled. Hence, the need for a component to handle the permission management for those secrets. You guessed it, it is the SMS! The SMS queries the Blockchain an determines, for each task, the required secrets and provisions them on the fly.

Unquestionably, the SMS is a critical component. That's why it runs inside an [enclave](intel-sgx-technology.md#enclave).

## How it works?

{% hint style="info" %}
For more information about SCONE, please refer to their documentation at [https://sconedocs.github.io](https://sconedocs.github.io/).
{% endhint %}

We explain the process of how to make your Intel® SGX application using iExec in details in the [next chapter](create-your-first-sgx-app.md). Here is a quick general overview:

**Applications:** choose a base docker image for your use case. A template Dockerfile is provided so you would, just, add your specific requirements and dependencies, then build you image. Push your docker image somewhere accessible and deploy your application on the Blockchain with the correct image URI and fingerprint.

**Datasets:** To make your dataset available on iExec, you should, first, encrypt it with the SDK, and put the encrypted file publicly available. Deploy your dataset on the Blockchain, then, push the encryption key into the SMS where it is securely saved \(protected by Intel® SGX, which means no one can access it\). Only applications you authorize can get this key.

