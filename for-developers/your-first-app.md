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

* `/iexec_ou/determinism.iexec` file is the deterministic proof of execution \(given the same inputs this file should always be the same\).
* If your app produce output files, you must copy them in `/iexec_out/` .

{% hint style="warning" %}
Your application must always create a deterministic file named `determinism.iexec` in `/iexec_out/` as a proof of execution.

The `determinism.iexec` file is compared across replicated tasks in the [Proof of Contribution protocol](../key-concepts/proof-of-contribution.md) to achieve a consensus on workers.

* `determinism.iexec` may contain a digest of the application deterministic result.
* in case of a non-deterministic result consider producing a custom deterministic proof of the execution.
{% endhint %}

## Build your app

Create the folder tree for your application in `~/iexec-projects/`.

```text
cd ~/iexec-projects
mkdir iexec-hello-world-app
cd iexec-hello-world-app
mkdir src
touch Dockerfile
touch src/iexec-hello-world.sh
```

### Write the app \(shell script example\)

**Copy the following content** in `src/iexec-hello-world.sh` .

{% tabs %}
{% tab title="iexec-hello-world" %}
{% code title="iexec-hello-world.sh" %}
```bash
#!/bin/sh

echo "stdout is logged into stdout.txt";
echo "hello world" > /iexec_out/my-app-output.txt;
echo $@$IEXEC_NB_INPUT_FILES > /iexec_out/determinism.iexec;
```
{% endcode %}
{% endtab %}

{% tab title="hackable-iexec-hello-world" %}
{% code title="iexec-hello-world.sh" %}
```bash
#!/bin/sh

echo "APP RUNNING";
echo;

echo "INPUT DIRECTORY CONTENT";
ls -a /iexec_in;
echo;

echo "OUTPUT DIRECTORY INITIAL CONTENT";
ls -a /iexec_out;
echo;

echo "READING IEXEC ARGS";
args=$@
echo $args;
echo;

echo 'READING IEXEC RUNTIME VARIABLES';
echo ' - IEXEC_INPUT_FILES_FOLDER='$IEXEC_INPUT_FILES_FOLDER;
echo ' - IEXEC_NB_INPUT_FILES='$IEXEC_NB_INPUT_FILES;
if [ "$IEXEC_NB_INPUT_FILES" -ge 1 ]; # print IEXEC_INPUT_FILE_NAME_X
then
    i=1;
    while [ $i -le $IEXEC_NB_INPUT_FILES ]
    do
       name='IEXEC_INPUT_FILE_NAME_'$i
       eval "value=\"\$$name\""
       echo '     - '$name=$value
       i=`expr $i + 1`
    done
fi
echo ' - IEXEC_DATASET_FILENAME='$IEXEC_DATASET_FILENAME;
echo ' - IEXEC_BOT_SIZE='$IEXEC_BOT_SIZE;
echo ' - IEXEC_BOT_FIRST_INDEX='$IEXEC_BOT_FIRST_INDEX;
echo ' - IEXEC_BOT_TASK_INDEX='$IEXEC_BOT_TASK_INDEX;
echo;

echo "CREATING OUTPUT FILES IN /iexec_out/";
echo "hello world" > /iexec_out/my-app-output.txt && echo "done";
echo;

echo "CREATING determinism.iexec IN /iexec_out/";
echo $args$IEXEC_DATASET_FILENAME$IEXEC_NB_INPUT_FILES > /iexec_out/determinism.iexec && echo "done";
echo;

echo "OUTPUT DIRECTORY FINAL CONTENT";
ls -a /iexec_out;
echo;

echo "FINISH";
```
{% endcode %}
{% endtab %}
{% endtabs %}

{% hint style="info" %}
`iexec-hello-world` is the minimum shell application, learn more with `hackable-iexec-hello-world`
{% endhint %}

### Dockerize your app

**Copy the following content** in `Dockerfile` .

{% code title="Dockerfile" %}
```text
FROM alpine:latest
COPY src/iexec-hello-world.sh /iexec-hello-world.sh
RUN chmod +x /iexec-hello-world.sh
ENTRYPOINT ["/iexec-hello-world.sh"]
```
{% endcode %}

