# Build trusted applications

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 8.0.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 4.0.1 or higher.
* [Quick dev start](../quick-start-for-developers.md) tutorial.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](scone-framework.md#scone-framework) framework.
{% endhint %}

After understanding the fundamentals of Confidential Computing and explaining technologies behind it, it is time to roll up our sleeves and start playing with [enclaves](intel-sgx-technology.md#enclave).

In this tutorial, we will be using Python as the programming language, but support of other languages is coming soon. Three major sections are presented:

* [Prepare and "sconify" an application.](create-your-first-sgx-app.md#prepare-the-application)
* [Build your application.](create-your-first-sgx-app.md#build-the-application)
* [Deploy & test on iExec.](create-your-first-sgx-app.md#deploy-the-application-on-iexec)
* [Download the result.](create-your-first-sgx-app.md#download-the-result)

For simplicity sake, a [Github repository](https://github.com/iExecBlockchainComputing/confidential-computing-tutorials.git) is provided. You will find all the code and file templates used in this tutorial. You can, also, use it as a starter to create your own applications. Start by cloning the Github repository and `cd` into `scone/hello-world-app` directory.

```
$ git clone https://github.com/iExecBlockchainComputing/confidential-computing-tutorials.git
$ cd confidential-computing-tutorials/scone/hello-world-app
```

Our application's source code is a python script that echos "hello world" to illustrate a simple run inside an enclave.

{% code title="src/app.py" %}
```bash
# print to stdout
print("Hello from inside the enclave!")

# produce a result file in /scone
with open("/scone/my-result.txt", "w+") as result_file:
    result_file.write("It's dark over here!")
```
{% endcode %}

{% hint style="info" %}
Note that the result files should be written in the **/scone** folder.
{% endhint %}

## Prepare the application:

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

{% hint style="warning" %}
The file **utils/signer.py** is just a temporary workaround and it will be removed in the next release. But, for now, it is mandatory that's why we copy it inside the Dockerfile. It is called during the execution time by the worker.
{% endhint %}

The `Dockerfile` is a ready-to-go template where you just need to add your system packages and application's dependencies in the dedicated block \(do not forget to put the correct docker entrypoint\).

```bash
################################

### install apk packages
RUN apk add --no-cache bash build-base gcc libgcc

### install python3 dependencies
RUN SCONE_MODE=sim pip3 install attrdict python-gnupg web3

### copy your code inside the image
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

Once the `Dockerfile` is ready we proceed to building the image. Make sure you are inside the right directory and run the following command:

```bash
$ docker image build -t <username>/scone-hello-world-app:0.0.1 .
```

If every thing goes well you should see this output at the end of the build \(with different hashes of course\):

```bash
#####################################################################
Application's fingerprint (use this when deploying your app onchain):
5abc9e3a43e26870b9967ef31ea5572f90f8a12873425305f4fdff9e730e09c0|d72cfe7975922ccb70b7b859970e16b0|16e7c11e75448e31c94d023e40ece7429fb17481bc62f521c8f70da9c48110a1
#####################################################################
```

As mentioned in the output, that alphanumeric string is the [fingerprint](scone-framework.md#applications-fingerprint) of your application. It allows the verification of it's integrity.

Push the obtained docker container to docker registry so it is publicly available and get its checksum:

```bash
$ docker image push <username>/scone-hello-world-app:0.0.1
...
0.0.1: digest: sha256:bdc482735010af7bf400... size: 2621

```

## Deploy & test on iExec

We explained in details the steps to deploy an application on iExec earlier in the documentation. We will directly use the commands here assuming you are already familiar with them. If not please refer to the [Quick dev start](../quick-start-for-developers.md) to get a deeper understanding of those steps.

First things first, you need a wallet, so let's start by creating one \(skip this step if you already have one\):

```bash
$ iexec wallet create
```

{% hint style="success" %}
Your wallet is stored in the ethereum keystore, the location depends on your OS:

* On Linux: ~/.ethereum/keystore
* On Mac : ~/Library/Ethereum/keystore
* On Windows: ~/AppData/Roaming/Ethereum/keystore

Wallet file name follow the pattern `UTC--CREATION_DATE--ADDRESS`
{% endhint %}

Create an iExec project and initialise it:

```bash
$ mkdir iexec/ && cd iexec/
$ iexec init --skip-wallet
```

For testing purpose, we will be using the blockchain Goerli Testnet. You need Goerli ETH to be able to send transactions to the network. Go to their [faucet](https://goerli-faucet.slock.it/) and paste your address to get some of it. To get your wallet address run this command:

```bash
$ iexec wallet show --chain goerli
```

Init a new iExec app:

```bash
$ iexec app init
```

A new section `"app"` appears in iexec.json, fill in its fields and put the application's fingerprint in the `mrenclave` attribute, then deploy your app:

```bash
$ cat iexec.json
{
  "description": "My iExec ressource description...",
  ...
  ...
  "app": {
    "owner": "<0x-your-wallet-address>",
    "name": "Scone hello world",
    "type": "DOCKER",
    "multiaddr": "registry.hub.docker.com/<username>/scone-hello-world-app:0.0.1",
    "checksum": "<0xf51494d7a...>",
    "mrenclave": "5abc9e3a43e26870b9967ef31ea5572f90f8a12873425305f4fdff9e730e09c0|d72cfe7975922ccb70b7b859970e16b0|16e7c11e75448e31c94d023e40ece7429fb17481bc62f521c8f70da9c48110a1"
  }
}

$ iexec app deploy --chain goerli
ℹ using chain [goerli]
? Using wallet UTC...
Please enter your password to unlock your wallet [hidden]
✔ Deployed new app at address <0x-your-app-address>
```

To test your application on iExec use the command below. The `--watch` option will follow and display the status of the task in real time.

```bash
$ iexec app run <0x-your-app-address> \
    --chain goerli                    \
    --params "python3 /app/app.py"    \
    --tag tee                         \
    --dataset 0x0000000000000000000000000000000000000000 \
    --beneficiary 0x0000000000000000000000000000000000000000 \
    --watch
    
ℹ using chain [goerli]
? Using wallet UTC...
Please enter your password to unlock your wallet [hidden]
? Do you want to spend 0 nRLC to execute the following request: 
app:         <0x-your-app-address>                      (0 nRLC)
workerpool:  0x706bB37Fb0545aD82f02721cDe7B8F9d351390Ec (0 nRLC)
params:      python3 /app/app.py
category:    3
tag:         0x0000000000000000000000000000000000000000000000000000000000000001
beneficiary: 0x0000000000000000000000000000000000000000
 Yes
ℹ deal submitted with dealid 0x29f85a881f72f7040ff3fe8b9218ee2b8cc3541167bc8815c027a8c48a128b27
ℹ Watching tasks execution...
- Task idx 0 (0x3d77255d4c1061aaa12fb0be79d4bc5cb613fc66ce143162ef8a4ee2383cdf1b) status COMPLETED

✔ 1 tasks COMPLETED with dealid 0x29f85a881f72f7040ff3fe8b9218ee2b8cc3541167bc8815c027a8c48a128b27
```

You can get all information about your tasks in the iExec [explorer](https://explorer.iex.ec/goerli). If the execution fails or takes so long, you can check the debug enclave logs at [https://graylog.iex.ec/goerli-tee](https://graylog.iex.ec/goerli-tee). Use the search box to filter logs by **task id.**

{% hint style="info" %}
Please request access here: [https://cutt.ly/grGXZNY](https://cutt.ly/grGXZNY).
{% endhint %}

## Download the result

After the execution is finished \(the status is `COMPLETED`\) download the result:

```bash
$ iexec task show 0x3d77255d4c1061aaa12fb0be79d4bc5cb613fc66ce143162ef8a4ee2383cdf1b --download --chain goerli

ℹ using chain [goerli]
? Using wallet UTC...
Please enter your password to unlock your wallet [hidden]
✔ Task 0x3d77255d4c1061aaa12fb0be79d4bc5cb613fc66ce143162ef8a4ee2383cdf1b details: 
status:               3
dealid:               0x29f85a881f72f7040ff3fe8b9218ee2b8cc3541167bc8815c027a8c48a128b27
idx:                  0
timeref:              10800
contributionDeadline: 1581577752
revealDeadline:       1581523872
finalDeadline:        1581610152
consensusValue:       0x79aa3a2408de8f0a3c53720eaf45d723a25ec94eca600bb6acff058f930037ba
revealCounter:        1
winnerCounter:        1
contributors: 
  0: 0x1cb25226FeCeE496f246DDd1D735276B2E168B5a
resultDigest:         0x96b341807bafa4ce791572b4cb3ee42ed0960a5d571af6ef704bf8df861ab794
results:              /ipfs/QmWqHs8Q4dwPJvQZnz1CZYscMZ3NdPs5oAjt6LWun7ZfqX
statusName:           COMPLETED

ℹ downloaded task result to file /tmp/scone/confidential-computing-tutorials/scone/hello-world-app/iexec/0x3d77255d4c1061aaa12fb0be79d4bc5cb613fc66ce143162ef8a4ee2383cdf1b.zip
```

Unzip the task folder, then unzip the result folder:

```bash
$ unzip 0x3d77255d4c1061aaa12fb0be79d4bc5cb613fc66ce143162ef8a4ee2383cdf1b.zip -d result
$ unzip result/iexec_out/result.zip -d result/iexec_out
$ rm result/iexec_out/result.zip
  
$ tree result/
result/
├── iexec_out
│   ├── enclaveSig.iexec
│   ├── my-result.txt
│   ├── public.key
│   └── volume.fspf
└── stdout.txt
```

As you can see, the `result/` directory contains an `stdout.txt` file and the well known `iexec_out/` folder. In `iexec_out` resides our result file `my-result.txt` . You can verify their contents:

```bash
$ grep -n "Hello from inside the enclave!" result/stdout.txt 
39:Hello from inside the enclave!

$ cat result/iexec_out/my-result.txt 
It's dark over here!
```

{% hint style="info" %}
The folder "iexec\_out" contains other metadata files: **enclaveSig.iexec** - signature of the enclave used for verification, **public.key** \(of the requester\) and **volume.fspf** files are both used in the case of an encrypted result \(see this [chapter](end-to-end-encryption.md)\).
{% endhint %}

That's it, you deployed you first confidential computing app on iExec and ran your first enclave-protected execution successfully.

## Next step?

In this tutorial you learned how to leverage your application with the power of Trusted Execution Environments using iExec. But according to your use case, you may need to use some confidential data to get the full potential of the **Confidential Computing** paradigm. Check out next chapter to see how.

