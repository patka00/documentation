# Create your "hello-world" Intel® SGX app

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 8.0.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 4.0.0 or higher.
* [Quick dev start](../../for-developers/quick-start-for-developers.md) tutorial.
{% endhint %}

To start this tutorial, make sure you are already familiar with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](scone-framework.md#scone-framework) framework. We will be using Python as the programming language, but support of other languages is coming soon. The tutorial is divided into tow major parts:

* [Prepare the application.](create-your-first-sgx-app.md#prepare-the-application)
* [Build its docker image.](create-your-first-sgx-app.md#build-the-application)
* [Deploy it on iExec](create-your-first-sgx-app.md#deploy-the-application-on-iexec)

Now, let's roll up our sleeves and start playing with [enclaves](intel-sgx-technology.md#enclave). 

## Prepare the application:

For simplicity sake, we provide a [Github repository](https://github.com/iExecBlockchainComputing/confidential-computing-tutorials.git) where you will find all the code and file templates used in this tutorial. You can, also, use it as a starter to create your own applications once you are ready. Start by cloning the Github repository and `cd` into `scone/hello-world-app` directory.

```
$ git clone https://github.com/iExecBlockchainComputing/confidential-computing-tutorials.git
$ cd confidential-computing-tutorials/scone/hello-world-app
```

If you `tree` the content of the directory you will find this structure:

```bash
$ tree
.
├── Dockerfile
├── src
│   └── app.py
└── utils
    ├── protect-fs.sh
    └── signer.py
```

Our application's source code is a python script that echos "hello world" to illustrate a simple run inside an enclave.

{% code title="src/app.py" %}
```bash
print("Hello from inside the enclave!")
print("It's dark over here!")
```
{% endcode %}

{% hint style="info" %}
The file **utils/signer.py** is just a temporary workaround and it will be removed in the next release. But, for now, it is mandatory for the execution to work properly.
{% endhint %}

The `Dockerfile` is a ready-to-go template where you just need to add your system packages and application's dependencies in the dedicated block \(do not forget to put the correct docker entrypoint\).

```bash
################################

### install apk dependencies
RUN apk add --no-cache bash build-base gcc libgcc

### uncomment and add your pip dependencies
# RUN SCONE_MODE=sim pip install web3

### or use a requirements.txt file
# COPY ./requirements.txt /requirements.txt
# RUN SCONE_MODE=sim pip install -r /requirements.txt

### copy the code inside the image
COPY ./src /app

################################
```

{% hint style="info" %}
That should be enough for this tutorial, but if you have other specifications you can manipulate the Dockerfile and adapt it to satisfy your requirements, just be sure to copy all your files in the image before invoking the **protect-fs.sh** script \(see below\).
{% endhint %}

The base docker image `iexechub/sconecuratedimages-iexec:python-3.7.3-alpine-3.10` contains a python interpreter that runs inside an enclave. When started, it will read the application's code and execute it. The question here is: **how would the enclave verify the integrity of the code?**

Well that's where the file `utils/protect-fs.sh` comes in place. If you inspect the content of this script, you can see that we use the famous [fspf](scone-framework.md#fspf-file-system-protection-file) feature of SCONE. We use SCONE's [CLI](https://sconedocs.github.io/SCONE_CLI/) to authenticate the file system directories that can be used by the application \(/bin, /lib...\) as well as the code itself, and take a snapshot of their state. This snapshot will be later shared with the enclave \(via the Blockchain\) to make sure everything is under control. If we change one bit of one of the authenticated files, the file system's state changes completely and the enclave will refuse to boot since it is a possible attack.

{% hint style="warning" %}
It is important to carefully choose files to authenticate. It can be tricky to consider including enough files to protect the application without being more general than we should. For example if we authenticate the entire /etc directory the enclave will fail to start because the content of /etc/hosts is modified at runtime by Docker.
{% endhint %}

{% hint style="warning" %}
That's why we do not simply authenticate "/" for example!
{% endhint %}

{% hint style="info" %}
Please note that the base docker image is an alpine 3.10 and the version of the python interpreter is 3.7.3, make sure those versions match your requirements.
{% endhint %}

## Build the application's docker image:

Once the `Dockerfile` is ready we proceed to building the image. Make sure you are inside the `confidential-computing-tutorials/scone/hello-world-app/` directory and run the following command:

```bash
$ docker image build -t scone-hello-world-app .
```

If every thing goes well you should see this output at the end of the build:

```bash
#####################################################################
Application's fingerprint (use this when deploying your app onchain):
5abc9e3a43e26870b9967ef31ea5572f90f8a12873425305f4fdff9e730e09c0|d72cfe7975922ccb70b7b859970e16b0|16e7c11e75448e31c94d023e40ece7429fb17481bc62f521c8f70da9c48110a1
#####################################################################
```

As mentioned in the output, that alphanumeric string is the [fingerprint](scone-framework.md#applications-fingerprint) of your application. It allows the verification of it's integrity.

Push the obtained docker container to dockerhub so it is publicly available.

```bash
$ docker image push <username>/scone-hello-world-app:1.0.0
```

## Deploy the application on iExec

We explained in details the steps to deploy an application on iExec earlier in the documentation. We will directly use the commands here assuming you are already familiar with them. If not please refer to the [Quick dev start](../../for-developers/quick-start-for-developers.md) to get a deeper understanding of those steps.

First things first, you need a wallet, so let's start by creating one:

```bash
$ iexec wallet create
```

Create an iExec project and initialise it:

```bash
$ mkdir scone-hello-world-app
$ cd scone-hello-world-app
$ iexec init --skip-wallet
```

For testing purpose, we will be using the blockchain Goerli Testnet. You need Goerli ETH to be able to send transactions to the network. Go to their [faucet](https://goerli-faucet.slock.it/) and paste your address to get some of it. To get your wallet address run this command:

```bash
$ iexec wallet show --chain goerli
```

Init a new iExec app and fill in the fields in `iexec.json` \(name, multiaddr,...\). Put the application's fingerprint in the `mrenclave` field then run:

```bash
$ iexec app deploy --chain goerli
```

**TODO: Add `iexec app run` to test the app.**

That's it, you deployed you first SCONE app on iExec and it is ready to be invoked by tasks. See how to publish orders &lt;here&gt;.

## Next step?

In this tutorial you learned how to leverage your application with the power of Trusted Execution Environments using iExec. But according to your use case, you may need to use some confidential data to get the full potential of the **Confidential Computing** paradigm. It is possible to do that using iExec, check out next chapter to see how.



