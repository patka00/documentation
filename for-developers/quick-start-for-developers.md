---
description: >-
  In this tutorial we will show you how you can create decentralized application
  over the iExec infrastructure.
---

# Quick Start

{% hint style="success" %}
**Prerequisite**

* [Nodejs](https://nodejs.org) 10.12.0 or higher
{% endhint %}

iExec enables decentralized docker app deployment and monetization on the blockchain.

In this guide, we will use the iExec SDK command-line interface to deploy an iExec app on a test blockchain.

**Tutorial Steps :**

* [Create your identity on the blockchain](quick-start-for-developers.md#create-your-identity-on-the-blockchain)
* [Initialize your iExec project](quick-start-for-developers.md#initialize-your-iexec-project)
* [Deploy your app on iExec](quick-start-for-developers.md#deploy-your-app-on-iexec)
* [Run your app on iExec](quick-start-for-developers.md#run-your-app-on-iexec)
* [Publish your app on iExec marketplace](quick-start-for-developers.md#publish-your-app-on-iexec-marketplace)
* [What's next?](quick-start-for-developers.md#whats-next)

## Create your identity on the blockchain

On the blockchain, your identity is defined by your **wallet,** constisting of cryptochraphically encrypted **private key** and **public address.** What you own on the blockchain is associated with your address. The applications you deploy on iExec are associated with your wallet.

Let's set up your wallet. 

Install the iExec SDK cli \(requires [Nodejs](https://nodejs.org)\)

```text
npm i -g iexec        # sudo <cmd> if needed
```

Create a new Wallet file

```text
iexec wallet create
```

You will be asked to choose a password to protect your wallet, don't forget it since there is no way to recover it. The SDK creates a wallet file that contains a randomly generated private key encrypted by the chosen password and the derived public address. Make sure to back up the wallet file in a safe place and write down your address.

{% hint style="success" %}
Your wallet is stored in the ethereum keystore, the location depends on your OS:

* On Linux: ~/.ethereum/keystore
* On Mac : ~/Library/Ethereum/keystore
* On Windows: ~/AppData/Roaming/Ethereum/keystore

Wallet file name follow the pattern `UTC--<CREATION_DATE>--<ADDRESS>`
{% endhint %}

{% hint style="info" %}
iExec SDK uses standard Ethereum wallet, you can reuse or import existing Ethereum wallet. See iExec SDK documentation [wallet command](https://github.com/iExecBlockchainComputing/iexec-sdk#wallet) and [wallet options](https://github.com/iExecBlockchainComputing/iexec-sdk#wallet-options).
{% endhint %}

## Initialize your iExec project

Create a new folder for your iExec project and initialize the project:

```text
mkdir ~/iexec-projects
cd ~/iexec-projects
iexec init --skip-wallet
```

{% hint style="info" %}
The iExec SDK creates the minimum configuration files:

* `iexec.json` contains the project configuration
* `chain.json` contains the blockchain connection configuration
* we use `--skip-wallet` to skip wallet creation as we already created it
{% endhint %}

You can now connect to the blockchain. In the following steps we will use the **Goerli testnet**. Goerli is an Ethereum blockchain operated for testing purposes.

You can now check your wallet content and funds on Goerli:

```text
iexec wallet show --chain goerli
```

For now your wallet is empty.

### Get some test ETH

Go to [Goerli Faucet](https://goerli-faucet.slock.it/) and paste your wallet address to ask some test ETH.

Check your wallet to see if you received the Goerli ETH funds in your wallet:

```text
iexec wallet show --chain goerli
```

{% hint style="info" %}
The ETH in your wallet will allow you to pay for Ethereum blockchain transaction fees. Every time you write on the blockchain \(ie: you make a transaction\) a small amount of ETH is taken from your wallet to reward the people operating the blockchain, this mechanism protects public blockchain against spam.

[Read more about transaction fees](https://bitfalls.com/2017/12/05/ethereum-gas-and-transaction-fees-explained/)
{% endhint %}

### Initialize your remote storage

iExec enables running apps producing output files, you will need a place for storing your apps outputs.

Initialize your default remote storage:

```text
iexec storage init --chain goerli
```

{% hint style="info" %}
iExec provides a default storage solution based on [IPFS](https://ipfs.io/). This solution ensures your result to be publicly accessible through a decentralized network.

As you may don't want all your business to be exposed to the world, iExec enables both optional **RSA result encryption** and pushing results to **private storage providers**.
{% endhint %}

## Deploy your app on iExec

iExec enables decentralized deployment of dockerized applications. The applications deployed on iExec are Smart Contracts identified by their Ethereum address and referencing a public docker image. Each iExec application has an owner who can set the execution permissions on iExec platform.

Let's deploy an iExec app!

Initialize a new application

```text
iexec app init
```

The iExec SDK writes the minimum app configuration in `iexec.json`

| **key** | **description** |
| :--- | :--- |
| owner | app owner ethereum address \(default your wallet address\) |
| name | name of the application |
| type | type of application \("DOCKER" for docker container\) |
| multiaddr | download URI of the application \(a public docker registry\) |
| checksum | checksum of the app \("0x" + docker image digest\) |
| mrenclave | app fingerprint used for confidential computing use cases \(default empty\) |

{% hint style="info" %}
The default app is the public docker image [iexechub/python-hello-world](https://hub.docker.com/repository/docker/iexechub/python-hello-world)
Given an input string, the application generates an ASCII art greeting.
{% endhint %}

You can deploy this application on iExec, it will run out of the box. Where you are confident with iExec concept, you can read [Your first app](your-first-app.md) and learn how to setup your own app on iExec.

You will now deploy your app on iExec, this will be your first transaction on the blockchain:

```text
iexec app deploy --chain goerli
```

{% hint style="success" %}
While running `iexec app deploy --chain goerli` you sent your first transaction on the goerli blockchain.

You spent a small amount of ETH from your wallet to pay for this transaction, you can check you new wallet ballance with `iexec wallet show --chain goerli`
{% endhint %}

You can check your deployed apps with their index, let's check your last deployed app:

```text
iexec app show --chain goerli
```

## Run your app on iExec

iExec allows you to run applications on a decentralized infrastructure with payment in **RLC** tokens \(the native cryptocurrency of iExec\).

Let's get some test RLC to run your app

```text
iexec wallet getRLC --chain goerli
```

After a few moments, your wallet will be credited with test RLC.

You can check your wallet content

```text
iexec wallet show --chain goerli
```

Congratulation you own RLC on Goerli testnet!

Next step is to top up your **iExec Account** and use your credit to run your application.

{% hint style="info" %}
Your iExec account is your credit ready to use to pay for computation, it is managed by smart contracts \(and not owned by iExec\).

When you request an execution the price for the task is locked from your account's stake then transferred to the workers contributing to the task \(read more about [Proof of Contribution](../key-concepts/proof-of-contribution.md) protocol\).

At any time you can:

* deposit RLC from your wallet to your iExec Account
* withdraw RLC from your iExec account to your wallet \(only stake can be withdrawed\)
{% endhint %}

Top up your iExec Account

```text
iexec account deposit 200 --chain goerli
```

You moved 200 nRLC from your wallet to your account.

You can now check it with the following commands

```text
iexec account show --chain goerli
iexec wallet show --chain goerli
```

Your application is deployed, you have some RLC in your iExec Account, everything is now ready to run your application!

```text
iexec app run --args <your-name-here> --watch --chain goerli
```

{% hint style="info" %}
`iexec app run` allows to run an application on iExec at the market price.

Useful options:

* `--args <args>`  specify the app execution arguments
* `--watch`  watch execution status changes

Discover more option with `iexec app run --help`
{% endhint %}

{% hint style="success" %}
Congratulation you requested the execution of [iexechub/python-hello-world](https://hub.docker.com/repository/docker/iexechub/python-hello-world).
This will generate an ASCII art greeting with your name.
{% endhint %}

Once the task is completed copy the taskid from `iexec app run` output \(taskid is a 32Bytes hexadecimal string\). 

Download the result of your task

```text
iexec task show <taskid> --download my-result --chain goerli
```

{% hint style="info" %}
A task result is a zip file containing the output files of the application.
{% endhint %}

[iexechub/python-hello-world](https://hub.docker.com/repository/docker/iexechub/python-hello-world) produce an text file in `result.txt`.

Let's discover the result of the computation.

```text
unzip my-result.zip -d my-result
cat my-result/result.txt
```

Congratulations! You successfully executed your application on iExec!

## Publish your app on the iExec Marketplace

Your application is deployed on iExec and you completed an execution on iExec. For now, only you can request an execution of your application. The next step is to publish it on the iExec Marketplace, making it available for anyone to use.

As the owner of this application, you can define the conditions under which it can be used

{% hint style="info" %}
iExec uses orders signed by the resource owner's wallet to ensure resources governance.

The conditions to use an app are defined in the **apporder**.
{% endhint %}

Publish a new apporder for your application.

```text
iexec app publish --chain goerli
```

{% hint style="info" %}
`iexec app publish` options allows to define custom access rules to the app \(run `iexec app publish --help` to discover all the possibilities\) You will learn more about orders management later, keep the apporder default values for now.
{% endhint %}

Your application is now available for everyone on iExec marketplace on the conditions defined in apporder.

You can check the published apporders for your app

```text
iexec orderbook app <your app address> --chain goerli
```

Congratulation you just created a decentralized application! Anyone can now trigger an execution of your application on the iExec decentralized infrastructure.

* With the iexec SDK CLI `iexec app run <app address> --chain goerli`
* On iExec marketplace

## What's next?

You are now familiar with the following key iExec concepts for developers:

* Your wallet is your on-chain ID and blockchain account
* You can deploy decentralized applications on iExec
* Anyone can run tasks against payment in RLC on iExec
* Payments are processed by the decentralized platform between users' iExec Accounts
* Resource governance is managed by orders

Continue with these guides:

* [Learn how to build your first application running on iExec](your-first-app.md)
* [Learn how to manage your apporders](advanced/manage-your-apporders.md)

