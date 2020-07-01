# Protect result

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 10.12.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 5.0.0 or higher.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](intel-sgx-technology.md#scone-framework) framework.
{% endhint %}

{% hint style="warning" %}
Please make sure you have already checked the [Quickstart](../your-first-app.md), [Your first application](../your-first-app.md) and [Build trusted applications](create-your-first-sgx-app.md) tutorials before learning how to protect the result of your task.
{% endhint %}

In previous tutorials, we saw how to build trusted applications that run securely inside [enclaves](intel-sgx-technology.md#enclave) and combine them with confidential datasets to get the most out of confidential computing advantages. In this chapter, we will push things further to protect the workflow in an end to end mode. That means the next step would be encrypting results.

{% hint style="info" %}
You don't need to change your application's code or redeploy it to add this feature.
{% endhint %}

Assuming your application is deployed \(if not please check how to do it [here](../your-first-app.md#deploy-your-app-on-iexec)\), before triggering an execution you need to generate a public/private AES key-pair and push the public part to the [Secret Management Service](intel-sgx-technology.md#secret-management-service-sms). The latter, in turn, will provide it, at runtime, to the enclave running your trusted application.

To generate the key-pair, go to `~/iexec-projects` and use the following SDK command:

```bash
iexec result generate-encryption-keypair
```

This generates two files in `.secrets/beneficiary/`. Make sure to back up the private key in the file `<0x-your-wallet-address>_key`.

```bash
.secrets
├── beneficiary
│   ├── <0x-you-wallet-address>_key
│   └── <0x-you-wallet-address>_key.pub
...
```

Now, push the public key to the SMS:

```bash
iexec result push-encryption-key --chain goerli
```

And check it using:

```bash
iexec result check-encryption-key --chain goerli
```

Now to see that in action, you'd need to trigger a task and specify yourself as the beneficiary in the command:

```bash
iexec app run <0x-your-app-address> \
    --chain goerli                  \
    --tag tee                       \
    --encrypt-result \
    --beneficiary <0x-your-wallet-address> \
    --watch
```

Wait for the task to be `COMPLETED` and download the result:

```bash
iexec task show <0x-your-task-id> --download --chain goerli
```

If you extract the obtained zip and try to read the content of the file `iexec_out/result.zip.aes` you will find it encrypted:

```bash
mkdir /tmp/trash && \
    unzip <0x-your-task-id>.zip -d /tmp/trash && \
    cat /tmp/trash/iexec_out/result.zip.aes
```

{% code title="iexec\_out/result.zip" %}
```bash
)3�Xq��Yv��ȿzE�fRu<\�ݵm�m���疞r���c��(a���{{'��ܼ���͛�q/[{����H�t>��������h��gD$g��\.�k��j�����"�s?"�h�J�_Q41�_[{��X��������Ԛ��a�蘟v���E����r����肽
�����Յ]9W�TL�*���
          �t��d���z��O`����!���e�&snoL3�K6L9���%
```
{% endcode %}

Now you should decrypt the result by running:

```bash
iexec result decrypt <0x-your-task-id.zip>
```

A new zip file appears in the current folder under the name `results.zip`. Eventually, unzip it:

```bash
unzip results.zip -d my-decrypted-result
```

And you can see the content of your result file:

```bash
$ cat my-decrypted-result/my-result.txt
Hello, world!
```

Voilà! By finishing this part, you should be able to use confidential computing on iExec like a Ninja. All parts of the workflow are protected: the execution, the dataset, and the result.

You can go to the advanced section and learn more about managing orders on the iExec to effectively monetize your applications and datasets.

