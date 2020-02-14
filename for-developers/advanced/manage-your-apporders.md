# Manage your apporders

Orders enables setting custom governance for resources.

Here you will learn how to manage your application's rules.

## Publish a custom apporder

Initialize an apporder

```text
iexec order init --app
```

Edit the apporder part in `iexec.json` to set the conditions to use your app

| key | description |
| :--- | :--- |
| app | app address |
| appprice | price to charge the requester for each execution of the app \(in nRLC\) |
| volume | number of order created, each usage decrease this number |
| tag | restrict usage to specific runtime such "tee" or "gpu" |
| datasetrestrict | restrict to use the app with a specific dataset \(1\) |
| workerpoolrestrict | restrict to run the app on a specific workerpool \(1\) |
| requesterrestrict | restrict the app usage to a specific requester \(1\) |

1. the restriction is disabled by default with 0x0000000000000000000000000000000000000000

When you are happy with your apporder sign it and publish it

```text
iexec order sign --app && iexec order publish --app
```

## Remove an order from iExec Marketplace

List the published orders for your app.

```text
iexec orderbook app <your app address>
```

Copy the `orderHash` of the order you want to remove

Unpublish the apporder from the iExec Marketplace

```text
iexec order unpublish --app <orderHash>
```

An unpublished order is still valid on the blockchain, to invalidate it use the cancel command.

```text
iexec order cancel --app <orderHash>
```
