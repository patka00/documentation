# Manage your datasetorders

Orders enables setting custom governance for resources.

Here you will learn how to manage your dataset's rules.

## Publish a custom datasetorder

Initialize an datasetorder

```text
iexec order init --dataset
```

Edit the datasetorder part in `iexec.json` to set the conditions to use your dataset

| key | description |
| :--- | :--- |
| dataset | dataset address |
| datasetprice | price to charge the requester for each use of the dataset \(in nRLC\) |
| volume | number of order created, each usage decrease this number |
| tag | restrict usage to specific runtime such "tee" or "gpu" |
| apprestrict | restrict to use the dataset with a specific app \(1\) |
| workerpoolrestrict | restrict to use the dataset on a specific workerpool \(1\) |
| requesterrestrict | restrict the dataset usage to a specific requester \(1\) |

1. the restriction is disabled by default with 0x0000000000000000000000000000000000000000

When you are happy with your datasetorder sign it and publish it

```text
iexec order sign --dataset && iexec order publish --dataset
```

## Remove an order from iExec Marketplace

List the published orders for your dataset.

```text
iexec orderbook dataset <your dataset address>
```

Copy the `orderHash` of the order you want to remove

Unpublish the apporder from the iExec Marketplace

```text
iexec order unpublish --dataset <orderHash>
```

An unpublished order is still valid on the blockchain, to invalidate it use the cancel command.

```text
iexec order cancel --dataset <orderHash>
```

