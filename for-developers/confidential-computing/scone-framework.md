# Confidential Computing with iExec

## SCONE Framework

For security reasons, the system calls inside SGX enclaves behave differently. It means that, in order to run you application inside an enclave, you would need to rewrite it using Intel's [SDK](https://software.intel.com/en-us/sgx/sdk). We agree with you, this is far from being convenient. And since, at iExec, developer experience is one of our top priorities, we always seek to integrate state of the art technologies that simplify your work. Thus, we partnered with [Scontain](https://scontain.com) to lighten this migration while maintaining transparency. With SCONE you can make your application compatible with SGX without modifying the source code. You would, still, re-build it though, but this is already a huge step forward. They provide a [curated list](https://sconedocs.github.io/SCONE_Curated_Images/) of base docker images that you can use according to your requirements.

## Terminology

### MrEnclave: \(TLDR; It is the id of an enclave\)

The [MrEnclave](https://sconedocs.github.io/MrEnclave/) is a hash value that identifies every enclave. It is obtained from the content of memory pages and access rights. After you build your SCONE app, you will get a fingerprint, the MrEnclave is the last part of the fingerprint.

{% hint style="info" %}
Please note that some of SCONE's [environment variables](https://sconedocs.github.io/SCONE_ENV/) such as **SCONE\_HEAP** can affect the value of the MrEnclave.
{% endhint %}

### FSPF \(File System Protection File\)

In addition to identifying the code, SCONE, also, takes a snapshot of the file system state. This guarantees that we cannot alter the enclave by modifying its files' state. To do this, scone uses two parameters the [FSPF\_KEY](https://sconedocs.github.io/SCONE_Fileshield/#file-system-protection-file) and [FSPF\_TAG](https://sconedocs.github.io/SCONE_Fileshield/#file-system-protection-file).

### Application's fingerprint

It is the concatenation of the MrEncalve, the FSPF\_KEY and the FSPF\_TAG seperated by a "\|". You should use this when deploying your application on the blockchain. The SMS uses this as a reference to evaluate the state of client enclaves and whether they should get secrets or not.

## Secret Management Service \(SMS\)

With the integration of SCONE in iExec, you do not need to worry about [remote attestation](https://github.com/iExecBlockchainComputing/documentation/tree/1279416f007d16fd94a317e82c2d740a14ef4d2e/get-started/confidential-computing/intel-sgx-technology.md#remote-attestation). We do that for you, we guarantee that the code is running inside an enclave. But that is not all, we also verify that the enclave asking for secrets is authorized to do so. Hence, we implemented a component to handle the permission management for those secrets. You guessed it, it is the SMS! The SMS queries the blockchain an determines, for each task, the required secrets and provisions them on the fly.

Unquestionably, the SMS is a critical component. That's why we run it inside and SGX enclave.

## How it works?

{% hint style="info" %}
For more information about SCONE, please refer to their documentation at [https://sconedocs.github.io](https://sconedocs.github.io/).
{% endhint %}

We explain the process of how to make your SGX application using iExec in details in the [next chapter](https://github.com/iExecBlockchainComputing/documentation/tree/1279416f007d16fd94a317e82c2d740a14ef4d2e/get-started/confidential-computing/create-your-first-sgx-app.md). Here is a quick general overview:

**SGX Application:** First things first, choose a base docker image for your use case. We provide a template Dockerfile so you would, just, add your specific requirements and dependencies, then build you image. Push you docker image somewhere accessible and deploy your application on the blockchain with the correct image URI and fingerprint.

**SGX Dataset:** To make your dataset available on iExec, you should, first, encrypt it with the SDK, and put the encrypted file publicly available. Deploy your dataset on the blockchain, then, push the encryption key into the SMS where it is securely saved \(protected by SGX, which means even us we cannot access it\). Only applications you authorize can get this key.

