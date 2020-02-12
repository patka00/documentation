# Manage confidential datasets

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 8.0.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 4.0.1 or higher.
* [Quick dev start](../quick-start-for-developers.md) tutorial.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](scone-framework.md#scone-framework) framework.
{% endhint %}

Trusted Execution Environments provide a huge advantage from a security perspective. They guarantee that the behavior of an execution does not change even when launched on an untrusted remote machine. The data inside this type of environments is also protected, which allows its monetization while preventing leakage.

With iExec, it is possible to authorize only applications you trust to use your datasets and get paid for it. Data is encrypted using standard encryption mechanisms and the plain version never leaves your machine. The encrypted dataset is made available for usage and the encryption key is pushed into the [SMS](scone-framework.md#secret-management-service-sms) which runs inside a secure [enclave](intel-sgx-technology.md#enclave). After you deploy the dataset on iExec it is you, and only you, who decides which application is allowed to get the secret to decrypt it.

{% hint style="warning" %}
Datasets are only decrypted inside authorized [enclaves](intel-sgx-technology.md#enclave) and never leave them. Same thing for secrets.
{% endhint %}

Let's see how to do all of that!

## Encrypt the dataset

Before starting you need to create a wallet if you don't already have one. Go ahead and run:

```text
$ iexec wallet create
```

{% hint style="success" %}
Your wallet is stored in the ethereum keystore, the location depends on your OS:

* On Linux: ~/.ethereum/keystore
* On Mac : ~/Library/Ethereum/keystore
* On Windows: ~/AppData/Roaming/Ethereum/keystore

Wallet file name follow the pattern `UTC--CREATION_DATE--ADDRESS`
{% endhint %}

{% hint style="warning" %}
If you create a new wallet don't forget to ask for some Goerli ETH on their faucet [https://goerli-faucet.slock.it/](https://goerli-faucet.slock.it/).
{% endhint %}

Create a new folder and init it using the iExec SDK:

```text
$ mkdir iexec/ && cd iexec/
$ iexec init --skip-wallet
$ tree
.
├── chain.json
└── iexec.json
```

You should see two new files in the directory `iexec.json` and `chain.json`. Now, init the dataset configuration. This command will create the folders `datasets/encrypted`, `datasets/original` and `.secrets/datasets`. A new section `"dataset"` will be added to the `iexec.json` file.

```text
$ iexec dataset init --encrypted
$ tree
.
├── chain.json
├── datasets
│   ├── encrypted
│   └── original
├── iexec.json
└── .secrets
    └── datasets
```

We will create a dummy dataset that has the line `"Hello, world!"` inside `datasets/original`. Alternatively, you can put the zip file of your dataset.

```text
$ echo "Hello, world!" > datasets/original/hello-world.txt
$ tree
.
├── chain.json
├── datasets
│   ├── encrypted
│   └── original
│       └── hello-world.txt
└── iexec.json
```

Now run the following command to encrypt the file:

```text
$ iexec dataset encrypt --algorithm scone
$ tree -a
.
├── chain.json
├── datasets
│   ├── encrypted
│   │   └── dataset_helloworld.txt.zip
│   └── original
│       └── hello-world.txt
├── iexec.json
└── .secrets
    └── datasets
        ├── dataset_helloworld.txt.scone.secret
        └── dataset.secret
```

As you can see the command generated the file `datasets/encrypted/dataset_helloworld.txt.zip`. That file is the encrypted version of your dataset so you can push it somewhere accessible for use and get its URI.

The file `.secrets/datasets/dataset_helloworld.txt.scone.secret` is the encryption key, make sure to back it up securely. The file `.secrets/datasets/dataset.secret` is just an "alias" in the sense that it has the key of the last encrypted dataset.

## Deploy the dataset

Fill in the fields of the `iexec.json` file.

```text
$ cat iexec.json
{
  "description": "My iExec ressource description...
  
  ...
  
  "dataset": {
    "owner": "0x-your-wallet-address",
    "name": "my-dataset",
    "multiaddr": "/ipfs/QmW2WQi7j6c7UgJTarActp7tDNikE4B2qXtFCfLPdsgaTQ",
    "checksum": "0x0000000000000000000000000000000000000000000000000000000000000000"
  }
}
```

Choose a name for your dataset \("Encrypted Hello world" for example\) and put the encrypted dataset's URI in `"multiaddr"`. Don't forget the checksum. To deploy the application run:

```text
$ iexec dataset deploy --chain goerli
```

You will get an hexadecimal address for you deployed application. Use that address to push the encryption key to the [SMS](scone-framework.md#secret-management-service-sms) so it is available for authorized applications.

```text
$ iexec dataset push-secret 0xaddress --chain goerli
```

Check it by doing:

```text
$ iexec dataset check-secret 0x5A713b5492EEa38928772d683Ec6183959a39058 --chain goerli
```

We saw in this section how to encrypt a dataset with [SCONE](scone-framework.md#scone-framework) and deploy it on iExec. We learned also how to push the encryption secret to the SMS. In the next chapter we will see how to combine both, the application and the dataset to create a complete workflow.





























