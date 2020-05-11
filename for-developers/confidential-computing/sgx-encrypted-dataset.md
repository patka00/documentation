# Manage confidential datasets

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 8.0.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 4.0.2 or higher.
* [Quick start](../quick-start-for-developers.md) tutorial.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](intel-sgx-technology.md#scone-framework) framework.
* [Build trusted applications](create-your-first-sgx-app.md) tutorial.
{% endhint %}

Trusted Execution Environments offer a huge advantage from a security perspective. They guarantee that the behavior of an execution does not change even when launched on an untrusted remote machine. The data inside this type of environments is also protected, which allows its monetization while preventing leakage.

With iExec, it is possible to authorize only applications you trust to use your datasets and get paid for it. Data is encrypted using standard encryption mechanisms and the plain version never leaves your machine. The encrypted version is made available for usage and the encryption key is pushed into the [SMS](intel-sgx-technology.md#secret-management-service-sms) which runs inside a secure [enclave](intel-sgx-technology.md#enclave). After you deploy the dataset on iExec it is you, and only you, who decides which application is allowed to get the secret to decrypt it.

{% hint style="warning" %}
Datasets are only decrypted inside authorized [enclaves](intel-sgx-technology.md#enclave) and never leave them. Same thing for secrets.
{% endhint %}

Let's see how to do all of that!

## Encrypt the dataset

Before starting let's make sure we are inside the `~/iexec-projects` folder that we created previously, during the [quick start](../quick-start-for-developers.md) tutorial.

```bash
cd ~/iexec-projects
```

Init the dataset configuration.

```bash
iexec dataset init --encrypted
```

This command will create the folders `datasets/encrypted`, `datasets/original` and `.secrets/datasets`. A new section `"dataset"` will be added to the `iexec.json` file as well.

```bash
.
├── chain.json
│
├── datasets
│   ├── encrypted
│   └── original
│
├── deployed.json
├── iexec.json
└── scone-hello-world-app
│   └── ...
│
└── .secrets
    └── datasets
```

First create your dataset folder:

```bash
mkdir datasets/original/my-first-dataset
```

We will create a dummy file that has `"Hello, world!"` as a content inside `datasets/original/my-first-dataset`. Alternatively, you can put your own dataset file.

```bash
echo "Hello, world!" > datasets/original/my-first-dataset/hello-world.txt
```

```bash
datasets
├── encrypted
└── original
    └── my-first-dataset
        └── hello-world.txt
```

Now run the following command to encrypt the file:

```bash
iexec dataset encrypt --algorithm scone
```

```bash
datasets
├── encrypted
│   └── my-first-dataset.zip
└── original
    └── my-first-dataset
        └── hello-world.txt
```

As you can see, the command generated the file `datasets/encrypted/my-first-dataset.zip`. That file is the encrypted version of your dataset, you should push it somewhere accessible because the worker will download it during the execution process. You will enter this file's URI in the `iexec.json`file \(`multiaddr` attribute\) when you will deploy your dataset. Make sure that the URI is a **DIRECT** download link \(not a link of a web page for example\).

{% hint style="info" %}
You can use Github for example to publish the file but you should add **/raw/** to the URI like this: [https://github.com/&lt;username&gt;/&lt;repo&gt;/raw/master/my-first-dataset.zip](https://github.com/<username>/<repo>/raw/master/my-first-dataset.zip)
{% endhint %}

The file `.secrets/datasets/my-first-dataset.scone.secret` is the encryption key, make sure to back it up securely. The file `.secrets/datasets/dataset.secret` is just an "alias" in the sense that it has the key of the last encrypted dataset.

```bash
.secrets
└── datasets
    ├── dataset.secret
    └── my-first-dataset.scone.secret
```

## Deploy the dataset

Fill in the fields of the `iexec.json` file. Choose a `name` for your dataset, put the encrypted file's URI in `multiaddr`\(the URI you got after publishing the file\), and add the `checksum` \(you can get it by running `sha256sum datasets/encrypted/my-first-dataset.zip`\)

```bash
$ cat iexec.json
{
  "description": "My iExec ressource description...",

  ...

  "dataset": {
    "owner": "0x-your-wallet-address",
    "name": "Encrypted hello world dataset",
    "multiaddr": "/ipfs/QmW2WQi7j6c7UgJTarActp7tDNikE4B2qXtFCfLPdsgaTQ",
    "checksum": "0x0000000000000000000000000000000000000000000000000000000000000000"
  }
}
```

To deploy your dataset run:

```bash
iexec dataset deploy --chain goerli
```

You will get a hexadecimal address for you deployed dataset. Use that address to push the encryption key to the [SMS](intel-sgx-technology.md#secret-management-service-sms) so it is available for authorized applications.

```bash
iexec dataset push-secret <0x-your-dataset-address> --chain goerli
```

Check it by doing:

```bash
iexec dataset check-secret <0x-your-dataset-address> --chain goerli
```

We saw in this section how to encrypt a dataset with [SCONE](intel-sgx-technology.md#scone-framework) and deploy it on iExec. In addition, we learned how to push the encryption secret to the [SMS](intel-sgx-technology.md#secret-management-service-sms). Now we need to build the application that is going to consume this dataset. To do that you can use our [template from Github](https://github.com/iExecBlockchainComputing/scone-hello-world-app-with-dataset).

Make sure you are in `~/iexec-projects` and clone the repository:

```bash
cd ~/iexec-projects && \
  git clone https://github.com/iExecBlockchainComputing/scone-hello-world-app-with-dataset.git && \
  cd scone-hello-world-app-with-dataset/
```

This python application will read your dataset and write its content to a result file:

{% code title="app.py" %}
```python
with open("/iexec_in/hello-world.txt") as fin:
    with open("/scone/iexec_out/my-result.txt", "w+") as fout:
        data = fin.read()
        print(data)
        fout.write(data)
```
{% endcode %}

{% hint style="info" %}
Note that the result files should be written in the **/scone** folder.
{% endhint %}

Now follow the exact same steps that we saw when [building our first trusted application](create-your-first-sgx-app.md#prepare-the-application) to build and deploy this new app. Don't forget to change the name of the docker image \(to `scone-hello-world-app-with-dataset` for example\) and use your dataset address instead of `0x0` with the `--dataset` option when running `iexec app run`.

At the end, when you download the result, unlike the last tutorial, you won't see any relevant logs in `stdout.txt` but you will find the content of the plain dataset inside `iexec_out/result.zip/my-result.txt`.

## Next step?

Thanks to the explained confidential computing workflow, it is possible to use an encrypted dataset with a trusted application. We can go another step further and protect the result too. See in the next chapter how to make your execution result encrypted so that you are the only one who can read it.

