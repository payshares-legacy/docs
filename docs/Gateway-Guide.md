Payshares Gateway Implementation Guide
====================================

This is an implementation guide for those who want to set up a gateway on the Payshares network. It assumes that the reader already understands how gateways work. To learn more about the concept of gateways, read [Introduction to Payshares Gateways](gateway-introduction.md). This guide will also be code heavy as it contains specific commands necessary to running a gateway.

This guide might not cover everything necessary for running a gateway, but gateway operators should understand everything in this document.

## Cold and hot wallet organization - Bank Wallet Paradigm
We recommend gateways to set up their cold and hot wallets using the bank wallet paradigm. There are other ways of organizing wallets but this one is preferred.

A gateway does not need to use a cold wallet to operate, but doing so reduces the severity of a security breach.

### Cold wallet
A cold wallet is a Payshares account whose secret keys are not on any devices that touch the internet. Transactions are manually initiated by a human and are signed locally on the offline machine. This is possible by using paysharesd's [`sign` api](https://www.payshares.co/api/#api-sign) to create a tx_blob containing the signed transaction. This tx_blob can be transported to a machine connected to the internet via offline methods such as usb or by hand. This makes the cold wallet much harder to steal from.

### Hot wallet
Conversely, a hot wallet is an account that is used on a machine that is connected to the internet. This hot wallet is used to send the gateway's credit to the users of the gateway. A hot wallet has a limited amount of funds to limit the amount lost in the event of a security breach.

### Trust lines
There are only two instances where trust needs to be extended:
- Gateway hot wallet(s) extends trust to the cold wallet limited to a small amount.
- Users extend trust to the gateway cold wallet so that they can receive funds.

### Flow of credits
First, the gateway cold wallet sends funds to the hot wallet so that the gateway can quickly send the money.

When a user enters the Payshares network (by depositing cash into the gateway), the gateway hot wallet sends credit to the user. For example, if user deposits $20 USD in cash into the gateway, the gateway hot wallet sends $20 USD credit to the user.

When a user wants to redeem their credit and exit the Payshares network, the user sends the funds to the cold wallet.

```
+------+      +-----+      +------+      
| cold | +--> | hot | +--> | user |
+------+      +-----+      +------+
   ^                           +
   |                           |
   +---------------------------+
```

## Payshares reference client user flow
Important for gateway operators to understand how end users and consumers will interact with gateways.

