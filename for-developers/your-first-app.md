---
description: >-
  In this section we will show you how you can create a Docker dapp over the
  iExec infrastructure.
---

# Your first application

{% hint style="success" %}
**Prerequisities**

* [Docker](https://docs.docker.com/install/) 17.05 or higher on the daemon and client.
* [Dockerhub](https://hub.docker.com/) account.
* [Nodejs](https://nodejs.org) 10.12.0 or higher.
* [iExec SDK](https://www.npmjs.com/package/iexec) 5.0.0 or higher.
* [Quick start](https://github.com/iExecBlockchainComputing/documentation/tree/651ca324fe3b9baf7e88a87401f74168e519ee83/quick-start-for-developers.md) tutorial completed
* Ethereum wallet charged with Goerli ETH an RLC
{% endhint %}

In this tutorial we will prepare an iExec app based on an existing docker image and we will run it on iExec decentralized infrastructure.

**Tutorial Steps :**

* [Understand what is an iExec decentralized application?](your-first-app.md#understand-what-is-an-iexec-decentralized-application)
* [Application I/O](your-first-app.md#application-i-o)
* [Build your app](your-first-app.md#build-your-app)
* [Test your app locally](your-first-app.md#test-your-app-locally)
* [Test your app on iExec](your-first-app.md#test-your-app-on-iexec)
* [Publish your app on iExec marketplace](your-first-app.md#publish-your-app-on-iexec-marketplace)
* [What's next?](your-first-app.md#whats-next)

## Understand what is an iExec decentralized application?

iExec leverage [Docker](https://www.docker.com/why-docker) containers to ensure the execution of your application on a decentralized infrastructure. iExec supports Linux-based docker images.

### Why using Docker containers?

* Docker Engine is the most **widely used** container engine. 
* A Docker container image is a **standard** unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another. This allows to **run on any worker** connected to the decentralized infrastructure.
* Docker also enable creating new layers on top of existing images. This allows to **easily build  iExec apps** on the top of existing docker images.

### What kind of application can I build on iExec?

Today you can run any application as a task. This mean services are not supported for now.

## Application I/O

This is an overview of an iExec application inputs and expected outputs. You probably don't have to deeply understand every part of this section to build your app, just pick what you need.

### Application args

The requester may specify the arguments to use with an application in the requestorder, theses arguments are forwarded as is to the application.

### Application input files

Your app may use input files, all the input files specified by the requester will be downloaded in the container directory `/iexec_in` before running your application.

#### Input files \(public\):

Input files contain non sensitive data publicly available on the Internet. The requester may specify any number of input files in the requestorder.

For each input file, the variable `IEXEC_INPUT_FILE_NAME_x` is set to the file name \(`x` is the index of the file starting with `1`\). The total number of input files is stored in the variable.

Use these variables in your application to find input files to process. \(first input file path is `/iexec_in/$IEXEC_INPUT_FILE_NAME_1`\)

#### Datasets \(confidential input files\):

Datasets are encrypted files available only in a Trusted Execution Environment \(TEE\). Your will learn how to deal with datasets in the next tutorial.

Similarly to input files, the dataset name is stored in `IEXEC_DATASET` variable. A single dataset file is currently supported.

### Runtime variables

The runtime variables are environment variables set by the iExec worker and available for your application.

#### Input files variables

Use these variables if your app deals with input files

| Name | Type | Content |
| :--- | :--- | :--- |
| IEXEC\_INPUT\_FILES\_FOLDER | path | Absolute path of iexec input folder \(`/iexec_in/`\) |
| IEXEC\_NB\_INPUT\_FILES | int &gt;= 0 | Total number of input files |
| IEXEC\_INPUT\_FILE\_NAME\_x | string or unset | Name of the input file indexed by x \(`x` starts with `1`\) |
| IEXEC\_DATASET\_FILENAME | string or unset | Name of the dataset file if used |

#### Bag of Tasks variables

The requester may request multiple tasks in a single transaction \(Bag of Tasks\), each task of the bag is given a unique index. If you intend to support running Bag of Tasks in your app, you can use the following variables to index tasks in parallelization use cases.

| Name | Type | Content |
| :--- | :--- | :--- |
| IEXEC\_BOT\_TASK\_INDEX | int &gt;= 0 | Index of the current task |
| IEXEC\_BOT\_FIRST\_INDEX | int &gt;= 0 | Index of the first task in the current Deal \(Bag of task subset\) |
| IEXEC\_BOT\_SIZE | int &gt;= 1 | Total number of parallelized tasks in a Bag of Tasks |

### Application outputs

An iExec app produce a `result.zip` file for the requester with the following tree:

```text
result.zip
  ├── iexec_out/
  └── stdout.txt
```

`stdout.txt` contains the logs of your application. This file is auto generated by the iExec worker.

`iexec_out` is a copy at the final state of your app container directory `/iexec_out/` final state, your application must create the following files in `/iexec_out/` :

* `/iexec_ou/computed.json` file should be created when your computing is over. It contains at least a field `deterministic-output-path` which is the path of the deterministic portion of your results. It should be either a file or a non-empty folder. It is required for the proof of execution \(given the same inputs this file should always be the same\).
* If your app produce output files, you must copy them in `/iexec_out/` .

{% hint style="warning" %}
Your application must always create a `computed.json` file in `/iexec_out` as a proof of execution which could look like `{ "deterministic-output-path" : "/iexec_out/result.txt" }`

The `computed.json` file is compared across replicated tasks in the [Proof of Contribution protocol](../key-concepts/proof-of-contribution.md) to achieve a consensus on workers.
{% endhint %}

## Build your app

Create the folder tree for your application in `~/iexec-projects/`.

```text
cd ~/iexec-projects
mkdir my-hello-world-app
cd my-hello-world-app
mkdir src
touch Dockerfile
```

### Write the app \(JavaScript script example\)

Following examples only feature Javascript and Python use cases for simplicity concerns, but remember that you can run on iExec anything which is Dockerizable.

**Copy the following content** in `src/` .

{% tabs %}
{% tab title="JavaScript" %}
{% code title="src/app.js" %}
```javascript
const fs = require('fs');

const iexec_in = process.env.IEXEC_IN
const iexec_out = process.env.IEXEC_OUT

// Do whatever you want (let's write hello world here)
var text = "Hello, World!"
if (process.argv.length > 2){
  text = 'Hello, ' + process.argv[2] + '!'
}

// Append some results in /iexec_out/
fs.writeFileSync(iexec_out + "/result.txt", text, {flag: 'w+'}, (err) => {
  if(err) {
      throw err;
  }
});
console.log(text);

// Declare everything is computed
var computedJsonObj = { "deterministic-output-path" : iexec_out + "/result.txt" }
fs.writeFile(iexec_out + "/computed.json", JSON.stringify(computedJsonObj), {flag: 'w+'}, (err) => {
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

iexec_in = os.environ['IEXEC_IN']
iexec_out = os.environ['IEXEC_OUT']

# Do whatever you want (let's write hello world here)
text = "Hello, World!"
if len(sys.argv) > 1:
    text = 'Hello, {}!'.format(sys.argv[1])

# Append some results in /iexec_out/
with open(iexec_out + '/result.txt', 'w+') as fout:
    fout.write(text)
    print(text)

# Declare everything is computed
with open(iexec_out + '/computed.json', 'w+') as f:
    json.dump({ "deterministic-output-path" : iexec_out + '/result.txt' }, f)
```
{% endcode %}
{% endtab %}
{% endtabs %}

### Dockerize your app

**Copy the following content** in `Dockerfile` .

{% tabs %}
{% tab title="JavaScript" %}
{% code title="Dockerfile" %}
```text
FROM node:8.9.4
### install your dependencies if you have some
RUN mkdir /app && cd /app && npm install figlet
COPY ./src /app
ENTRYPOINT [ "node", "/app/app.js"]
```
{% endcode %}
{% endtab %}

{% tab title="Python" %}
{% code title="Dockerfile" %}
```text
FROM python:3.7.3-alpine3.10
### install python dependencies if you have some
RUN pip3 install pyfiglet
COPY ./src /app
ENTRYPOINT ["python", "/app/app.py"]
```
{% endcode %}
{% endtab %}
{% endtabs %}

Build the docker image.

```text
sudo docker build . --tag my-hello-world
```

{% hint style="success" %}
`docker build` produce an image id, using `--tag <name>` option is a convenient way to name the image to reuse it in the next steps.
{% endhint %}

Congratulation you built your first docker image for iExec!

## Test your app locally

### Basic test

Prepare local volumes for binding.

```text
mkdir -p /tmp/iexec_in
mkdir -p /tmp/iexec_out
```

Prepare to run your application locally \(container volumes bound with local volumes\).

{% code title="run" %}
```text
#!/bin/sh
cd $(dirname $0)

IEXEC_IN=/tmp/iexec_in
IEXEC_OUT=/tmp/iexec_out
rm -rf $IEXEC_OUT
mkdir -p $IEXEC_OUT

sudo docker run --rm \
    -e IEXEC_IN=/iexec_in \
    -e IEXEC_OUT=/iexec_out \
    -v /tmp/iexec_in:/iexec_in \
    -v /tmp/iexec_out:/iexec_out \
    my-hello-world $@

echo
find $IEXEC_OUT
```
{% endcode %}

Run it locally

```text
./run arg1 arg2 arg3
```

{% hint style="success" %}
docker run \[options\] image \[args\]

**docker run usage:**

`docker run [OPTIONS] IMAGE [COMMAND] [ARGS...]`

Use `[COMMAND]` and `[ARGS...]` to simulate the requester arguments

**useful options for iExec:**

`-v` : Bind mount a volume. Use it to bind `/iexec_in` and `/iexec_out`

`-e`: Set environnement variable. Use it to simulate iExec Runtime variables
{% endhint %}

### Test with input files

Starting with the basic test you can simulate input files.

For each input file:

* Copy it in the local volume bound to `/iexec_in` .
* Add `-e IEXEC_INPUT_FILE_NAME_x=NAME` to docker run options \(`x` is the index of the file starting by 1 and `NAME` is the name of the file\)

Add `-e IEXEC_NB_INPUT_FILES=n` to docker run options \(`n` is the total number of input files\).

Example with two inputs files:

```text
touch /tmp/iexec_in/file1 && \
touch /tmp/iexec_in/file2 && \
sudo docker run \
    -v /tmp/iexec_in:/iexec_in \
    -v /tmp/iexec_out:/iexec_out \
    -e IEXEC_INPUT_FILE_NAME_1=file1 \
    -e IEXEC_INPUT_FILE_NAME_2=file2 \
    -e IEXEC_NB_INPUT_FILES=2 \
    my-hello-world \
    arg1 arg2 arg3
```

## Test your app on iExec

### Push your app to Dockerhub

Login to your Dockerhub account.

```text
sudo docker login
```

Tag you application image to push it to your dockerhub public repository.

```text
sudo docker tag my-hello-world <dockerusername>/my-hello-world:1.0.0
```

{% hint style="warning" %}
replace `<dockerusername>` with your docker user name
{% endhint %}

Push the image to Dockerhub.

```text
sudo docker push <dockerusername>/my-hello-world:1.0.0
```

**Congratulation, you app is ready to be deployed on iExec!**

### Deploy your app on iExec

You already learnt how to deploy the default app on iExec in the [previous tutorial](quick-start-for-developers.md).

Go back to the `iexec-project` folder.

```text
cd ~/iexec-projects/
```

You will need a few configuration in `iexec.json` to deploy your app:

* Replace app **name** with your application name \(display only\)
* Replace app **multiaddr** with your app image download URI \(should looks like `registry.hub.docker.com/<dockerusername>/my-hello-world:1.0.0`\)
* Replace app **checksum** with your application image checksum \(see tip below\)

{% hint style="info" %}
The checksum of your app is the sha256 digest of the docker image prefixed with `0x` , you can use the following command to get it.

```text
docker pull <dockerusername>/my-hello-world:1.0.0 | grep "Digest: sha256:" | sed 's/.*sha256:/0x/'
```
{% endhint %}

Deploy your app on iExec

```text
iexec app deploy --chain goerli
```

Verify the deployed app \(name, multiaddr, checksum, owner\)

```text
iexec app show --chain goerli
```

### Run your app on iExec

Before requesting an execution make sure your account stake is charged with Goerli RLC

```text
iexec account show --chain goerli
```

Run your application on iExec

```text
iexec app run --watch --chain goerli
```

{% hint style="info" %}
**Using arguments:**

You can pass arguments to the app using `--args <args>` option.

With `--args "dostuff --with-option"` the app will receive `["dostuff", "--with-option"]` as process args.

**Using input files:**

You can pass input files to the app using `--input-files <list of URL>` option.

With `--input-files https://example.com/file-A.txt,https://example.com/file-B.zip` the iExec worker will download the files before running the app in `IEXEC_INPUT_FILES_FOLDER`, and let the app access them throug variables:

* `file-A.txt` as`IEXEC_INPUT_FILE_NAME_1`
* `file-B.zip` as`IEXEC_INPUT_FILE_NAME_2`
{% endhint %}

Once the run is completed copy the taskid from `iexec app run` output to download and check the result

```text
iexec task show <taskid> --download my-app-result --chain goerli  \
    && unzip my-app-result.zip -d my-app-result
```

Congratulation your app successfully ran on iExec!

## Publish your app on iExec marketplace

```text
iexec app publish --chain goerli
```

**Congratulation your application is now available on iExec!**

## Whats next?

In this tutorial you learnt about the key concepts for building an app on iExec:

* iExec app inputs and outputs
* iExec app must produce a `computed.json` file \(required for the proof of execution\)
* using docker to package your app with all its dependencies
* testing an iExec app locally
* publishing on dockerhub

Resources:

* A list of iExec applications with their Docker images can be found at [https://github.com/iExecBlockchainComputing/iexec-apps](https://github.com/iExecBlockchainComputing/iexec-apps)

Continue with these articles:

* [Confidential app](confidential-computing/)
* [Learn how to manage your apporders](advanced/manage-your-apporders.md)

