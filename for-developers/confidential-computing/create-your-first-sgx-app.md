# Build trusted applications

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 8.0.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 4.0.2 or higher.
* [Quickstart](../quick-start-for-developers.md) tutorial.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](intel-sgx-technology.md#scone-framework) framework.
{% endhint %}

{% hint style="warning" %}
Please make sure you have already checked the [quickstart](../your-first-app.md) tutorial before doing the following.
{% endhint %}

After understanding the fundamentals of Confidential Computing and explaining the technologies behind it, it is time to roll up our sleeves and get hands-on with [enclaves](intel-sgx-technology.md#enclave). In this guide, we will focus on protecting an application - that is already compatible with the iExec platform - using SGX, and without changing the source code.

Three main sections are presented:

* [Prepare and "sconify" an application.](create-your-first-sgx-app.md#prepare-the-application)
* [Build your application.](create-your-first-sgx-app.md#build-the-application)
* [Deploy & test on iExec.](create-your-first-sgx-app.md#deploy-the-application-on-iexec)
* [Download the result.](create-your-first-sgx-app.md#download-the-result)

For simplicity sake, a [Github repository](https://github.com/iExecBlockchainComputing/scone-hello-world-app) is provided. You will find all the code and file templates used in this tutorial. You can, also, use it as a starter to create your own applications. Let's open up a terminal and jump inside the `~/iexec-projects` folder that we already created earlier in the [quick start](../quick-start-for-developers.md) tutorial. Start by cloning the repository and `cd` into it:

```text
cd ~/iexec-projects
git clone https://github.com/iExecBlockchainComputing/scone-hello-world-app.git
cd scone-hello-world-app
```

Our application's source code is a python script that echos "hello world" to illustrate a simple run inside an enclave.

{% code title="app.py" %}
```python
with open("/scone/iexec_out/my-result.txt", "w+") as fout:
    message = "Hello from inside the enclave, it's dark over here!"

    # print to stdout
    print(message)

    # write result file in /scone/iexec_out
    fout.write(message)
```
{% endcode %}

{% hint style="info" %}
Note that the result files should be written in the **/scone** folder.
{% endhint %}

## Prepare the application:

If you `tree` the content of the directory you will find this structure:

```bash
.
├── app.py
├── Dockerfile
└── protect-fs.sh
```

The `Dockerfile` is a ready-to-go template where you just need to add your system packages and application's dependencies in the dedicated block.

{% code title="Dockerfile" %}
```bash
...
################################

### install apk packages
RUN apk add --no-cache bash build-base gcc libgcc

### install python3 dependencies
RUN SCONE_MODE=sim pip3 install attrdict python-gnupg web3

### copy your code inside the image
COPY app.py /app.py

################################
...
```
{% endcode %}

{% hint style="info" %}
That should be enough for this tutorial, but if you have other specifications you can manipulate the Dockerfile and adapt it to satisfy your requirements, just be sure to copy all your files in the image before invoking the **protect-fs.sh** script \(see below\).
{% endhint %}

The base docker image `iexechub/sconecuratedimages-iexec:python-3.7.3-alpine-3.10` contains a python interpreter that runs inside an enclave. When started, it will read the application's code and execute it. The question here is: **how would the enclave verify the integrity of the code?**

Well that's where the file `protect-fs.sh` comes in place. If you inspect the content of this script, you can see that we use the famous [fspf](intel-sgx-technology.md#fspf-file-system-protection-file) feature of SCONE. We use SCONE's [CLI](https://sconedocs.github.io/SCONE_CLI/) to authenticate the file system directories that can be used by the application \(/bin, /lib...\) as well as the code itself, and take a snapshot of their state. This snapshot will be later shared with the enclave \(via the Blockchain\) to make sure everything is under control. If we change one bit of one of the authenticated files, the file system's state changes completely and the enclave will refuse to boot since it is a possible attack.

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

Once the `Dockerfile` is ready we proceed to building the image. Make sure you are inside the right directory and run the following command in the terminal \(replace all occurrences of `<username>` with your Dockerhub username\):

```bash
docker image build -t <username>/scone-hello-world-app:0.0.1 .
```

If every thing goes well you should see this output at the end of the build \(with different hashes of course\):

```bash
#####################################################################
Application's fingerprint (use this when deploying your app onchain):
5abc9e3a43e26870b9967ef31ea5572f90f8a12873425305f4fdff9e730e09c0|d72cfe7975922ccb70b7b859970e16b0|16e7c11e75448e31c94d023e40ece7429fb17481bc62f521c8f70da9c48110a1
#####################################################################
```

As mentioned in the output, that alphanumeric string is the [fingerprint](intel-sgx-technology.md#applications-fingerprint) of your application. It allows the verification of it's integrity.

Push the obtained docker container to docker registry so it is publicly available and get its checksum:

```bash
docker image push <username>/scone-hello-world-app:0.0.1
```

## Deploy & test on iExec

Earlier in the documentation, we explained the steps to deploy an application on iExec. Now, we will use these previously detailed commands, assuming you are already familiar with them. If not, please refer to the [quick start](../quick-start-for-developers.md) to get a deeper understanding of those steps.

First things first, go back to the `~/iexec-projects` folder where we initialized our environment:

```bash
cd ~/iexec-projects # or a - super special - trick: run "cd .."!
```

Init a new iExec app:

```bash
iexec app init
```

A new section `"app"` appears in `iexec.json`. Fill in the fields: `name`, `mutiaddr` \(the URI of the docker image\), `checksum`, and put the application's fingerprint in the `mrenclave` attribute.

```bash
$ cat iexec.json
{
  "description": "My iExec ressource description...",
  ...
  ...
  "app": {
    "owner": "<0x-your-wallet-address>",
    "name": "Scone hello world app",
    "type": "DOCKER",
    "multiaddr": "registry.hub.docker.com/<username>/scone-hello-world-app:0.0.1",
    "checksum": "<0xf51494d7a...>",
    "mrenclave": "5abc9e3a43e26870b9967ef31ea5572f90f8a12873425305f4fdff9e730e09c0|d72cfe7975922ccb70b7b859970e16b0|16e7c11e75448e31c94d023e40ece7429fb17481bc62f521c8f70da9c48110a1"
  }
}
```

Now, deploy your app:

```bash
iexec app deploy --chain goerli
```

You should see this output:

```bash
ℹ using chain [goerli]
? Using wallet UTC<...>
Please enter your password to unlock your wallet [hidden]
✔ Deployed new app at address <0x-your-app-address>
```

To test your application on iExec use the command below. The `--watch` option will follow and display the status of the task in real time.

```bash
iexec app run <0x-your-app-address>   \
    --chain goerli                    \
    --params "python3 /app/app.py"            \
    --tag tee                         \
    --dataset 0x0000000000000000000000000000000000000000 \
    --beneficiary 0x0000000000000000000000000000000000000000 \
    --watch
```

If everything goes well you should see this:

```bash
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

You can get all information about your tasks in the iExec [explorer](https://explorer.iex.ec/goerli). If the execution fails or takes longer than it should, you can check the debug enclave logs at [https://graylog.iex.ec/goerli-tee](https://graylog.iex.ec/goerli-tee). Use the search box to filter logs by **task id.**

{% hint style="info" %}
Please request access here: [https://cutt.ly/grGXZNY](https://cutt.ly/grGXZNY).
{% endhint %}

## Download the result

After the execution is finished \(the status is `COMPLETED`\) download the result of your task. This command also provides a summary of the execution:

```bash
iexec task show 0x3d77255d4c1061aaa12fb0be79... --download --chain goerli
```

{% hint style="info" %}
Note that you should use the **task id** not the deal id.
{% endhint %}

```bash
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

ℹ downloaded task result to file /home/<user>/scone-hello-world-app/iexec/0x3d77255d4c1061aaa12fb0be79d4bc5cb613fc66ce143162ef8a4ee2383cdf1b.zip
```

Unzip the task folder:

```bash
unzip 0x3d77255d4c1061aaa12fb0be79...f1b.zip -d result
```

Then unzip the result folder:

```bash
unzip result/iexec_out/result.zip -d result/iexec_out && rm result/iexec_out/result.zip
```

The folder `result/` contains the following structure:

```bash
result/
├── iexec_out
│   ├── enclaveSig.iexec
│   ├── my-result.txt
│   ├── public.key
│   └── volume.fspf
└── stdout.txt
```

As you can see, we have an `stdout.txt` file that contains you application's logs and the well known `iexec_out/` folder. In `iexec_out` we find our result file `my-result.txt` . You can verify the content of these files:

```bash
$ grep -n "Hello from inside the enclave!" result/stdout.txt
39:Hello from inside the enclave!

$ cat result/iexec_out/my-result.txt
It's dark over here!
```

Don't worry if sometimes you see this error message in the `stdout.txt` file, it shouldn't affect the execution or the result.

```bash
Traceback (most recent call last):
  File "/signer/signer.py", line 40, in GetPublicKey
    pubKeyObj = RSA.importKey(key.read())
  File "/usr/lib/python3.7/site-packages/Crypto/PublicKey/RSA.py", line 785, in import_key
    raise ValueError("RSA key format is not supported")
ValueError: RSA key format is not supported
```

{% hint style="info" %}
The folder "iexec\_out" contains other metadata files: **enclaveSig.iexec** - signature of the enclave used for verification, **public.key** \(of the requester\) and **volume.fspf** files are both used in the case of an encrypted result \(see this [chapter](end-to-end-encryption.md)\).
{% endhint %}

That's it, you have successfully deployed your confidential computing app on iExec and ran your enclave-protected execution successfully.

## Next step?

In this tutorial you learned how to leverage your application with the power of Trusted Execution Environments using iExec. But according to your use case, you may need to use some confidential data to get the full potential of the **Confidential Computing** paradigm. Check out next chapter to see how.