In the Payshares reference client ([launch.payshares.co](https://launch.payshares.co)), one must add a gateway before being able to receive credit issued by that gateway. The user specifies the domain of the gateway and the client scans for the [payshares.txt](Payshares.txt.md) file of that domain to find the currencies that the gateway supports.

## Paysharesd setup
[paysharesd](https://github.com/payshares/paysharesd) is the daemon that powers the backend of the Payshares network. Gateways should run their own `paysharesd` instances so that they do not have to rely on other `paysharesd` instances being up and running. Another benefit is that if gateway software relies on `paysharesd` to do key signing, it must be run locally so tha secret keys are not sent to a third party.

When running a `paysharesd` instance, make sure that RPC and WebSocket access to the gateway `paysharesd` is not accessible from the outside world. If it is accessible to the outside world, the `paysharesd` instance can be vulnerable to a denial-of-service attack.

## Account setup
- Create both the cold and hot wallet accounts. The keys for the cold wallet should be created on an computer that does not have access to the internet after generating the keys. For example, the keys can be safely generated on a live boot of ubuntu in which data is wiped after it is run.
- Fund the cold wallet and hot wallets so that they have well above the minimum deposit so that it can pay the transaction fees necessary to operate.
- The hot wallet account has to extend trust to the cold wallet account using the `paysharesd` api call [`submit TrustSet`](https://www.payshares.co/api/#api-trustset). The amount of trust should be high enough to be able to handle many transactions before requiring refunding from the cold wallet while also low enough that if compromised, places the gateway's future at great risk.
- The cold wallet should require [destination tags](Destination-Tags.md) for all incoming transactions. It is useful when receiving credits from users.

## payshares.txt setup
`payshares.txt` is a file that contains data about Payshares-related services a domain provides in a machine readable format. Gateways use this file to tell clients what currencies they support and  how to use the gateway.

Read more about [payshares.txt](Payshares.txt.md) and how to set it up correctly.

### [currencies]
A gateway defines the currencies that it supports and the issuing accounts (cold wallets as specified above) of the currencies. Multiple currencies are allowed to use the same accounts but do not necessarily have to.

As noted in the client user flow explanation above, a client will look at the payshares.txt file to find out which currencies a gateway supports so that it can extend trust to the issuing account.

Currencies are denoted with their 3 letter symbol and the corresponding issuing account address.
```
[currencies]
USD gazAaB4WtLjLz5RMqvL1LB4KvxWFXEe5tX
BTC gazAaB4WtLjLz5RMqvL1LB4KvxWFXEe5tX
CHF gUuPMNHLxkd98o3xe8CKjYGR4vyuYPZuYE
```

## Sending credit (for users to enter the Payshares network)
Gateways should use the hot wallet to send credit to users. It is important that all actions be logged and transactions sent and managed robustly.

## Sending transactions robustly
Infrastructure is never perfectly reliable. Software, hardware and networks can all encounter hiccups. There are many ways in which a transaction can go wrong--it might never be sent or might even be sent multiple times. Mistakes are costly, especially when dealing with money. By using transaction robustness techniques, one can handle these transactions safely.

Only one machine should be using the hot wallet if using techniques described on the transaction robustness page.

Read more about [transaction robustness](Transaction-Robustness.md). It is critical that gateways use techniques like this.

## Receiving credit (for users redeeming their credit from the gateway)
When a user wants redeem some credit they have from a gateway, the user sends the credit to the gateway cold wallet.

The gateway should require [Destination Tags](Destination-Tags.md) in payments to the cold wallet. This tag is a number assigned to a user so that the gateway can receive funds and know whom the deposit belongs to. Before a user sends credit to the gateway, the gateway tells them what destination tag to use.

The gateway software can then [subscribe](https://www.payshares.co/api/#api-subscribe) to incoming transactions and listen for incoming transactions.

The gateway software should also have functionality to [scan through previous transactions](Gateway-API.md#previous-transactions) and search through previous transactions and find ones that have yet to be processed. This is important so that the gateway can automatically resume operation after downtime.

## Important operational concerns

### Hot wallet depletion alerts
The gateway software should send automated alerts to the gateway operator and notify when the hot wallet is low on funds and needs to be replenished from the cold wallet.

### Unsolicited receiving of funds / non-existent destination tag
Users might send payments to the cold wallet with a destination tag that isn't associated with any account. Users might also send funds to the hot wallet for no reason. It is up to the gateway to decide how to deal with these incidents.

Most gateways will just send the funds back. However, this makes the gateway hot wallet vulnerable to a denial of service attack.

### Hot wallet denial of service
Users (intentional or not) could cause a denial of service on the hot wallet. This is possible by depleting the hot wallet. Here are several ways this could happen:
- User sends funds (credit or XPS) to the cold wallet with a non-existent destination tag and the gateway uses the hot wallet to send them back.
- User sends funds (credit or XPS) to the hot wallet and the gateway sends the funds back.
- User deposits credit to the gateway cold wallet then withdraws back to their Payshares account (from the gateway hot wallet); user repeats and drains the hot wallet.

As a result, the gateway would be unable to send from their hot wallet until a human operator replenishes the hot wallet manually.

## Checklist
- Stuff

## Extra features

### SetRegularKey security enhancement
For extra security, gateway wallets can use `SetRegularKey` to create a new temporary key that can sign transactions from a secondary key. If this key is compromised, the gateway can use the master key to revoke this secondary key and create a new one.

### Authorized accounts
Payshares provides the ability for gateways to restrict access for who is allowed to send and receive credits issued by a gateway.
