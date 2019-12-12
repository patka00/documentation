---
description: >-
  In this section we will show you how you can create a Docker dapp over the
  iExec infrastructure.
---

# How set up your app

In the **task as a service** model, each time a task is launched through the iExec network, Developers set the price of their app. Requesters pay on a pay-per-task basis.And you can then withdraw your funds at anytime to your own wallet.

### Set up your app

#### Why using Docker containers?

A container is a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another.Docker Engine is the most widely used container engine. A Docker container image is a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries and settings.Docker is very convenient because it simplifies the deployment process, while ensuring consistency and repeatability in builds. Different people at different times will therefore build the same binary and obtain the same application behavior.Another feature of Docker is the possibility of creating new layers that build on top of existing images. These existing images could be yours, or images proposed by the community.

[https://docs.docker.com/storage/storagedriver/\#images-and-layers](https://docs.docker.com/storage/storagedriver/#images-and-layers)

#### Build & test your Docker image

We suppose your wallet is already created and charged with ETH to deploy your dapp and with RLC for testing.

Firstly you need to build a Docker image that contains your application.

iExec supports linux-based Docker container.

Your image will be launched by an iExec worker using following command. If you application manages dataset, during the set up of your application, the dataset must be placed in the DIR\_IN directory

```text
docker run -v ${DIR_IN}:/iexec_in -v ${DIR_OUT}:/iexec_out ${DOCKERIMAGE} ${CMDLINE}
```

Warning

Use absolute path to define ${DIR\_IN} and ${DIR\_OUT} and not a relative path.

| Parameter | Meaning |
| :--- | :--- |
| DIR\_IN | directory where datasets is downloaded during the task initialization. |
| DIR\_OUT | put all results files here. Full directory zipped in finalisation step. The result of the application \(as well as the determinism.iexec file\) should be in the iexec folder of the container. URL of the scheduler |
| DOCKERIMAGE | path to Docker image to run |
| ARGS | command to execute for the application |

A list of applications with their Docker images can be found at [https://github.com/iExecBlockchainComputing/iexec-apps](https://github.com/iExecBlockchainComputing/iexec-apps)

#### Deterministic result

iExec allows requesters to ask for a result with a predefined level of trust.For the PoCo to run smoothly and verify that different workers return the same result, some determinism is needed at some point in the execution.Since it is not always easy \(or even possible\) to have exactly the same output of a job \(for example, compute 3D rendering images on 2 different machines may produce 2 slightly different images\).The PoCo will look for determinism of a file called **determinism.iexec**.This file **has to be created by the dapp and must be deterministic**.It can contain anything but the multiple runs of the job should produce exactly the same determinism.iexec file.If not, the PoCo will not find a consensus.

**Example:**Considering a application to blur faces on pictures,the content of the determinism.iexec file could simply be the coordinates of the faces in the pictures.The output of the execution \(images with blur faces\) may not be exactly the same, but the determinism.iexec file will be.blur-face: [https://github.com/iExecBlockchainComputing/iexec-apps/tree/master/blur-face](https://github.com/iExecBlockchainComputing/iexec-apps/tree/master/blur-face)find-face: [https://github.com/iExecBlockchainComputing/iexec-apps/tree/master/find-face](https://github.com/iExecBlockchainComputing/iexec-apps/tree/master/find-face)

#### How to manage datasets

If the applications manages dataset, the dataset is downloaded at the initialization of the task

You can test your application before the deploiement locally.

Create directories iexec\_in iexec\_out and put the dataset in iexec\_in

```text
mkdir iexec_in iexec_out
cp nsfw_model.zip iexec_in/.
```

Run and test locally your application with the following command

```text
docker run -e IEXEC_DATASET_FILENAME="nsfw_model.zip" -v `pwd`/iexec_in/:/iexec_in -v `pwd`/iexec_out:/iexec_out iexechub/nsfw_prediction:1.0 https://www.w3schools.com/w3css/img_lights.jpg
```

#### Put your image in Dockerhub

You must push your image to a public repository at DockerHub. Before the execution of the task, iExec worker will pull the image from public repository.

Note

Use docker tags mechanism to manage your application versioning.

```text
docker tag iexechub/nilearn iexechub/nilearn:1.0
docker push iexechub/nilearn:1.0
```

#### Deploy your app

Once the application is available on Docker, you have to register your application on the blockchain and really create your decentralized and autonomous application, **a dapp**

Set up a configuration file.

```text
iexec init --skip-wallet
iexec app init
```

to set up the template in iexec.json and fill information for registation: name, source, …

| Parameter | Meaning |
| :--- | :--- |
| owner | the wallet address of the owner |
| name | the dapp name |
| multiaddr | docker hub address of the application |
| checksum | “0x” + sha256 of the digest of the docker image |

Get the digest sha256:

```text
docker pull iexechub/nilearn:1.0
1.0: Pulling from iexechub/nilearn
Digest: sha256:f8a48dc5125fe762e3e35b9493291b8472a68782dd19d741a7e7aa062ef73dd6
Status: Image is up to date for iexechub/nilearn:1.0
```

Don’t forget the prefix “0x” for the checksum.

0xf8a48dc5125fe762e3e35b9493291b8472a68782dd19d741a7e7aa062ef73dd6

Edit iexec.json file, set up the name, the docker address and the hash of the docker image For a docker the checksum is obtained with a docker of the image

```text
"app": {
  "owner": "0x47d0Ab8d36836F54FD9587e65125Bbab04958310",
  "name": "Nilearn",
  "type": "DOCKER",
  "multiaddr": "registry.hub.docker.com/iexechub/nilearn:1.0",
  "checksum": "0xf8a48dc5125fe762e3e35b9493291b8472a68782dd19d741a7e7aa062ef73dd6",
  "mrenclave": ""
}
```

Then deploy the app.

```text
iexec app deploy --wallet-file developper_wallet
ℹ using chain [kovan]
? Using wallet developper_wallet
Please enter your password to unlock your wallet [hidden]
✔ Deployed new app at address 0xC97b068BffDf6Cf07C25d0Cfb01Bd079EebB134D
```

#### Publish app order

Now the application registration is completed, let’s publish an order to propose the application to the market

The application order will set the price, the volume and restriction. Restriction are not mandatory.

* Create a order template

```text
iexec order init --app --wallet-file developper_wallet
ℹ using chain [kovan]
✔ Saved default apporder in "iexec.json", you can edit it:
app:                0xC97b068BffDf6Cf07C25d0Cfb01Bd079EebB134D
appprice:           0
volume:             1000000
tag:                0x0000000000000000000000000000000000000000000000000000000000000000
datasetrestrict:    0x0000000000000000000000000000000000000000
workerpoolrestrict: 0x0000000000000000000000000000000000000000
requesterrestrict:  0x0000000000000000000000000000000000000000
```

Sign the order

Edit the order part in iexec.json to describe your task,

| Parameter | Meaning |
| :--- | :--- |
| app | app address |
| appprice | app price |
| volume | number of order created, each usage decrease this number |
| tag | not use |
| datasetrestrict: | restricted to a dataset \(1\) |
| workerpoolrestrict | restricted to a workerpool \(1\) |
| requesterrestrict: | restricted to a requester \(1\) |

1. the restriction is disabled by default with 0x0000000000000000000000000000000000000000

```text
iexec order sign --app --wallet-file developper_wallet
ℹ using chain [kovan]
? Using wallet developper_wallet
Please enter your password to unlock your wallet [hidden]
✔ apporder signed and saved in orders.json, you can share it:
app:                0xC97b068BffDf6Cf07C25d0Cfb01Bd079EebB134D
appprice:           0
volume:             1000000
tag:                0x0000000000000000000000000000000000000000000000000000000000000000
datasetrestrict:    0x0000000000000000000000000000000000000000
workerpoolrestrict: 0x0000000000000000000000000000000000000000
requesterrestrict:  0x0000000000000000000000000000000000000000
salt:               0xda9180521bb3eb495e5fc9723d351199324b96481cdd85e9f7004477911045f0
sign:               0xad835e8b86ccb9b44d3704fd64166da648927adf9dc88e96931de388033fb178192ee52a8c665fefe66b99296e299226d0f047aa8fb5bd87b7b165374154e3c51c
```

Publish the order

```text
iexec order publish --app --wallet-file developper_wallet
ℹ using chain [kovan]
? Using wallet developper_wallet
Please enter your password to unlock your wallet [hidden]
? Do you want to publish the following apporder?
app:                0xC97b068BffDf6Cf07C25d0Cfb01Bd079EebB134D
appprice:           0
volume:             1000000
tag:                0x0000000000000000000000000000000000000000000000000000000000000000
datasetrestrict:    0x0000000000000000000000000000000000000000
workerpoolrestrict: 0x0000000000000000000000000000000000000000
requesterrestrict:  0x0000000000000000000000000000000000000000
salt:               0xda9180521bb3eb495e5fc9723d351199324b96481cdd85e9f7004477911045f0
sign:               0xad835e8b86ccb9b44d3704fd64166da648927adf9dc88e96931de388033fb178192ee52a8c665fefe6
6b99296e299226d0f047aa8fb5bd87b7b165374154e3c51c
 Yes
✔ apporder successfully published with orderHash 0x2d09cc3e08e675fc290b683aa376b7038d1762f31674e97baaaa723a0e879fdc
```

Now the application is available.

Check out [http://explorer.iex.ec](http://explorer.iex.ec/)

Go to the [Quick start](https://docs.iex.ec/quickstart.html) section to learn how to test your dapp .

#### Variables available at the runtime

When a worker triggers the computation of a task, a few variables are available to the application that is running. They can be used by the application.

> **General variables**

Those variables are available in the container performing the computation of a task:

| Variables | Meaning |
| :--- | :--- |
| IEXEC\_DATASET\_FILENAME | name of the dataset filename that is in the description of task |
| IEXEC\_INPUT\_FILES\_FOLDER | name of the folder \(in the container\) where are all the input files and dataset |
| IEXEC\_NB\_INPUT\_FILES | number of input files described in the task and that have been downloaded to IEXEC\_INPUT\_FILES\_FOLDER |
| IEXEC\_INPUT\_FILE\_NAME\_1 | name of the first input file in the list of input files given in parameters of the task. |
| IEXEC\_INPUT\_FILE\_NAME\_2 | name of the second input file in the list of input files given in parameters of the task. |
| IEXEC\_INPUT\_FILE\_NAME\_n | name of the nth input file in the list of input files given in parameters of the task. |

There will be as many IEXEC\_INPUT\_FILE\_NAME\_\* variables as there are input files in the parameters of the task.

> **BoT variables**

Some additional variables are available regarding the Bag Of Task, in order for the worker to know which part of the BoT it is processing:

| Variables | Meaning |
| :--- | :--- |
| IEXEC\_BOT\_SIZE | Size of the BoT, which means that it is the number of Tasks contained in the BoT. |
| IEXEC\_BOT\_FIRST\_INDEX | Index of the first task in the BoT. |
| IEXEC\_BOT\_TASK\_INDEX | Index of the current task that is being processed. |

### Provide a dataset

In this section we will show you how you can propose a dataset or any valuable data over iExec infrastructure.In the **task-as-a-service** model, each time a task is launched through the iExec network,The dataset providers set the price of their datasets. Requesters pay on a pay-per-task basis.And you can then withdraw your funds at anytime to your own wallet.

Whitelisting and ordering Dataset owner will manage:

> * who can process the dataset
> * which application can run the dataset
> * make restriction for computing resources
> * set up a cutting-edge pricing management

#### Deploy your dataset

Zip your dataset, or model.

Put the data on public data storage, the dataset must be accessible in direct download.

Set up the iexec.json configuration file.

```text
iexec init --skip-wallet
iexec dataset init
```

Edit the iexec.json to describe your dataset.

```text
 "dataset": {
  "owner": "0x9CdDC59c3782828724f55DD4AB4920d98aA88418",
  "name": "Neurovault_brainstatsmaps",
  "multiaddr": "https://raw.githubusercontent.com/ericr6/nilearn/master/nilearn_data.zip",
  "checksum": "0x0000000000000000000000000000000000000000000000000000000000000000"
},
```

Then you deploy your dataset:

```text
iexec dataset deploy --wallet-file data_owner_wallet
ℹ using chain [kovan]
? Using wallet data_owner_wallet
Please enter your password to unlock your wallet [hidden]
✔ Deployed new dataset at address 0xCb781f3106E25E2A9408C4B89C47034877223D12
```

#### Publish a dataset order

* Create an order template

```text
iexec order init --dataset --wallet-file developper_wallet
```

Edit the order part in iexec.json to describe the dataset.

| Parameter | Meaning |
| :--- | :--- |
| dataset | dataset address |
| datasetprice | dataset price |
| volume | number of order created |
| tag | tag for extra computational requirement \(1\) |
| dapprestrict: | restricted to an application defined by its address \(1\) |
| workerpoolrestrict | restricted to a workerpool defined by its address \(1\) |
| requesterrestrict: | restricted to a requester defined by its address \(1\) |

1. the restriction is disabled by default with 0x0000000000000000000000000000000000000000

The volume is the total number of tasks allowed within the order created. Once all the volume is consumed, the dataset won’t be available, the dataset owner has to publish a new datasetorder:

```text
"datasetorder": {
  "dataset": "0xCb781f3106E25E2A9408C4B89C47034877223D12",
  "datasetprice": 2,
  "volume": 1000000,
  "tag": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "apprestrict": "0x0000000000000000000000000000000000000000",
  "workerpoolrestrict": "0x0000000000000000000000000000000000000000",
  "requesterrestrict": "0x0000000000000000000000000000000000000000"
}
```

Sign the order

```text
iexec order sign --dataset --wallet-file data_owner_wallet
ℹ using chain [kovan]
? Using wallet data_owner_wallet
Please enter your password to unlock your wallet [hidden]
✔ datasetorder signed and saved in orders.json, you can share it:
dataset:            0xCb781f3106E25E2A9408C4B89C47034877223D12
datasetprice:       2
volume:             1000000
tag:                0x0000000000000000000000000000000000000000000000000000000000000000
apprestrict:        0x0000000000000000000000000000000000000000
workerpoolrestrict: 0x0000000000000000000000000000000000000000
requesterrestrict:  0x0000000000000000000000000000000000000000
salt:               0xaaae00a749e198b9f43bc89c420aaf146f3a224c8500d327c3569075eea2c2ae
sign:               0x87f720bb9e09762257bd62561f52b22237b2982397cb8aae19e84adf8afcb4d21f758f40dcc001a5dd018aaf48ccfd59a91f3c18adcb27c414da44436bea8c931b
```

Publish the order

```text
iexec order publish --dataset --wallet-file data_owner_wallet
ℹ using chain [kovan]
? Using wallet developper_wallet
Please enter your password to unlock your wallet [hidden]
? Do you want to publish the following apporder?
app:                0xC97b068BffDf6Cf07C25d0Cfb01Bd079EebB134D
appprice:           0
volume:             1000000
tag:                0x0000000000000000000000000000000000000000000000000000000000000000
datasetrestrict:    0x0000000000000000000000000000000000000000
workerpoolrestrict: 0x0000000000000000000000000000000000000000
requesterrestrict:  0x0000000000000000000000000000000000000000
salt:               0xda9180521bb3eb495e5fc9723d351199324b96481cdd85e9f7004477911045f0
sign:               0xad835e8b86ccb9b44d3704fd64166da648927adf9dc88e96931de388033fb178192ee52a8c665fefe6
6b99296e299226d0f047aa8fb5bd87b7b165374154e3c51c
 Yes
✔ apporder successfully published with orderHash 0x2d09cc3e08e675fc290b683aa376b7038d1762f31674e97baaaa723a0e879fdc
```

Now the dataset is available.

### Dataset encryption

As a dataset provider, you might want to protect your dataset with encryption in order to monetize it. Any encrypted dataset will be decrypted on worker resources with a dataset secret key retrieved from the Secret Management Service. This dataset secret key need to be created and push by the dataset owner. At this point, the decrypted dataset will be ready to be used by the app.

See the SDK tutorial for more info.

First, initialize the folder structure

```text
iexec dataset init --encrypted

ℹ Created dataset folder tree for encryption
✔ Saved default dataset in "iexec.json", you can edit it:
owner:     0x62F2a967EaF91976763B96E515E4014a5526b6D3
name:      my-dataset
multiaddr: /ipfs/QmW2WQi7j6c7UgJTarActp7tDNikE4B2qXtFCfLPdsgaTQ
checksum:  0x0000000000000000000000000000000000000000000000000000000000000000
```

This will create the template for the dataset info in the _iexec.json_ and the following folders:

```text
├── datasets
│   ├── encrypted
│   └── original
└── .secrets
    └── datasets
```

Copy your dataset in the _datasets/original/_ folder, then encrypt it with the SDK:

Generate a secret key for each file or folder in dataset/original/ and encrypt it

```text
iexec dataset encrypt
```

It produces the secret key for decrypting the dataset

```text
cat ./.secrets/dataset/myAwsomeDataset.file.secret
```

and the encrypted dataset, you must share at a public url

```text
ls -lh ./datasets/encrypted/myAwsomeDataset.file.enc
```

Securely share the dataset secret key \(Encrypted datasets only\)

Disclaimer: The secrets pushed in the Secret Management Service will be shared with the worker to process the dataset in the therms your specify in the dataset order. Make sure to always double check your selling policy in the dataset order before signing it.

Push the secret in the Secret Management Service \(sms\)

```text
iexec dataset push-secret
```

Go to the [Quick start](quick-start-for-developers/) section to learn how to test a dapp .

