---
description: >-
  In this tutorial we will show you how you can create decentralized application
  over the iExec infrastructure.
---

# Quick start

{% hint style="success" %}
**Prerequisite**

* [Nodejs](https://nodejs.org) 8.0.0 or higher
{% endhint %}

iExec enables decentralized docker app deployment and monetization on the blockchain.

In this tutorial we will use the iExec SDK command line to deploy an iExec app on a test blockchain.

**Tutorial Steps :**

* [Create your identity on the blockchain](quick-start-for-developers.md#create-your-identity-on-the-blockchain)
* [Initialize your iExec project](quick-start-for-developers.md#initialize-your-iexec-project)
* [Deploy your app on iExec](quick-start-for-developers.md#deploy-your-app-on-iexec)
* [Run your app on iExec](quick-start-for-developers.md#run-your-app-on-iexec)
* [Publish your app on iExec marketplace](quick-start-for-developers.md#publish-your-app-on-iexec-marketplace)
* [What's next?](quick-start-for-developers.md#whats-next)

## Create your identity on the blockchain

On the blockchain, your identity is defined by your **wallet**, a cryptographic pair of private key and public address. What you own on the blockchain is associated with your address. The applications you deploy on iExec are associated with your wallet.

Let's setup your wallet.

Install the iExec SDK cli \(requires [Nodejs](https://nodejs.org)\)

```text
sudo npm i -g iexec
```

Create a new Wallet file

```text
iexec wallet create
```

You will be asked to choose a password to protect your wallet, don't forget it there is no way to recover it. The SDK creates a wallet file that contains a random generated private key encrypted by the chosen password and the derived public address. Make sure to backup the wallet file in a safe place and write down your address.

{% hint style="success" %}
Your wallet is stored in the ethereum keystore, the location depends on your OS:

* On Linux: ~/.ethereum/keystore
* On Mac : ~/Library/Ethereum/keystore
* On Windows: ~/AppData/Roaming/Ethereum/keystore

Wallet file name follow the pattern `UTC--CREATION_DATE--ADDRESS`
{% endhint %}

{% hint style="info" %}
iExec SDK uses standard Ethereum wallet, you can reuse or import existing Ethereum wallet. See iExec SDK documentation [wallet command](https://github.com/iExecBlockchainComputing/iexec-sdk#wallet) and [wallet options](https://github.com/iExecBlockchainComputing/iexec-sdk#wallet-options).
{% endhint %}

## Initialize your iExec project

Create a new folder for your iexec project and initialize the project:

```text
mkdir ~/iexec-projects
cd ~/iexec-projects
iexec init --skip-wallet
```

{% hint style="info" %}
The iExec SDK creates the minimum configuration files:

* `iexec.json` contains the project configuration
* `chains.json` contains the blockchain connection configuration
* we use `--skip-wallet` to skip wallet creation as we already created it
{% endhint %}

You can now connect to the blockchain. In the following steps we will use the **Goerli testnet**. Goerli is an Ethereum blockchain operated for testing purpose.

Check your wallet content on Goerli

```text
iexec wallet show --chain goerli
```

For now your wallet is empty.

Go to [Goerli Faucet](https://goerli-faucet.slock.it/) and paste your wallet address to ask some test ETH.

Check you received some Goerli ETH in your wallet:

```text
iexec wallet show --chain goerli
```

{% hint style="info" %}
The ETH in your wallet will allow you to pay for the Ethereum blockchain transaction fees. Every time you write on the blockchain \(ie: you make a transaction\) a small amount of ETH is taken from your wallet to reward the people operating the blockchain, this mechanism protects public blockchain against spam.

[Read more about transaction fees](https://bitfalls.com/2017/12/05/ethereum-gas-and-transaction-fees-explained/)
{% endhint %}

## Deploy your app on iExec

iExec enables decentralized deployment of dockerized applications. The applications deployed on iExec are Smart Contracts identitified by their ethereum address and referencing a public docker image. Each iExec application has an owner who can set the execution permissions on iExec platform.

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

{% hint style="info" %}
The default app is the public docker image [iexechub/vanityeth](https://hub.docker.com/r/iexechub/vanityeth)

This application allows to generate an ethereum with a specific address starting pattern by generating thousands of random wallets. This process requiring high computation power is enabled by running the app on iExec.

Disclaimer: Don't use this app output wallet to store any value as output may be accessed by an untrusted actor. You will learn how to protect data in a next chapter.
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

iExec enables running applications on a decentralized infrastructure against RLC \(iExec coin\).

Let's get some test RLC to run your app

```text
iexec wallet getRLC --chain goerli
```

After a few moments your wallet will be credited with test RLC.

You can check your wallet content

```text
iexec wallet show --chain goerli
```

Congratulation you own RLC on Goerli testnet!

Next step is to topup your **iExec Account** and use your credit to run your application.

{% hint style="info" %}
Your iExec account is your compution credit managed by iExec smart contracts.

When you request an execution the price for the task is locked from your account's stake then transferred to the workers contributing to the task \(read more about [Proof of Contribution](../key-concepts/proof-of-contribution.md) protocol\).

At any time you can:

* deposit RLC from your wallet to your iExec account
* withdraw RLC from your iExec account to your wallet \(only stake can be withdrawed\)
{% endhint %}

Topup your iExec account

```text
iexec account deposit 200 --chain goerli
```

You moved 200 nRLC from your wallet to your account.

Check it whit the following commands

```text
iexec account show --chain goerli
iexec wallet show --chain goerli
```

Your application is deployed, you have some RLC on your account, everything is ready to run your application!

```text
iexec app run --params "beef" --watch --chain goerli
```

{% hint style="info" %}
`iexec app run` allows to run an application on iExec at the market price.

Useful options:

* `--params <params>`  specify the app execution params
* `--watch`  watch execution status changes

Discover more option with `iexec app run --help`
{% endhint %}

{% hint style="success" %}
Congratulation you requested the execution of [iexechub/vanityeth](https://hub.docker.com/r/iexechub/vanityeth) with the parameters `"beef"`. This should compute an Ethereum address starting with `0xbeef` .
{% endhint %}

Once the task is completed copy the taskid from `iexec app run` output \(taskid is a 32Bytes hexadecimal string\).

Download your task result

```text
iexec task show <taskid> --download my-result --chain goerli
```

{% hint style="info" %}
A task result is a zip file with tree

```text
result.zip
  ├── iexec_out/
  └── stdout.txt
```

* `stdout.txt` always contains the application logs.
* `iexec_out/` content is application specific
{% endhint %}

[iexechub/vanityeth](https://hub.docker.com/r/iexechub/vanityeth) produce an Ethereun keypair in `iexec_out/keypair.txt` . Let's discover the result of the computation.

```text
unzip my-result.zip -d my-result
cat my-result/stdout.txt
cat my-result/iexec_out/keypair.txt
```

Congratulation you successfully executed your application on iExec!

## Publish your app on iExec marketplace

Your application is deployed on iExec and you completed an execution on iExec. For now, only you can request an execution of your application. Next step is to make it available for anyone.

As owner of this application you define the conditions to use your application.

{% hint style="info" %}
iExec uses orders signed by the resource owner's wallet to ensure resources governance.

The conditions to use an app are defined in the **apporder**.
{% endhint %}

Initialize a new apporder

```text
iexec order init --app --chain goerli
```

The SDK prepares the default apporder configuration in `iexec.json`.

| **key** | **description** |
| :--- | :--- |
| app | ethereum address of the deployed app |
| appprice | application price per run |
| volume | number of execution allowed each execution decrease the remaining volume |

{% hint style="info" %}
You will learn more about orders management later, keep the apporder default values for now.
{% endhint %}

Sign the apporder with your wallet to make it valid on the blockchain

```text
iexec order sign --app --chain goerli
```

{% hint style="success" %}
The signed apporder is stored locally in `orders.json` .

Orders remains private until their publication on the marketplace. Once published, anyone matching the order condition can execute an application!
{% endhint %}

Publish the apporder on iExec marketplace to share it with others

```text
iexec order publish --app --chain goerli
```

Your application is now available for everyone on iExec marketplace on the conditions defined in apporder.

You can check the published apporders for your app

```text
iexec orderbook app <your app address> --chain goerli
```

Congratulation you just created a decentralized application! Anyone can now trigger an execution of your application on the iExec decentralized infrastructure.

* With the iexec SDK cli `iexec app run <app address> --chain goerli`
* On iExec marketplace

## What's next?

You are now familiar with the iExec key concepts for the developers:

* your wallet is your onchain ID and blockchain account
* you can deploy decentralized applications on iExec
* anyone can run tasks against RLC on iExec
* payments are processed by the decentralized platform between iExec users accounts
* resources governance is managed by orders

Continue with these articles:

* [Learn how to build your fisrt application running on iExec](your-first-app.md)
* [Learn how to manage your apporders](advanced/manage-your-apporders.md)