{% hint style="info" %}
Starting from the Alpine Linux image ensure we can use `/bin/sh`.

If your app requires specific dependencies, you must start from an image including these dependencies or install them.
{% endhint %}

Build the docker image.

```text
sudo docker build . --tag iexec-hello-world
```

{% hint style="success" %}
`docker build` produce an image id, using `--tag <name>` option is a convenient way to name the image to reuse it in the next steps.
{% endhint %}

Congratulation you built your first docker image for iExec!

## Test your app locally

### Basic test

Prepare local volumes for binding.

```text
mkdir ./iexec_in/
mkdir ./iexec_out/
```

Run your application locally \(container volumes bound with local volumes\).

```text
sudo docker run \
    --volume $(pwd)/iexec_in:/iexec_in \
    --volume $(pwd)/iexec_out:/iexec_out \
    iexec-hello-world \
    arg1 arg2 arg3
```

The application output files should be created in `./iexec_out/`, verify the `determinism.iexec` was created.

{% hint style="success" %}
docker run \[options\] image \[args\]

**docker run usage:**

`docker run [OPTIONS] IMAGE [COMMAND] [ARGS...]`

Use `[COMMAND]` and `[ARGS...]` to simulate the requester arguments

**useful options for iExec:**

`--volume` : Bind mount a volume. Use it to bind `/iexec_in/` and `/iexec_out/`

`--env`: Set environnement variable. Use it to simulate iExec Runtime variables
{% endhint %}

### Test with input files

Starting with the basic test you can simulate input files.

For each input file:

* Copy it in the local volume bound to `/iexec_in/` .
* Add `--env IEXEC_INPUT_FILE_NAME_x=NAME` to docker run options \(`x` is the index of the file starting by 1 and `NAME` is the name of the file\)

Add `--env IEXEC_NB_INPUT_FILES=n` to docker run options \(`n` is the total number of input files\).

Example with two inputs files:

```text
touch ./iexec_in/file1 && \
touch ./iexec_in/file2 && \
sudo docker run \
    --volume $(pwd)/iexec_in:/iexec_in \
    --volume $(pwd)/iexec_out:/iexec_out \
    --env IEXEC_INPUT_FILE_NAME_1=file1 \
    --env IEXEC_INPUT_FILE_NAME_2=file2 \
    --env IEXEC_NB_INPUT_FILES=2 \
    iexec-hello-world \
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
sudo docker tag iexec-hello-world <dockerusername>/iexec-hello-world:1.0.0
```

{% hint style="warning" %}
replace `<dockerusername>` with your docker user name
{% endhint %}

Push the image to Dockerhub.

```text
sudo docker push <dockerusername>/iexec-hello-world:1.0.0
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
* Replace app **multiaddr** with your app image download URI \(should looks like `registry.hub.docker.com/<dockerusername>/iexec-hello-world:1.0.0`\)
* Replace app **checksum** with your application image checksum \(see tip below\)

{% hint style="info" %}
The checksum of your app is the sha256 digest of the docker image prefixed with `0x` , you can use the following command to get it.

```text
docker pull <dockerusername>/iexec-hello-world:1.0.0 | grep "Digest: sha256:" | sed 's/.*sha256:/0x/'
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

With `--input-files https://example.com/file-A.txt,https://example.com/file-B.zip`  the iExec worker will download the files before running the app in `IEXEC_INPUT_FILES_FOLDER`, and let the app access them throug variables:

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
* iExec app must produce a consensus file `determinism.iexec` 
* using docker to package your app with all its dependencies
* testing an iExec app locally
* publishing on dockerhub

Resources:

* A list of iExec applications with their Docker images can be found at [https://github.com/iExecBlockchainComputing/iexec-apps](https://github.com/iExecBlockchainComputing/iexec-apps)

Continue with these articles:

* [Confidential app](confidential-computing/)
* [Learn how to manage your apporders](advanced/manage-your-apporders.md)

