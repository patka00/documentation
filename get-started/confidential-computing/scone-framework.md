# Intel SGX with iExec

## SCONE Framework

For security reasons, the system calls inside SGX enclaves behave differently. It means that, in order to run you application inside an enclave, you would need to rewrite it using Intel's [SDK](https://software.intel.com/en-us/sgx/sdk). We agree with you, this is far from being convenient. And since, at iExec, developer experience is one of our top priorities, we always seek to integrate state of the art technologies that simplify your work. Thus, we partnered with [Scontain](https://scontain.com) to lighten this migration while maintaining transparency. With SCONE you can make your application compatible with SGX without modifying the source code. You would, still, re-build it though, but this is already a huge step forward. They provide a [curated list](https://sconedocs.github.io/SCONE_Curated_Images/) of base docker images that you can use according to your requirements.

## Secret Management Service \(SMS\)

With the integration of SCONE in iExec, you do not need to worry about [remote attestation](intel-sgx-technology.md#remote-attestation). We do that for you, we guarantee that the code is running inside an enclave. But that is not all, we also verify that the enclave asking for secrets is authorized to do so. Hence, we implemented a component to handle the permission management for those secrets. You guessed it, it is the SMS! The SMS queries the blockchain an determines, for each task, the required secrets and provisions them on the fly.

Unquestionably, the SMS is a critical component. That's why we run it inside and SGX enclave.

## Terminology

#### MrEnclave: \(TLDR; It is the id of an enclave\)

The [MrEnclave](https://sconedocs.github.io/MrEnclave/) is a hash value that identifies every enclave. It is obtained from the content of memory pages and access rights. After you build your SCONE app, you will get a fingerprint that is composed of 3 parts. the 1st and 2nd parts are explained in the section below. The 3rd part is the MrEnclave.

{% hint style="info" %}
Please note that some of SCONE's [environment variables](https://sconedocs.github.io/SCONE_ENV/) such as **SCONE\_HEAP** can affect the value of the MrEnclave.
{% endhint %}

#### FSPF \(File System Protection File\)

In addition to identifying the code, SCONE, also, takes a snapshot of the file system state. This guarantees that we cannot alter the enclave by modifying its files. The 1st and 2nd parts of the application's fingerprint are the [FSPF\_KEY](https://sconedocs.github.io/SCONE_Fileshield/#file-system-protection-file) and [FSPF\_TAG](https://sconedocs.github.io/SCONE_Fileshield/#file-system-protection-file).

## How it works?

{% hint style="info" %}
For more information about SCONE, please refer to their documentation at [https://sconedocs.github.io](https://sconedocs.github.io/).
{% endhint %}

We explain the process of how to make you application SGX enabled using iExec and SCONE in details in the [next chapter](create-your-first-sgx-app.md). But the general overview is simple:

**SGX Application:** First things first, you should start by choosing a base docker image for your use case. We provide a Dockerfile template so you would, just, need to add your specific requirements and dependencies. When your docker file is ready, build your image and save the [MrEnclave](scone-framework.md#mrenclave) to use it later when ~~**deploying your app &lt;add link&gt;**~~ on the blockchain.

**SGX Dataset:** To make your dataset usable with SGX applications on iExec, you would need to encrypt it with our SDK and push your encryption key into the SMS

**Request:** params protected and filesystem.



You use SCONE make your applications run inside an enclave. After building your app, SCONE measures the bina.

~~~~

~~Our SGX framework is based on the Scone runtime, that allows us to run unmodified apps inside SGX enclaves.~~

~~Hence your Docker image should be built from our python\_sgx image available on our docker repository.~~

~~~~



To avoid this and make the use of SGX through iExec as developer friendly as possible, iExec provides a transparent integration with Scone, a runtime component developed by Scontain that allows to run applications in SGX enclaves in an unmodified way. We provide several docker images, that already include the Scone components as well as iExec integration code, that make the development of iExec-ready, SGX-enabled dApp as simple as a few Dockerfile lines.



MrEnclave

Protected region

CAS

LAS

