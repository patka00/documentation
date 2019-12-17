---
description: >-
  A workerpool manager is an essential actor in the iExec network. It will be in
  charge of proposing available computing resources (workers) to task
  requesters.
---

# Manage a Pool of Workers

The iexec-core software will take care of handling tasks comming from blockchain events and will distribute it to alive workers.

## Prerequisites

Workers are deployed with **docker** containers.  
Follow instructions on [the docker website](https://docs.docker.com/install/) for installation.

## Docker compose file

> Since all the services used by iExec run in docker, we will use docker-compose to start the scheduler and its related service. You can follow the instructions on the [the docker compose website](https://docs.docker.com/compose/) to install it.

In the docker compose file, you will find the following components:

### **Database service**

A database is required for running the scheduler.

```text
  image: mongo:4-xenial
  container_name: mongo
  networks:
    - iexec-net
  restart: unless-stopped

mongo_ui:
  image: mongo-express:0.49
  container_name: mongo_ui
  environment:
    - ME_CONFIG_BASICAUTH_USERNAME=mongoUiAdmin
    - ME_CONFIG_BASICAUTH_PASSWORD=mongoUiPassw0rd
  ports:
    - 8081:8081
  networks:
    - iexec-net
  depends_on:
    - mongo
  restart: unless-stopped
```

### **IPFS service**

An IPFS node is required for running the scheduler.

```text
ipfs:
  image: jbenet/go-ipfs:latest
  container_name: ipfs-node
  ports:
    - 4001:4001
  volumes:
    - ipfs-docker-staging:/export
    - ipfs-docker-data:/data/ipfs
  networks:
    - iexec-net
  restart: unless-stopped
```

### **Scheduler service**

The main component is the scheduler service. In the docker compose file, it is defined as follow:

```text
core:
  image: iexechub/iexec-core:3.0.0
  container_name: core
  environment:
    - IEXEC_CORE_WALLET_PATH=/iexec-wallet/wallet.json
    - IEXEC_CORE_WALLET_PASSWORD=myWalletPassw0rd
    - IEXEC_PRIVATE_CHAIN_ADDRESS=https://my-jsonrpc-enabled-ethereum-node.com
    - IEXEC_PUBLIC_CHAIN_ADDRESS=https://my-default-public-ethereum-node-for-workers.com
    - IEXEC_HUB_ADDRESS=0xiexechubaddress
    - POOL_ADDRESS=0xmyworkerpool
    - IEXEC_START_BLOCK_NUMBER=0
    - REVEAL_TIMEOUT_PERIOD=120000
    - IEXEC_ASK_REPLICATE_PERIOD=30000
    - IEXEC_RESULT_REPOSITORY_HOST=my-workerpool.com
    - IEXEC_RESULT_REPOSITORY_PORT=18090
    - IEXEC_CHAIN_ID=42
    - MONGO_HOST=mongo
    - IEXEC_IPFS_HOST=ipfs
    - IEXEC_IPFS_PORT=5001
    - IEXEC_CORE_PORT=18090
    - IEXEC_SMS_HOST=http://my-sms.com
  volumes:
    - ./wallet.json:/iexec-wallet/wallet.json
  ports:
    - 18090:18090
  networks:
    - iexec-net
  depends_on:
    - mongo
    - ipfs
  restart: unless-stopped
```

Like the database, the different variables used here are defined in the file .env and **should be modified**.

## Start a scheduler

To start a scheduler, it is pretty straightforward since the scheduler can be started like any docker compose service, so it can be started using:

```text
docker-compose up -d
```

Please note that you need to make sure the scheduler has finished its start before starting any worker that will connect to the pool.

## Stop a scheduler

In a similar fashion, the scheduler can be stopped with the following command:

```text
docker-compose down
```

Please note that once the scheduler is turned off, the workers will not work anymore.

