# Approder mangement

## Publish a custom apporder

Initialize an apporder
```
iexec order init --app
```

Edit the apporder part in iexec.json to set the conditions to use your app

| key | description |
| :--- | :--- |
| app | app address |
| appprice | price to charge the requester for each execution of the app (in nRLC) |
| volume | number of order created, each usage decrease this number |
| tag | not use |
| datasetrestrict: | restrict to use the app with a specific dataset \(1\) |
| workerpoolrestrict | restrict to run the app on a specific workerpool \(1\) |
| requesterrestrict: | restrict the app usage to a specific requester \(1\) |

1. the restriction is disabled by default with 0x0000000000000000000000000000000000000000

When you are happy with your apporder sign it and publish it
```
iexec order sign --app && iexec order publish --app
```

## Remove an order from iExec Marketplace

List the published orders for your app.
```
iexec orderbook app <your app address>
```

Copy the `orderHash` of the order you want to remove

Unpublish the apporder from the iExec Marketplace
```
iexec order unpublish --app <orderHash>
``` 
