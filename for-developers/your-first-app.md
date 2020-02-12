---
description: >-
  In this section we will show you how you can create a Docker dapp over the
  iExec infrastructure.
---

# Your First App

In this tutorial we will prepare an iExec app based on an existing docker image and we will run it on iExec decentralized infrastructure.

**Tutorial Steps :**

* Understand what is an iExec decentralized application?
* Application I/O
* Build your application
* Test your application locally
* Run your application on iExec
* What's next?

**prerequisite:**

* Completed [Quick Dev Start](quick-start-for-developers.md)
* Ethereum wallet charged with goerli ETH an RLC
* docker installed
* [Dockerhub](https://hub.docker.com/) account

## Understand what is an iExec decentralized application?

iExec leverage [Docker](https://www.docker.com/why-docker) containers to ensure the execution of your application on a decentralized infrastructure. iExec supports Linux-based docker images.

### Why using Docker containers?

* Docker Engine is the most **widely used** container engine. 
* A Docker container image is a **standard** unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another. This allows to **run on any worker** connected to the decentralized infrastructure.
* Docker also enable creating new layers on top of existing images. This allows to **easily build  iExec apps** on the top of existing docker images.

### What kind of application can I build on iExec?

Today you can run any application as a task. This mean services are not supported for now. 

## Application I/O

This is an overview of an iExec application inputs and expected outputs. You probably don't have to deeply understand every part of this section to build your app but you will find some 

### Application args

The requester specify the arguments to use with an application in the requestorder, theses arguments are forwarded as is to the application.

### Application input files

Your app may use input files, all the input files specified by the requester will be downloaded in the container directory `/iexec_in` before running your application.

#### Input files \(public\):

Input files contain non sensitive data publicly available on the Internet. The requester may specify any number of input files in the requestorder.

For each input file, the variable `IEXEC_INPUT_FILE_NAME_x` is set to the file name \(`x` is the index of the file starting with `1`\).  The total number of input files is stored in the variable.

Use these variables in your application to find input files to process. \(first input file path is `/iexec_in/$IEXEC_INPUT_FILE_NAME_1`\)

#### Datasets \(confidential input files\):

Datasets are encrypted files available only in a Trusted Execution Environment \(TEE\). Your will learn how to deal with datasets in the next tutorial.

Similarly to input files, the dataset name is stored in `IEXEC_DATASET` variable.

### Runtime variables

The runtime variables are environment variables set by the iExec worker and available for your application.

#### Input files variables

Use these variables if your app deals with input files

<table>
  <thead>
    <tr>
      <th style="text-align:left">Name</th>
      <th style="text-align:left">Type</th>
      <th style="text-align:left">Content</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="text-align:left">IEXEC_INPUT_FILES_FOLDER</td>
      <td style="text-align:left">path</td>
      <td style="text-align:left">Absolute path of iexec input folder (<code>/iexec_in/</code>)</td>
    </tr>
    <tr>
      <td style="text-align:left">IEXEC_NB_INPUT_FILES</td>
      <td style="text-align:left">int &gt;= 0</td>
      <td style="text-align:left">Total number of input files</td>
    </tr>
    <tr>
      <td style="text-align:left">IEXEC_INPUT_FILE_NAME_x</td>
      <td style="text-align:left">string or unset</td>
      <td style="text-align:left">
        <p>Name of the input file indexed by x (<code>x</code>
        </p>
        <p>starts with <code>1</code>)</p>
      </td>
    </tr>
    <tr>
      <td style="text-align:left">IEXEC_DATASET_FILENAME</td>
      <td style="text-align:left">string or unset</td>
      <td style="text-align:left">Name of the dataset file if used</td>
    </tr>
  </tbody>
</table>#### Bag of Tasks variables

Use these variables to index tasks in parallelization use cases.

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

The `determinism.iexec`is used in the Proof of Contribution protocol to achieve a consensus on replicated tasks.
{% endhint %}

## Build your application

Create a directory for your application.

```text
mkdir iexec-hello-world-app && cd iexec-hello-world-app
```

### Write the application \(shell script example\)

Create the app file `iexec-hello-world.sh`

```bash
touch iexec-hello-world.sh
```

**Copy the following content** in `iexec-hello-world.sh` .

{% tabs %}
{% tab title="iexec-hello-word.sh" %}
```bash
#!/bin/sh

echo "stdout is logged into stdout.txt";
echo "hello world" > /iexec_out/my-app-output.txt;
echo $@$IEXEC_NB_INPUT_FILES > /iexec_out/determinism.iexec;

```
{% endtab %}

{% tab title="hackable-iexec-hello-world.sh" %}
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
{% endtab %}
{% endtabs %}

{% hint style="info" %}
`iexec-hello-world.sh` is the minimum shell application, learn more with `hackable-iexec-hello-world.sh` 
{% endhint %}

### Dockerize your application

Create the `Dockerfile` file to describe your app docker image.

```text
touch Dockerfile
```

**Copy the following content** in `Dockerfile` .

```text
FROM alpine:latest
COPY iexec-hello-world.sh /iexec-hello-world.sh
RUN chmod +x /iexec-hello-world.sh
ENTRYPOINT ["/iexec-hello-world.sh"]
```

{% hint style="info" %}
Starting from the Alpine Linux image ensure we can use `/bin/sh`.

If your app requires specific dependencies, you must start from an image including these dependencies or install them.
{% endhint %}

Build the docker image.

```text
sudo docker build . --tag iexec-hello-world
```

{% hint style="success" %}
`docker build` produce an image id, using `--tag <name>`  option is a convenient way to name the image to reuse it in the next steps.
{% endhint %}

**Congratulation you built your first docker image for iExec!**

## Test your application locally

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

## Test your application on iExec

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

Go back to the iexec project folder created in the previous tutorial.

You will need a few configuration in `iexec.json` to deploy your app:

* Replace app **name** with your application name \(display only\)
* Replace app **multiaddr** with your app image download URI \(should looks like `registry.hub.docker.com/dockerusername/iexec-hello-world:1.0.0`\)
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

Once the run is completed copy the taskid to download and check the result

```text
iexec task show <taskid> --download my-app-result --chain goerli  \
    && unzip my-app-result.zip -d my-app-result
```

Congratulation your app succesfully ran on iExec!

## Publish your app on iExec marketplace

```text
iexec order init --app --chain goerli
iexec order sign --app --chain goerli
iexec order publish --app --chain goerli
```

**Congratulation your application is now available on iExec!**

## Whats next?

In this tutorial you learnt about the key concepts for deploying an app on iExec:

* application inputs and outputs
* iExec consensus file `determinism.iexec` 
* using docker to package your app with all its dependencies
* testing an iExec app locally
* publishing on dockerhub

Resources:

* A list of iExec applications with their Docker images can be found at [https://github.com/iExecBlockchainComputing/iexec-apps](https://github.com/iExecBlockchainComputing/iexec-apps)

Continue with these articles:

* Confidential app
* [Learn how to manage your apporders](manage-your-apporders.md)

