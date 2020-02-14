# Protect result

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 8.0.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 4.0.2 or higher.
* [Quick start](../quick-start-for-developers.md) tutorial.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](intel-sgx-technology.md#scone-framework) framework.
* [Build trusted applications](create-your-first-sgx-app.md) tutorial.
{% endhint %}

In previous tutorials, we saw how to build trusted applications that run securely inside [enclaves](intel-sgx-technology.md#enclave) and combine them with confidential datasets to get the most out of confidential computing advantages. In this chapter, we will push thing further to protect the workflow in an end to end mode. That means the next step would be encrypting results.

Assuming your application is deployed \(also your dataset if you use one\), before triggering an execution you need to tell the [Secret Management Service](intel-sgx-technology.md#secret-management-service-sms) that you want your results to be encrypted. To do that you should generate a public/private AES key-pair and push the public part to the SMS. The latter will provide it, at runtime, to the enclave running your trusted application.

{% hint style="info" %}
you don't need to change your applications code to implement the encryption, it is already in there. You only need to make sure that the results are produced in the **/scone** folder.
{% endhint %}

To generate the key-pair, got to `~/iexec-projects` and use the following SDK command:

```text
iexec result generate-keys
```

This generates two files in `.secrets/beneficiary/`. Make sure to back up the private key in the file `<0x-your-wallet-address>_key`.

```text
.secrets
├── beneficiary
│   ├── <0x-you-wallet-address>_key
│   └── <0x-you-wallet-address>_key.pub
└── datasets
    ├── dataset.secret
    └── my-first-dataset.scone.secret
```

 Now, push the public key to the SMS:

```text
iexec result push-secret --chain goerli
```

And check it using:

```text
iexec result check-secret --chain goerli
```

Now to see that in action, you'd need to trigger a task and specify yourself as the beneficiary in the command:

```text
iexec app run <0x-your-app-address> \
    --chain goerli                  \
    --params "<your params>"        \
    --tag tee                       \
    --dataset <0x-your-dataset-address or 0x0> \
    --beneficiary <0x-your-wallet-address> \
    --watch
```

Wait for the task to be `COMPLETED` and download the result:

```text
iexec task show <0x-your-task-id> --download --chain goerli
```

If you try to inspect the content of the sub-directory `iexec_out`, you would see that it is encrypted:

























