---
description: >-
  PoCo is a protocol designed to provide trust in an open and decentralized
  environment of untrusted machines.
---

# Proof of Contribution

rThe iExec platform provides a network where application provider, workers, and users can gather and work together. The fully decentralized nature of iExec implies that no single agent is trusted by default, and that those agents require incentives to contribute correctly.

In this context, Proof-of-Contribution \(PoCo\) is the protocol used by iExec for consensus over off-chain computing.

## Protocol

### Objectives

PoCo is a protocol designed to provide trust in an open and decentralized environment of untrusted machines.

In addition to providing trust, PoCo also orchestrates the different contributions to the iExec network, ensuring payments are always fair and timely.

A major quality of PoCo lies in the fact that it is a modular protocol. It comes with features that are context-specific.

### **Result consolidation**

> PoCo relies on replication to achieve result consolidation. This is a purely software solution that enforces a confidence level on the result. [This confidence level can be customized by the requester.](proof-of-contribution.md#replication-and-trust)
>
> This layer also supports the onchain consolidation of execution results carried out in Trusted Execution Environments \(TEE\) such as Intel SGX.

### **Secure payment**

> Once a deal is sealed on the iExec Marketplace, requester funds are locked to ensure all resource providers are paid for their contributions. Resources can take the form of data, applications or computing power.
>
> Workers must achieve consensus on the execution result to get the requester’s funds. If consensus is not achieved, the requester is reimbursed.
>
> Worker and scheduler must stake RLC to participate as a computing providers. Bad behaviour from an actor results in a loss of stake.
>
> This is essential on the public blockchain, but all values can be set to 0 for private blockchain solutions.

### **Permissioning**

> For an execution to happen, a deal must be signed between the different parties involved. A permission mechanism can be used to control access to applications, datasets and worker pools.
>
> The secure payment layer can be disabled for a private blockchain, or it can also be used in the context of the public blockchain to increase security. An example of permissioning is dataset restriction for a specific application.

### Overview

PoCo describes the succession of contributions that are required to achieve consensus on a given result. Its logic is detailed in two blog articles:

* [PoCo series \#1: Initial PoCo description](https://medium.com/iex-ec/about-trust-and-agents-incentives-4651c138974c)
* [PoCo series \#3: Updated PoCo description](https://medium.com/iex-ec/poco-series-3-poco-protocole-update-a2c8f8f30126)

The [nominal workflow](https://github.com/iExecBlockchainComputing/iexec-doc/raw/master/techreport/nominalworkflow-ODB.png) is also available in the [technical report section](../help/glossary.md)

Below are the details of the implementations:

1. **Deal**

> [A deal is sealed by the Clerk. ](proof-of-contribution.md#brokering)This marks the beginning of the execution. An event is created to notify the worker pool’s scheduler.
>
> The consensus timer starts when the deal is signed. The corresponding task must be completed before the end of this countdown. Otherwise, the scheduler gets punished by a loss of stake and reputation, and the user reimbursed.

1. **Initialization**

> The scheduler calls the `initialize` method. Given a deal id and a position in the request order \(within the deal window\), this function initializes the corresponding task and returns the _taskid_.
>
> `bytes32 taskid = keccak256(abi.encodePacked(_dealid, idx));`

1. **Authorization signature**

> The scheduler designates workers that participate to this task. The scheduler’s Ethereum wallet signs a message containing the worker’s Ethereum address, the taskid, and \(optional\) the Ethereum address of the workers enclave. If the worker doesn’t use an enclave, this field must be filled with `address(0)`.
>
> This Ethereum signature \(authorization\) is sent to the worker through an off-chain channel implemented by the middleware.

1. **Task computation**

> Once the authorization is received and verified, the worker computes the requested tasks. Results from this execution are placed in the `/iexec_out` folder. The following values are then computed:
>
> * _bytes32 digest_: a digest \(sha256\) of the result folder.
> * _bytes32 hash_: the hash of the _digest_, used to produce a consensus
> * _bytes32 seal_: the salted hash of the _digest_, used to prove a worker’s knew the _digest_ value before it is published.
>
> `resultHash == keccak256(abi.encodePacked( taskid, resultDigest))` `resultSeal == keccak256(abi.encodePacked(worker, taskid, resultDigest))`
>
> In computer science, a deterministic algorithm is an algorithm which, given a particular input, will always produce the same output. An application can override the computation of the result digest \(usually the hash of the result archive\) by providing a specific file `/iexec_out/determinism.iexec`. This is necessary to achieve consensus on non-deterministic applications.
>
> If a TEE was used to produce the result, the enclave should produce a `/iexec_out/enclaveSig.iexec` that contains the enclave signature \(of the resultHash and resultSeal\).
>
> Finally, if the requester asked for a callback, the value of this callback must be specified in `/iexec_out/callback.iexec`. Otherwize, the digest will be sent through the callback.

1. **Contribution**

> Once the execution has been performed, the worker pushes its contribution using the `contribute` method. The contribution contains:
>
> * _bytes32 taskid_
> * _bytes32 resultHash_
> * _bytes32 resultSeal_
> * _address enclaveChallenge_
>
>   The address of the enclave \(specified in the scheduler’s authorization\). If no enclave is specified, this parameter should be set to `address(0)`
>
> * _bytes enclaveSign_
>
>   The enclave signature coming from `/iexec_out/enclaveSig.iexec`. This is required if the `enclaveChallenge` is not `address(0)`. Otherwise it should be set to the empty byte string `0x`
>
> * _bytes workerpoolSign_
>
>   The signature computed by the scheduler at step 2.

1. **Consensus**

> During the contribution, the consensus is updated and verified. Contributions are possible until the consensus is reached, at which point the contributions are closed. We then enter a 2h reveal phase.

1. **Reveal**

> During the reveal phase, workers that have contributed to the consensus must call the `reveal` method with the `resultDigest`. This verifies that the `resultHash` and `resultSeal` they provided are valid. Failure to reveal is equivalent to a bad contribution, and results in a loss of stake and reputation.

1. **Finalize**

> Once all contributions have been revealed, or at the end of the reveal period if some \(but not all\) reveals are missing, the scheduler must call the `finalize` method. This finalizes the task, rewards good contribution and punishes bad ones. This must be called before the end of the consensus timer.

### Staking and Payment

Among the objectives of PoCo, we want to ensure a worker that contributes correctly is rewarded and, at the same time, that a requester won’t be changed unless a consensus is achieved. This is achieved by locking the requester’s funds for the duration of the consensus, and unlocking them depending on the outcome.

Staking in use to prevent bad behaviour and encourage good contributions.

Your account, managed by the `Escrow` part of the `IexecClerk`, separates between `balance.stake` \(available, can be withdrawn\) and `balanced.locked` \(unavailable, frozen by a running task\). The `Escrow` exposes the following mechanism:

`lock`: Moves value from the `balance.stake` to `balance.lock`

> * Locks the requester stake for payment
> * Locks the scheduler stake to protect against failed consensus
> * Locks the worker stake when making a contribution

`unlock`: Moves value from the `balance.lock` back to the `balance.stake`

> * Unlock the requester stake when consensus fails
> * Unlock the scheduler stake when consensus is achieved
> * Unlock the worker stake when they contributed to a successful consensus

`seize`: Confiscate value from `balance.lock`

> * Seize the requester stake when the consensus is achieved \(payment\)
> * Seize the scheduler stake when consensus fails \(send to the reward kitty\)
> * Seize the worker stake when a contribution fails \(redistributed to the other workers in the task\)

`reward`: Award value to the `balance.stake`

> * Reward the scheduler when consensus is achieved
> * Reward the worker when they contributed to a successful consensus
> * Reward the app and dataset owner

The requester payment is composed of 3 parts, one for the worker pool, one for the application and one for the dataset. When a consensus is finalized, the payment is seized from the requester and the application and dataset owners are rewarded accordingly. The worker pool part is put inside the `totalReward`. Stake from the losing workers is also added to the `totalReward`. The scheduler takes a fixed portion of the `totalReward` as defined in the worker pool smart contract \(`schedulerRewardRatioPolicy`\).

The remaining reward is then divided between the successful workers proportionally to the impact their contribution made on the consensus. If there is anything left \(division rounding, a few nRLC at most\) the scheduler gets it. The scheduler also gets part of the reward kitty.

#### Parameters

`FINAL_DEADLINE_RATIO = 10`, `CONTRIBUTION_DEADLINE_RATIO = 7`, `REVEAL_DEADLINE_RATIO = 2`

Parameters of the consensus timer. They express the number of reference timers \(category duration\) that are dedicated to each phase. These settings corresponds to a 70%-20%-10% distribution between the contribution phase, the reveal phase and the finalize phase.

> * `FINAL_DEADLINE_RATIO` This describes the total duration of the consensus. At the end of this timer the consensus must be finalized. If it is not, the user can make a claim to get a refund.
> * `CONTRIBUTION_DEADLINE_RATIO` This describes the duration of the contribution period. The consensus can finalize before that, but no contribution will be allowed after the timer to ensure enough time is left for the reveal and finalize steps.
> * `REVEAL_DEADLINE_RATIO` This describes the duration of the reveal period. Whenever a contribution triggers a consensus, a reveal period of this duration is reserved for the workers to reveal their contribution. Note that this period will necessarily start before the end of the contribution phase.

Lets consider a task of category GigaPlus, which reference duration is 1 hour. If the task was submitted at 9:27AM, the contributions must be sent before 4:27PM \(16:27\). Whenever a contribution triggers a consensus, a 2 hours long reveal period will start. Whatever happens, the consensus has to been achieved by 7:27PM \(19:27\).

`WORKERPOOL_STAKE_RATIO = 30`

Percentage of the worker pool price that has to be staked by the scheduler. For example, for a `20 RLC` task, with an additional `1 RLC` for the application and `5 RLC` for the dataset, the worker will have to lock `26 RLC` in total and the scheduler will have to lock \(stake\) `30% * 20 = 6 RLC`.

This stake is lost and transferred to the reward kitty if the consensus is not finalized by the end of the consensus timer.

`KITTY_RATIO = 10`

Percentage of the reward kitty for the scheduler per successful execution. If the reward kitty contains 42 RLC when a finalize is called, then the scheduler will get 4.2 extra RLC and the reward kitty will be left with 37.8 RLC.

`KITTY_MIN = 1 RLC`

Minimum reward on successful execution \(up to the reward kitty value\).

> * If the reward kitty contains 42.0 RLC, the reward is 4.2
> * If the reward kitty contains 5.0 RLC, the reward should be 0.5 but gets raised to 1.0
> * If the reward kitty contains 0.7 RLC, the reward should be 0.07 but gets raised to 0.7 \(the whole kitty\)
>
> `reward = kitty.percentage(KITTY_RATIO).max(KITTY_MIN).min(kitty)`

#### Example

Lets consider a worker pool with the policies `workerStakeRatioPolicy = 35%` and `workerStakeRatioPolicy = 5%`.

* A requester offers `20 RLC` to run a task. The task is free but it uses a dataset that cost `1 RLC`. The requester locks `21 RLC` and the scheduler `30% * 20 = 6 RLC`. The trust objective is `99%` \(`trust = 100`\)
* 3 workers contribute:
  * The first one \(`score = 12 → power = 3`\) contributes `17`. He has to lock `7 RLC` \(35% of the `20 RLC` awarded to the worker pool\).
  * The second worker \(`score = 100 → power = 32`\) contributes `42`. He also locks `7 RLC`.
  * The third worker \(`score = 300 → power = 99`\) contributes `42`. He also locks `7 RLC`.
* After the third contribution, the value `42` has reached a `99.87%` likelihood. Consensus is achieved and the two workers who contributed toward `42` have to reveal.
* After both workers reveal, the scheduler finalizes the task:
  * The requester locked value of `21 RLC` is seized.
  * The dataset owner gets `1 RLC` for the use of its dataset.
  * Stake from the scheduler is unlocked.
  * Stakes from workers 2 and 3 are also unlocked.
  * The first workers stake is seized and he loses a third of its score. The correspond `7 RLC` are added to the `totalReward`
  * We now have `totalReward = 27 RLC`:
    * We save 5% for the scheduler, `workersReward = 95% * 27 = 25.65 RLC`
    * Worker 2 has weight `log2(32) = 5` and worker 3 has a weight `log2(99) = 6`. Total weight is `5+6=11`
    * Worker 2 takes `25.65 * 5/11 = 11.659090909 RLC`
    * Worker 3 takes `25.65 * 6/11 = 13.990909090 RLC`
    * Scheduler takes the remaining `1.350000001 RLC`
  * If the reward kitty is not empty, the scheduler also takes a part of it.

## Replication & Trust

### How to achieve trust ?

The PoCo offers a consensus mechanism that can certify the likelihood of a result to be valid.This consensus relies on the scoring of workers and the replication of a task’s execution to combine the score of the workers that come up with the same result.This consensus is largely based on Sarmenta’s work [\[Sarmenta2002\]](proof-of-contribution.md#requiring-a-trust-level) with specific tuning of the scoring function [\[Trust2018\]](proof-of-contribution.md#contribution-credibility).

### Contribution credibility

Each worker’s contribution has an associated credibility. This credibility derives from the worker’s history score.As described in [\[Trust2018\]](proof-of-contribution.md#requiring-a-trust-level), a worker score is a positive integer that is incremented for each valid and verified contribution.In case of bad contribution, a worker loses one third of it’s score.This credibility can be expressed as a likelihood percentage but also as a weight value that can be used to detect consensus without resorting to floating point arithmetics,more details in [\[Trust2018\]](proof-of-contribution.md#requiring-a-trust-level).

### Requiring a trust level

Based on [\[Sarmenta2002\]](proof-of-contribution.md#requiring-a-trust-level) describing the way to combine worker’s contribution and to evaluate a result’s likelihood, a requester can ask for the level of trust as a input for the PoCo processing, to impose a certain quality of service.The trust level corresponds to a minimum correctness likelihood that a result must achieve to be valid. For example, a trust level of 0 means any contribution would be accepted, regardless of the score of the worker proposing it. On the other hand a trust level of 99.99% means a result will only be accepted if the contribution towards it result shows a correctness probability higher than 99.99%.

The trust level is expressed, on-chain, by a integer `trust` such that `threshold = 1 - 1 / trust`.

| **Trust** | **PoCo enforced confidence threshold** |
| :--- | :--- |
| 0 | 0% |
| 1 | 0% |
| 2 | 50% |
| 100 | 99% |
| 10000 | 99.99% |
| 1000000 | 99.9999% |

### Limitation

This consensus mechanism requires replicable applications.Non deterministic applications, long running jobs such as webservers, do not meet this requirement out of the box.When it is possible, the application developer must provide a deterministic result in the `/iexec_out/determinism.iexec` file.Otherwise, the user can still run its application on the iExec platform but would have to disable the PoCo’s consensus layer.Hardware security as TEE is an option to overpass this limitation.

#### References

| \[Sarmenta2002\] | _\(1, 2\)_ Luis F.G.Sarmenta. Sabotage-tolerance mechanisms for volunteer computing systems. 2002. Future Generation Computer Systems, 18\(4\), 561–572 [Sarmenta2002 PDF](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.67.2962&rep=rep1&type=pdf) |
| :--- | :--- |


| \[Trust2018\] | _\(1, 2, 3\)_ Trust management in the Proof of Contribution protocol. 2018. Technical report. [Trust2018 PDF](https://github.com/iExecBlockchainComputing/iexec-doc/raw/master/techreport/iExec_PoCo_and_trustmanagement_v1.pdf) |
| :--- | :--- |


## Brokering

The blockering will be released in june 2019, with the version 3 of the platform

### Overview

We studied many possible evolutions of the brokering process. The first requirement is to include bid orders to complete the already available ask orders. It soon became clear that the evolution had to go beyond a simple interaction. We considered on-chain and off-chain approach and finally went for a solution where both the pairing and the order book are off-chain. It might sound counterintuitive, but it has many advantages.

If the orders and the pairing are handled off-chain, how can we build an on-chain agreement knowing that all parties have agreed? Is there a threat to the platform security?

This option relies on the use of cryptographic signatures for order authentication. We represent orders using structures containing all the required details. The hash of this structure uniquely identifies the order. The structure by itself is worthless as anyone could write and publish it. However, if we add a valid cryptographic signature \(of the identifying hash\), then the origin of the order can be certified with the same level of security as if it was published on-chain. This role is fulfilled by a smart-contract called the iExecClerk.

An overview of the iExecODB \(Open Decentralized Brokering\) is available in this blog article:

* [PoCo series \#5: Open decentralized brokering on the iExec platform](https://medium.com/iex-ec/poco-series-5-open-decentralized-brokering-on-the-iexec-platform-67b266e330d8)

### Orders structure

As discussed earlier, iExec introduces the offchain signature of orders as a new core element of the iExec Open Decentralized Brokering, the iExec Clerk should match these orders. There are 4 types of orders corresponding to the 4 actors involved: the worker pool, the application, the dataset and the requester. Each order types has to follow a specific structure and is signed using [EIP712](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md) structure signature mechanism.

### Orders description:

#### **AppOrder**

```text
struct AppOrder
{
    address app;
    uint256 appprice;
    uint256 volume;
    uint256 tag;
    address datasetrestrict;
    address workerpoolrestrict;
    address requesterrestrict;
    bytes32 salt;
    bytes   sign;
}
```

* `app` Address of the smartcontract describing the application. Must be registered in the AppRegistry.
* `appprice` Price of a run of the application.
* `volume` Number of run authorized \(by this order\).
* `tag` Special requirements for the application \(see [tag](https://docs.iex.ec/poco.html#tag)\).
* `datasetrestrict` Matching restrictions. Dataset or group of datasets that can be matched. Let null value to disable.
* `workerpoolrestrict` Matching restrictions. Worke rpool or group of worker pools that can be matched. Let null value to disable.
* `requesterrestrict` Matching restrictions. Requester or group of requesters that can be matched. Let null value to disable.
* `salt` A random value to ensure order uniqueness.
* `sign` cryptographic signature of the order, the smart contract is securely linked to application owner.

### **DatasetOrder**

```text
struct DatasetOrder
{
    address dataset;
    uint256 datasetprice;
    uint256 volume;
    uint256 tag;
    address apprestrict;
    address workerpoolrestrict;
    address requesterrestrict;
    bytes32 salt;
    bytes   sign;
}
```

* `dataset` Address of the smartcontract describing the dataset. Must be registered in the DatasetRegistry.
* `datasetprice` Price of a use of the dataset.
* `volume` Number of authorized uses \(by this order\).
* `tag` Special requirements of the dataset \(see [tag](https://docs.iex.ec/poco.html#tag)\).
* `apprestrict` Matching restrictions. App or group of apps that can be matched. Let null value to disable.
* `workerpoolrestrict` Matching restrictions. Workerpool or group of workerpools that can be matched. Let null value to disable.
* `requesterrestrict` Matching restrictions. Requester or group of requesters that can be matched. Let null value to disable.
* `salt` A random value to ensure order uniqueness.
* `sign` cryptographic signature of the order, the smart contract is securely linked to dataset owner.

### **WorkerpoolOrder**

```text
struct WorkerpoolOrder
{
    address workerpool;
    uint256 workerpoolprice;
    uint256 volume;
    uint256 tag;
    uint256 category;
    uint256 trust;
    address apprestrict;
    address datasetrestrict;
    address requesterrestrict;
    bytes32 salt;
    bytes   sign;
}
```

* `workerpool` Address of the smartcontract describing the worker pool. Must be registered in the WorkerpoolRegistry.
* `workerpoolprice` Price of an execution on the worker pool.
* `volume` Number of executions proposed \(by this order\).
* `tag` Special features proposed by the worke rpool \(see [tag](https://docs.iex.ec/poco.html#tag)\).
* `category` Order category.
* `trust` Trust level used to consolidated results.
* `apprestrict` Matching restrictions. App or group of apps that can be matched. Let null value to disable.
* `datasetrestrict` Matching restrictions. Dataset or group of datasets that can be matched. Let null value to disable.
* `requesterrestrict` Matching restrictions. Requester or group of requesters that can be matched. Let null value to disable.
* `salt` A random value to ensure order uniqueness.
* `sign` cryptographic signature of the order, the smart contract is securely linked to worker pool manager.

### **RequesterOrder**

```text
struct RequestOrder
{
    address app;
    uint256 appmaxprice;
    address dataset;
    uint256 datasetmaxprice;
    address workerpool;
    uint256 workerpoolmaxprice;
    address requester;
    uint256 volume;
    uint256 tag;
    uint256 category;
    uint256 trust;
    address beneficiary;
    address callback;
    string  params;
    bytes32 salt;
    bytes   sign;
}
```

* `app` Address of the smartcontract describing the application. Must be registered in the AppRegistry.
* `appmaxprice` Maximum price allowed by the requester for the payment of the application.
* `dataset` Address of the smartcontract describing the dataset. Must be registered in the DatasetRegistry. Null if no dataset is required.
* `datasetmaxprice` Maximum price allowed by the requester for the payment of the dataset \(if any\).
* `workerpool` Matching restrictions. Worker pool or group of worker pools that are allowed to run tasks from this order. Leave null value to disable check.
* `workerpoolmaxprice` Maximum price allowed by the requester for the payment of the execution \(scheduler + workers\).
* `requester` Address of the requester \(paying for the executions\).
* `volume` Number of tasks that are part of this order \(size of the Bag Of Task\).
* `tag` Special features required by the requester \(see [tag](https://docs.iex.ec/poco.html#tag)\).
* `category` tasks category required.
* `trust` Minimum trust level required by the requester.
* `beneficiary` Address of the beneficiary of the computation. Used to require output data encryption.
* `callback` Address to callback with the results \(following EIP1154 interface\). Let empty \(null\) if no callback is required. Learn more about the callback mechanism.
* `params` Parameters of the application \(application specific\).
* `salt` A random value to ensure order uniqueness.
* `sign` cryptographic signature of the order, the smart contract is securely linked to requester.

## Tag

The tag specifies the need or the availability of features that go beyond the specifications of the category. The tag is a requirement when it is part of an app order, a dataset order or a requester order. On the other hand, the tag in the workerpoolorder express the availability of the corresponding features.

In V3, tags are 32 bytes \(256 bits\) long array where each bit corresponds to a feature.

| **Value** | **Description** |
| :--- | :--- |
| `0x0000000000000000000000000000000000000000000000000000000000000001` | TEE \(physical enclave\) |
| `0x0000000000000000000000000000000000000000000000000000000000000002` | — |
| `0x0000000000000000000000000000000000000000000000000000000000000004` | — |
| `0x0000000000000000000000000000000000000000000000000000000000000008` | — |
| `0x0000000000000000000000000000000000000000000000000000000000000010` | — |
| … | … |
| `0x8000000000000000000000000000000000000000000000000000000000000000` | — |

For orders matching, the worker pool order must enable all bits that are enables in any of the app order, dataset order and requester order. Meaning that if the app order tag is `0x12 = 0x10 | 0x02`, the dataset order is `0x81 = 0x80 | 0x01` and the requester order is `0x03 = 0x02 | 0x01`, then the worker pool order must, at least, have a tag `0x93 = 0x80 | 0x10 | 0x02 | 0x01`.

### Matching Conditions

In order to trigger an execution, a deal must be registered by the iExec Clerk. Deals are produced when orders are succesfuly matched by the clerk. A matching requires 3 or 4 orders depending on the requester requirements, the dataset order is optional.

Orders compatibility required:

1. The worker pool’s category and the requester’s category must be equal.

   `require(_requestorder.category == _workerpoolorder.category);`

2. The worker pool’s trust must be greater or equal to the requester’s trust.

   `require(_requestorder.trust == _workerpoolorder.trust);`

3. The app’s, dataset’s and worker pool’s prices must be less or equal to the requester’s appmaxprice, datasetmaxprice and workerpoolmaxprice.

   `require(_requestorder.appmaxprice >= _apporder.appprice);` `require(_requestorder.datasetmaxprice >= _datasetorder.datasetprice);` `require(_requestorder.workerpoolmaxprice >= _workerpoolorder.workerpoolprice);`

4. The worker pool’s tag must enable all the features required by the app’s tag, the dataset’s tag and the worker pool’s tag.

   `require((_apporder.tag | _datasetorder.tag | _requestorder.tag) & ~_workerpoolorder.tag == 0x0);`

5. The app provided by the apporder must match the one required by the requester.

   `require(_requestorder.app == _apporder.app);`

6. The dataset provided by the datasetorder must match the one required by the requester.

   `require(_requestorder.dataset == _datasetorder.dataset);`

7. If the requester specified a worker pool restriction, the worker pool must match this value or be part of the corresponding group.

   `require(checkRestriction(_requestorder.workerpool, _workerpoolorder.workerpool, 0x01));`

8. The application must fit the dataset’s and the worker pool’s application restrictions \(if any\).

   `require(checkRestriction(_datasetorder.apprestrict, _apporder.app, 0x01));` `require(checkRestriction(_workerpoolorder.apprestrict, _apporder.app, 0x01));`

9. The dataset must fit the application’s and the worker pool’s restrictions \(if any\).

   `require(checkRestriction(_apporder.datasetrestrict, _datasetorder.dataset, 0x01));` `require(checkRestriction(_workerpoolorder.datasetrestrict, _datasetorder.dataset, 0x01));`

10. The worker pool must fit the application’s and the dataset’s restrictions \(if any\).

    `require(checkRestriction(_apporder.workerpoolrestrict, _workerpoolorder.workerpool, 0x01));` `require(checkRestriction(_datasetorder.workerpoolrestrict, _workerpoolorder.workerpool, 0x01));`

11. The requester must fit the application’s, the dataset’s and the worker pool’s restrictions \(if any\).

    `require(checkRestriction(_apporder.requesterrestrict, _requestorder.requester, 0x01));` `require(checkRestriction(_datasetorder.requesterrestrict, _requestorder.requester, 0x01));` `require(checkRestriction(_workerpoolorder.requesterrestrict, _requestorder.requester, 0x01));`

12. The application, dataset and worker pool must be registered in the iExecHub.

    `require(iexechub.checkResources(_apporder.app, _datasetorder.dataset, _workerpoolorder.workerpool));`

13. All orders must be signed or presigned.

    `require(verify(App(_apporder.app).m_owner(), _apporder.hash(), _apporder.sign));` `require(verify(Dataset(_datasetorder.dataset).m_owner(), _datasetorder.hash(), _datasetorder.sign));` `require(verify(Workerpool(_workerpoolorder.workerpool).m_owner(), _workerpoolorder.hash(), _workerpoolorder.sign));` `require(verify(_requestorder.requester, _requestorder.hash(), _requestorder.sign));`

14. The deal produced must contain at least one task.
15. Requester and worker pool must be able to stake.

### FAQ : How to write an order ?

**\[Requester\] How do I enable PoCo’s consolidation of results?**

A requester can enable the trust layer of the PoCo by setting the trust value in tue requestorder. As described here, the trust is defined with the required confidence level. If the requester wants à 99.99% confidence level on the results, it must set the `trust` field to `10000`

**\[Requester\] How do I run a non deterministic application despite requiring determinism?**

The PoCo requires an application to be deterministic for the replication layer to provide trust in the result. If an application is not deterministic, consensus cannot be achieved and replication is not necessary.

In order to obtain a result, the requester must prevent replication by asking a `trust` value of `0`. To protect your result, the requester can ask to run in an enclave by setting the `tag` to `0x1`.

**\[Requester\] How do I protect my result using encryption?**

The result of an execution can be valuable to the end user, and the requester might want to protect this result from leaking with encryption.

Anyone can set up an encryption key in a SMS \(Secret Management Service\) of its choice and set up the SMS address in the directory.

Once a user set an encryption key \(see TODO\), any computation result can be encrypted with, you have to set up the beneficiary address in the RequesterOrder.

An application can only perform result encryption inside an enclave. No encryption key will be provided by the SMS to an application that doesn’t run outside an enclave.

\(TODO: potential issue, key leaking to malicious application with the requester attacking a beneficiary\)

**\[Dataset owner\] How do I limit the usage of my dataset to a specific application**

iExec’s Data wallet and Data store is a complete solution to monetize valuable datasets preserving privacy. Before uploading a dataset you should encrypt it using the iExec SDK. Through this process, the encryption key becomes the valuable data that has to be protected.

The encryption key must be stored in a SMS and the address of the corresponding SMS recorded in the directory. The SMS stores this encryption key and will only communicate it to an application running in an enclave.

Before a worker runs this application, the worker must first prove that its access is legitimate by providing the scheduler authorization. The SMS will verify that this authorization’s signature is valid and that the corresponding task is registered onchain. This means that any deal signed in the iExec Clerk will grant access to the dataset’s encryption key.

Therefore in order to restrict the dataset’s usage, the dataset owner should et up restriction before signing a brokering order. This is done through the `apprestrict` field of the datasetorder. The dataset owner can deploy a `SimpleGroup` smart contract, have the `apprestrict` field point to it, and then whitelisting the applications that will have access to the dataset’s encryption key.

**\[Scheduler\] How do I protect myself from non deterministic applications?**

When a scheduler publishes an order, it makes a commitment to achieve consensus on any task that is part of a deal made. While everything is done to ensure an application cannot hurt a worker pool’s machines, not reaching the consensus would cause a loss of stake for the scheduler. The scheduler must therefore take action to prevent this.

Whenever the scheduler proposes to certify a result using the PoCo’s trust layer, it should make sure the application’s developper took the actions required to avec it compatible with the PoCo. On the other hand, any application could be executed with the PoCo’s trust layer disabled.

A scheduler could therefore emit two kinds of workerpoolorder:

* A workerpoolorder offering execution with the PoCo’s trust layer disabled \(`trust = 0`\) and accepting all applications \(`apprestrict = 0`\)
* A workerpoolorder offering secure exection of whitelisted tasks. The application whitelist would use the `GroupInterface` to be verifyed by the iExec Clerk. This group could either be managed by the scheduler or by a certification authority that would check applications determinism.

## Other technical choices

### Callback

Some requester might want an onchain callback with the result of the execution. The callback mechanism is based on [\[EIP1154\]](https://docs.iex.ec/pocosrc/poco-else.html#eip1154). The result is a `bytes` value that is set during the `finalize`. The `IexecHub` implements both side of the [\[EIP1154\]](https://docs.iex.ec/pocosrc/poco-else.html#eip1154).

**Pull**

> Results are identified by their `taskid` and can be pulled through the `resultFor` method.

**Push**

> In order to use the push approach, the requester can use the `callback` field to specify the address of a smart contract that implement the `receiveResult` method specified in [\[EIP1154\]](https://docs.iex.ec/pocosrc/poco-else.html#eip1154). This method will be called during the finalization with at minimum of 100000 nanoRLC gas to proceed [\[\*\]](https://docs.iex.ec/poco.html#id13).
>
> In order to protect the scheduler and the workers, any error raised during this callback will be disregarded and will not prevent the finalization from happening. The same mechanism goes for the callback running out of gas.

### Consensus & Reveal duration

When orders match, IexecClerk records the deal which details the parameters of the task. If a consensus on a result is achieved before the countdown, the task is successful.After the countdown, as no consensus is reached, the execution is failed. The duration of the consensus timer is a balance between the quality of service offered to the requester \(short timer\) and the margin available for the scheduler and the worker to achieve consensus \(and go through the reveal process\).

The maximum duration of a task is governed by the category the task fits in. While the consensus duration can obviously not be shorter than the task runtime a significant margin is required for the scheduler to do its job correctly. Multiple workers are likely to contribute and extra time must be planed for the revealing and finalization steps.

The consensus timer starts when the deal is recorded by the IexecClerk.

In case of failure, the requester can claim the refund.

In the v3-alpha version, this timer lasts for 10 category runtime, for all category.The reveal timer starts whenever a consensus is reached and determines the time frame the workers have to reveal their contributions.This should be long enough for worker to have time to reveal while not being to long so that the requesters waits to long for its result or the consensus fails because the scheduler cannot finalize in time. In the v3-alpha version, this timer lasts for 2 runtime whatever the category runtime. This leaves a gap of at least 1 time the category runtime for the scheduler to finalize the task.

### Reward kitty

When a consensus fails, the requester gets a refund and the scheduler loses its stake. In order to remove an attack vector, the requester does not get any of the seized stake.If this was a feature, anyone could build a flawed application that would not reach consensus to drain money from the scheduler.This would force the scheduler to whitelist all applications thus reducing the usability of the platform.Seized stake from the scheduler goes into a specific account called the _reward kitty_. No one controls this account, nor can withdraw from it.However, the tokens are not burned. Whenever a task is finalized, the scheduler that organized the execution of this task is rewarded by the requester and also gets a small part of the reward kitty.

As described in the protocol parameters section, this reward is `reward = kitty.percentage(KITTY_RATIO).max(KITTY_MIN).min(kitty)`.

### References

| \[EIP1154\] | _\(_[_1_](https://docs.iex.ec/poco.html#id8)_,_ [_2_](https://docs.iex.ec/poco.html#id9)_,_ [_3_](https://docs.iex.ec/poco.html#id10)_\)_ [EIP 1154: Oracle Interface](https://eips.ethereum.org/EIPS/eip-1154) |
| :--- | :--- |


| [\[\*\]](proof-of-contribution.md#other-technical-choices) | value susceptible to change. |
| :--- | :--- |


