# Manage confidential datasets

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Nodejs](https://nodejs.org) 10.12.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 5.0.0 or higher.
* Familiarity with the basic concepts of [Intel® SGX](intel-sgx-technology.md#intel-r-software-guard-extension-intel-r-sgx) and [SCONE](intel-sgx-technology.md#scone-framework) framework.
{% endhint %}

{% hint style="warning" %}
Please make sure you have already checked the [Quickstart](../your-first-app.md), [Your first application](../your-first-app.md) and [Build trusted applications](create-your-first-sgx-app.md) tutorials before learning how to manage confidential datasets.
{% endhint %}

Trusted Execution Environments offer a huge advantage from a security perspective. They guarantee that the behavior of execution does not change even when launched on an untrusted remote machine. The data inside this type of environment is also protected, which allows its monetization while preventing leakage.

With iExec, it is possible to authorize only applications you trust to use your datasets and get paid for it. Data is encrypted using standard encryption mechanisms and the plain version never leaves your machine. The encrypted version is made available for usage and the encryption key is pushed into the [SMS](intel-sgx-technology.md#secret-management-service-sms). After you deploy the dataset on iExec it is you, and only you, who decides which application is allowed to get the secret to decrypt it.

{% hint style="warning" %}
Datasets are only decrypted inside authorized [enclaves](intel-sgx-technology.md#enclave) and never leave them. The same thing applies to secrets.
{% endhint %}

{% hint style="info" %}
Our goal with the SMS is to run it entirely inside an enclave.
{% endhint %}

Let's see how to do all of that!

## Encrypt the dataset

Before starting let's make sure we are inside the folder `~/iexec-projects` that we created previously, during the [quick start](../quick-start-for-developers.md) tutorial.

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
├── datasets
│   ├── encrypted
│   └── original
└── .secrets
│    └── datasets
...
```

First, create your dataset folder:

```bash
mkdir datasets/original/my-first-dataset
```

We will create a dummy file that has `"Hello, world!"` as content inside `datasets/original/my-first-dataset`. Alternatively, you can put your own dataset file.

```bash
echo "Hello, world!" > datasets/original/my-first-dataset/dataset.txt
```

```bash
datasets
├── encrypted
└── original
    └── my-first-dataset
        └── dataset.txt
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
        └── dataset.txt
```

As you can see, the command generated the file `datasets/encrypted/my-first-dataset.zip`. That file is the encrypted version of your dataset, you should push it somewhere accessible because the worker will download it during the execution process. You will enter this file's URI in the `iexec.json`file \(`multiaddr` attribute\) when you will deploy your dataset. Make sure that the URI is a **DIRECT** download link \(not a link to a web page for example\).

{% hint style="info" %}
You can use Github for example to publish the file but you should add **/raw/** to the URI like this: [https://github.com/&lt;username&gt;/&lt;repo&gt;/raw/master/my-first-dataset.zip](https://github.com/<username>/<repo>/raw/master/my-first-dataset.zip)
{% endhint %}

The file `.secrets/datasets/my-first-dataset.scone.secret` is the encryption key, make sure to back it up securely. The file `.secrets/datasets/dataset.secret` is just an "alias" in the sense that it is the key of the last encrypted dataset.

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

You will get a hexadecimal address for your deployed dataset. Use that address to push the encryption key to the [SMS](intel-sgx-technology.md#secret-management-service-sms) so it is available for authorized applications.

```bash
iexec dataset push-secret <0x-your-dataset-address> --chain goerli
```

Check it by doing:

```bash
iexec dataset check-secret <0x-your-dataset-address> --chain goerli
```

We saw in this section how to encrypt a dataset with [SCONE](intel-sgx-technology.md#scone-framework) and deploy it on iExec. In addition, we learned how to push the encryption secret to the [SMS](intel-sgx-technology.md#secret-management-service-sms). Now we need to build the application that is going to consume this dataset.

## Prepare your application:

Let's create a directory tree for this app in `~/iexec-projects/`.

```bash
cd ~/iexec-projects
mkdir my-tee-dataset-app && cd my-tee-dataset-app
mkdir src
touch src/app.js # or app.py
touch Dockerfile
touch protect-fs.sh
```

In the folder `src/` create the file `app.js` \(or `app.py` if you want to use Python\) then copy this code inside:

In `app.js` \(or `app.py`\), we read the content of the dataset and write it in the result's folder \(in an artistic way using **Figlet\)**:

{% tabs %}
{% tab title="Javascript" %}
{% code title="src/app.js" %}
```javascript
const fs = require('fs');
var figlet = require('figlet');

const iexecOut = process.env.IEXEC_OUT;
const iexecIn = process.env.IEXEC_IN;
const datasetFilepath = iexecIn + "/" + process.env.IEXEC_DATASET_FILENAME;

var text = "";

// Eventually use some confidential assets
if (fs.existsSync(datasetFilepath)) {
  var dataset = fs.readFileSync(datasetFilepath);
  text = figlet.textSync(dataset);
}

// Append some results
fs.writeFileSync(iexecOut + "/result.txt", text, {flag: 'w+'}, (err) => {
  if(err) {
      throw err;
  }
});
console.log(text);

// Declare everything is computed
var computedJsonObj = { "deterministic-output-path" : iexecOut + "/result.txt" }
fs.writeFile(iexecOut + "/computed.json", JSON.stringify(computedJsonObj), {flag: 'w+'}, (err) => {
  if(err) {
      throw err;
  }
  process.exit(0)
});
```
{% endcode %}
{% endtab %}

{% tab title="Python" %}
{% code title="src/app.py" %}
```python
import os
import sys
import json
from pyfiglet import Figlet

iexec_out = os.environ['IEXEC_OUT']
iexec_in = os.environ['IEXEC_IN']
dataset_filepath = iexec_in + '/' + os.environ['IEXEC_DATASET_FILENAME']

text = ""

# Eventually use some confidential assets
if os.path.exists(dataset_filepath):
    with open(dataset_filepath, 'r') as dataset:
        text = Figlet().renderText(dataset.read())
        print(text)

# Append some results
with open(iexec_out + '/result.txt', 'w+') as fout:
    fout.write(text)

# Declare everything is computed
with open(iexec_out + '/computed.json', 'w+') as f:
    json.dump({ "deterministic-output-path" : iexec_out + '/result.txt' }, f)
```
{% endcode %}
{% endtab %}
{% endtabs %}

The Dockerfile is the same as the one we saw [previously](create-your-first-sgx-app.md#build-the-docker-image) for a trusted application:

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
COPY ./protect-fs.sh ./Dockerfile /build/
RUN sh /build/protect-fs.sh /app

ENTRYPOINT [ "node", "/app/app.js"]
```
{% endcode %}
{% endtab %}

{% tab title="Python" %}
```bash
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

And the last thing to add is the file `protect-fs.sh` :

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

## Build the docker image:

Once everything is ready we proceed to build the image. Make sure you are inside the right directory and run the following command in your terminal:

```bash
docker image build . --tag my-tee-dataset
```

At the end of the build's output, you should see this message \(with different a hash of course\):

```bash
Your application fingerprint (mrenclave) is ready:
#####################################################################
iexec.json:

"app": { 
 "owner" : ... 
 "name": ... 
  ...
 "mrenclave": "8a2e59370e47425ebaad0ba72ab06beb49ddf53aa1575c0de9a32dc82687d20c|695e1fd6bb78cc6745786d9941dda921|a8e434c81b82012c19d028ab3e7ef3adecb7786c10e5739422a7f7444e2d323c|node /app/app.js"
}
#####################################################################
Hint: Replace 'mrenclave' before doing 'iexec app deploy' step.
```

That alphanumeric string is the [fingerprint](intel-sgx-technology.md#applications-fingerprint) of your application. It allows the verification of its integrity.

## Test your app on iExec

At this stage, your application is ready to be tested on iExec. The process is similar to testing any type of application on the platform, with these minor exceptions:

* When you tag your Docker image, use `my-tee-dataset` instead of `my-hello-world`.
* Do not forget the put the application's fingerprint inside `iexec.json` \(`"mrenclave"` attribute\).
* Add the option `--tag tee` to the command `iexec app run`.
* Finally, specify the address of your deployed dataset `--dataset <0x-your-dataset-address>`.

Please go ahead and follow [these steps](../your-first-app.md#test-your-app-on-iexec) to run your application on iExec.

## Next step?

Thanks to the explained confidential computing workflow, it is possible to use an encrypted dataset with a trusted application. We can go another step further and protect the result too. See in the next chapter how to make your execution result encrypted so that you are the only one who can read it.

