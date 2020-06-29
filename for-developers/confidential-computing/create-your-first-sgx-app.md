# Build trusted applications

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 8.0.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 4.0.2 or higher.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](intel-sgx-technology.md#scone-framework) framework.
{% endhint %}

{% hint style="warning" %}
Please make sure you have already checked the [quickstart](../your-first-app.md) tutorial before doing the following steps.
{% endhint %}

After understanding the fundamentals of Confidential Computing and explaining the technologies behind it, it is time to roll up our sleeves and get hands-on with [enclaves](intel-sgx-technology.md#enclave). In this guide, we will focus on protecting an application - that is already compatible with the iExec platform - using SGX, and without changing the source code. That means we will use the same code we [previously](../your-first-app.md#build-your-app) deployed for a basic iExec application.

Create a directory tree for your application in `~/iexec-projects/`.

```bash
cd ~/iexec-projects
mkdir my-tee-hello-world-app && cd my-tee-hello-world-app
mkdir src
touch Dockerfile
```

In the folder `src/` create the file `app.js` \(or `app.py` if you want to use Python\) then copy [this](../your-first-app.md#write-the-app-shell-script-example) code inside.

As we mentioned earlier, the advantage of using **SCONE** is the ability to make the application **Intel SGX-enabled** without changing the source code. The only thing we are going to modify is the `Dockerfile`. First, we need to change the base image from the official `node (or python)` to the one provided by SCONE `sconecuratedimages/apps:node-8.9.4-alpine-scone3.0`or `sconecuratedimages/apps:python-3.7.3-alpine3.10-scone3.0`. Those base docker images contain a `nodejs/python` interpreter that runs inside an enclave.

{% tabs %}
{% tab title="Javascript" %}
{% code title="Dockerfile" %}
```bash
FROM sconecuratedimages/sconecli:alpine3.7-scone3.0 AS scone

FROM sconecuratedimages/apps:node-8.9.4-alpine-scone3.0

COPY --from=scone   /opt/scone/scone-cli    /opt/scone/scone-cli
COPY --from=scone   /usr/local/bin/scone    /usr/local/bin/scone
COPY --from=scone   /opt/scone/bin          /opt/scone/bin

### install dependencies you need
RUN apk add bash nodejs-npm
RUN SCONE_MODE=sim npm install figlet

COPY ./src /app

###  protect file system with Scone
COPY ./tee/protect-fs.sh ./tee/Dockerfile /build/
RUN sh /build/protect-fs.sh /app

ENTRYPOINT [ "node", "/app/app.js"]
```
{% endcode %}
{% endtab %}

{% tab title="Python" %}
```
FROM sconecuratedimages/apps:python-3.7.3-alpine3.10-scone3.0

### install python3 dependencies you need
RUN SCONE_MODE=sim pip3 install pyfiglet

COPY ./src /app

###  protect file system with Scone
COPY ./tee/protect-fs.sh ./tee/Dockerfile /build/
RUN sh /build/protect-fs.sh /app

ENTRYPOINT ["python", "/app/app.py"]
```
{% endtab %}
{% endtabs %}

In the above [multi-stage build](https://docs.docker.com/develop/develop-images/multistage-build/), we copied some needed SCONE CLI binaries from the image `sconecuratedimages/sconecli:alpine3.7-scone3.0` then we installed the dependencies needed by the application \(`L1-11`\). The section `L16-17` is the answer to the legitimate question: **how would the enclave verify the integrity of the code?**

The short answer is: the application is protected by taking a snapshot of the file system's state. The script `protect-fs.sh` uses the famous [fspf](intel-sgx-technology.md#fspf-file-system-protection-file) feature of SCONE to authenticate the file system directories that would be used by the application \(/bin, /lib...\) as well as the code itself. It takes a snapshot of their state that will be later shared with the worker \(via the Blockchain\) to make sure everything is under control. If we change one bit of one of the authenticated files, the file system's state changes completely and the enclave will refuse to boot since it considers it as a possible attack.

{% code title="protect-fs.sh" %}
```bash
#!/bin/sh

cd $(dirname $0)

if [ ! -e Dockerfile ]
then
    printf "\nFailed to parse Dockerfile ENTRYPOINT\n"
    printf "Did you forget to add your Dockerfile in your build?\n"
    printf "COPY ./tee/Dockerfile /build/\n\n"
    exit 1
fi

ENTRYPOINT_ARSG=$(grep ENTRYPOINT ./Dockerfile | tail -1 |  grep -o '"[^"]\+"' | tr -d '"')
echo $ENTRYPOINT_ARSG > ./entrypoint

if [ -z "$ENTRYPOINT_ARSG" ]
then
    printf "\nFailed to parse Dockerfile ENTRYPOINT\n"
    printf "Did you forget to add an ENTRYPOINT to your Dockerfile?\n"
    printf "ENTRYPOINT [\"executable\", \"param1\", \"param2\"]\n\n"
    exit 1
fi

INTERPRETER=$(awk '{print $1}' ./entrypoint) # python
ENTRYPOINT=$(cat ./entrypoint) # /python /app/app.py

export SCONE_MODE=sim
export SCONE_HEAP=1G

APP_FOLDER=$1

printf "\n### Starting file system protection ...\n\n"

scone fspf create /fspf.pb
scone fspf addr /fspf.pb /          --not-protected --kernel /
scone fspf addr /fspf.pb /usr       --authenticated --kernel /usr
scone fspf addf /fspf.pb /usr       /usr
scone fspf addr /fspf.pb /bin       --authenticated --kernel /bin
scone fspf addf /fspf.pb /bin       /bin
scone fspf addr /fspf.pb /lib       --authenticated --kernel /lib
scone fspf addf /fspf.pb /lib       /lib
scone fspf addr /fspf.pb /etc/ssl   --authenticated --kernel /etc/ssl
scone fspf addf /fspf.pb /etc/ssl   /etc/ssl
scone fspf addr /fspf.pb /sbin      --authenticated --kernel /sbin
scone fspf addf /fspf.pb /sbin      /sbin
printf "\n### Protecting code found in folder \"$APP_FOLDER\"\n\n"
scone fspf addr /fspf.pb $APP_FOLDER --authenticated --kernel $APP_FOLDER
scone fspf addf /fspf.pb $APP_FOLDER $APP_FOLDER

scone fspf encrypt /fspf.pb > ./keytag

MRENCLAVE="$(SCONE_HASH=1 $INTERPRETER)"
FSPF_TAG=$(cat ./keytag | awk '{print $9}')
FSPF_KEY=$(cat ./keytag | awk '{print $11}')
FINGERPRINT="$FSPF_KEY|$FSPF_TAG|$MRENCLAVE|$ENTRYPOINT"
echo $FINGERPRINT > ./fingerprint

printf "\n\n"
printf "Your application fingerprint (mrenclave) is ready:\n"
printf "#####################################################################\n"
printf "iexec.json:\n\n"
printf "%s\n" "\"app\": { " " \"owner\" : ... " " \"name\": ... " "  ..." " \"mrenclave\": \"$FINGERPRINT\"" "}"
printf "#####################################################################\n"
printf "Hint: Replace 'mrenclave' before doing 'iexec app deploy' step.\n"
printf "\n\n"

```
{% endcode %}

{% hint style="info" %}
All dependencies and files must be added to the image before invoking the **protect-fs.sh** script \(see below\).
{% endhint %}

{% hint style="warning" %}
It is important to carefully choose files to authenticate. It can be tricky to consider including enough files to protect the application without being more general than we should. For example if we authenticate the entire /etc directory the enclave will fail to start because the content of /etc/hosts is modified at runtime by Docker.
{% endhint %}

{% hint style="warning" %}
That's why we do not simply authenticate "/" for example!
{% endhint %}









In the end, you should have this structure:

```bash
.
├── app.py
sdjflskdjflsjdf
├── Dockerfile
└── protect-fs.sh
```

## Build the application's docker image:

Once the `Dockerfile` is ready we proceed to building the image. Make sure you are inside the right directory and run the following command in the terminal \(replace all occurrences of `<dockerusername>` with your Dockerhub dockerusername\):

```bash
docker image build -t <dockerusername>/scone-hello-world-app:0.0.1 .
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
docker image push <dockerusername>/scone-hello-world-app:0.0.1
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
    "multiaddr": "registry.hub.docker.com/<dockerusername>/scone-hello-world-app:0.0.1",
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

