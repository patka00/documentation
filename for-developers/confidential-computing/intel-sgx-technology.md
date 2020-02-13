# Intel® SGX technology

## Intel® Software Guard Extension \(Intel® SGX\)

[Intel® SGX](https://software.intel.com/en-us/sgx) is a technology that enables **Trusted** and **Confidential Computing**. At its core, it relies on the creation of a special zone in the memory, to which, only the CPU can have access. Neither privileged access level such as root, nor the operating system itself is capable of inspecting the content of this region. The code as well as the data inside the protected zone is totally unreadable and unalterable from the outside. Which guarantees both, non-disclosure of data and a tamper-proof execution of the code.

An application's code can be partitioned into "trusted" and "untrusted" parts where sensitive data is manipulated inside the protected area.

## Enclave

In confidential computing jargon, it is called "enclave" the special memory zone protected by the CPU. For simplicity sake, we can refer to private regions of memory defined by Intel® SGX application as "enclave".

## Remote attestation

As explained by [Intel](https://software.intel.com/en-us/sgx/attestation-services), the remote attestation is the process that happens before any exchange between a remote provider and an enclave. It allows the provider to verify that the expected software is running in an Intel® SGX-protected way while getting some details about the application being attested. If the attestation is successful, a secure communication channel is established between the two, and secrets can safely land in the enclave.

