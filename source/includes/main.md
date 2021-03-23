---
title: API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - json

toc_footers:
  - <a href='#'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/slatedocs/slate'>Documentation Powered by Slate</a>

search: true

code_clipboard: true
---

# DerivaDEX API

DerivaDEX is a decentralized derivatives exchange that combines the performance of centralized exchanges with the security of decentralized exchanges.

DerivaDEX currently offers a public WebSocket API for traders and developers. You must first make a deposit to the DerivaDEX ethereum contracts. The API will then enable you to open and manage your positions via commands, and subscribe to market data via subscriptions. 

Find us online [Discord](https://discord.gg/a54BWuG) | [Telegram](https://t.me/DerivaDEX) | [Medium](https://medium.com/derivadex)

# Getting Started

DerivaDEX is a Decentralized Exchange, which means trading is non-custodial. Users are responsible for their own funds, which are deposited to the DerivaDEX smart contract on Ethereum or trading. 

You will connect to DerivaDEX and authenticate yourself via your Ethereum account. You can deposit funds via an Ethereum client, and then authenticate yourself with the DerivaDEX API. Generally, you will want to follow these steps:
1. Deposit funds via Ethereum
2. Connect to the websocket API
3. Submit and cancel orders via `commands`
4. Subscribe to data feeds via `subscriptions`. 

Additionally, you should familiarize yourself with EIP 712 signatures, which is how `command` data is signed and submitted. 

## Making a deposit

To deposit funds on DerivaDEX, first ensure that you have created an Ethereum account. TBD -- based on Adi's work.

## Connecting to the Websocket API

Addresses used to connect to the websocket API must _already_ have funds deposited. If you haven't, [do that first](Making a deposit)

The steps to connect are:

-   Generate a numeric value (nonce) that is greater than the last value used for this address
    -   [Unix time](https://en.wikipedia.org/wiki/Unix_time) is commonly used, but users may opt to maintain their own sequence counter. Requests with a nonce that are less than or equal to the previous value will be rejected.
-   Concatenate and hash the nonce with "Derivadex API":
    `keccak256(abi.encode("DerivaDEX API", nonce))`
-   Generate a signature of the hash using eth_sign or equivalent signer.
-   Connect to the websocket with the url:
    `wss://api.derivadex.com?nonce=[nonce]&signature=[signature]`

For more information, see the [code samples](samples.md).

# Commands
The websocket API offers commands for placing and canceling orders, as well as withdrawals. Requests must include an [EIP712 signature](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md). Examples are included in the [sample code](samples.md).

## Place Order

> Request format:

```json
{
  "request": {
    "order": {
      "makerAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
      "symbol": "ETHPERP",
      "strategy": "main",
      "side": "Bid",
      "orderType": "Market",
      "requestId": "0x0e37c859bd811c0d877940c528df79baad972282e82ef3dd37e6fbe21b988653",
      "amount": "123.45",
      "price": "234.56",
      "stopPrice": "0",
      "signature": "0x89fb49d2d125adfef56328ee7367d23d1642d2fb6cfdf8843fa94ae7eac9ab23"
    }
  }
}
```

makerAddress | The trader's address
symbol | Currently always 'ETHPERP'. New symbols coming soon! 
strategy | Always `main`. Support for multiple strategies coming soon!
side | Whether the order is a Bid or an Ask (case sensitive)
orderType | `Market`. Other order types coming soon!
requestID | An incrementing numeric identifier for this request
amount | The amount requested
price | The Bid or Ask price
stopPrice | Currently, always 0 as stops are not implemented.
signature | EIP712 signature

EIP712 struct OrderParams

  address | makerAddress;  
  bytes32 | symbol;  
  bytes32 | strategy;  
  uint256 | side;  
  uint256 | orderType;  
  bytes32 | clientId;  
  uint256 | assetAmount;  
  uint256 | price;  
  uint256 | stopPrice;  


## Cancel Order

>Request format:

```json
{
  "request": {
    "cancelOrder": {
      "symbol": "ETHPERP",
      "orderHash": "0xa2e47f2d462c4368fdae25028447c78118ba349141d0ba945c6f100dfd149029",
      "orderRequestId": "0x0e37c859bd811c0d877940c528df79baad972282e82ef3dd37e6fbe21b988653",
      "requestId": "0x0000000000000000000000000000000000000000000000000000000000000653",
      "signature": "0x89fb49d2d125adfef56328ee7367d23d1642d2fb6cfdf8843fa94ae7eac9ab23"
    }
  }
}
```
EIP712 struct CancelParams

type | 
---------------------------- 
bytes32 | orderHash;  
bytes32 | orderRequestid;  
bytes32 | requestId;  


type | description
-----------------------------
symbol | Currently always 'ETHPERP'. New symbols coming soon! 
orderHash| Where does this come from?
orderRequestId | How is this different from requestID?
requestID | An incrementing numeric identifier for this request.
signature | EIP712 signature

## Withdraw

>Request format:

```json
{
  "request": {
    "withdraw": {
      "traderAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
      "strategyId": "main",
      "currency": "0x41082c820342539de44c1b404fead3b4b39e15d6",
      "amount": "440.32",
      "requestId": "0x0e37c859bd811c0d877940c528df79baad972282e82ef3dd37e6fbe21b988653",
      "signature": "0x89fb49d2d125adfef56328ee7367d23d1642d2fb6cfdf8843fa94ae7eac9ab23"
    }
  }
}
```
field | description
------------------------
traderAddress | Address of the account for the withdrawal
strategyId | 
currency | contract address for currency (USDC)
amount | amount to be withdrawin
requestId | An incrementing numeric identifier for this request.
signature | EIP712 signature

EIP712 Struct `WithdrawParams`|
------------------------------
address | traderAddress
bytes32 | strategyId
address | currency
uint128 | amount
bytes32 | requestId


## Receipts

```json
{
  "receipt": {
    "type": "Received",
    // The requestId supplied in the initial request - can be used to correlated requests with receipts
    "requestId": "0x0e37c859bd811c0d877940c528df79baad972282e82ef3dd37e6fbe21b988653",
    // A ticket number which guarentees fair sequencing
    "requestIndex": "0x5",
    // An Operator's signature which proves secure handling of the request.
    "enclaveSignature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
  }
}
```
Each command returns a receipt, which confirms that an Operator has received the request sequenced it for processing. The receipt `type` will be either "Received" or "Error". The `requestId` is the requestId from the original command. The `requestIndex` is the Operator's sequence id.

receipt |
---------------------
type | Received
requestId | The requestId supplied in the initial request - can be used to correlated requests with receipts
requestIndex | A ticket number which guarentees fair sequencing (how?)
enclaveSignature | An Operator's signature which proves secure handling of the request

DerivaDEX operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the guaranteed associated with Intel SGX TEEs. 

## Receipt error

>Receipt error message

```json
{
  "receipt": {
    "type": "Error",
    "msg": "Nonces cannot go backwards",
  }
}
```

A error will return a receipt with type of `Error` and an error message.


# Subscriptions

## Subscribe to account feed

> Subscribe to account feed

```json
{
  "subscribeAccount": {
    "trader": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
    "strategies": ["main"],
    "events": ["StrategyUpdate", "PositionUpdate"]
  }
}
```
Account feeds are subscriptions to Strategy and Position events for a trader address. To subscribe to an account feed, send a websocket message following the code sample at right:

subscription data |
----------------------------
trader | The address you want to subscribe to (i.e., your own Ethereum address)
strategies | The strategy 'main' must be included.
events | Pass in one or both of StrategyUpdate and PositionUpdate to subscribe to an account feed

## Account feed subscription confirmation

> Confirmation receipt

```json
{
  "receipt": {
    "type": "Subscribed",
    "msg": "Subscribed to PositionUpdate for 0x0e37c859bd811c0d877940c528df79baad972282e82ef3dd37e6fbe21b988653"
  }
}
```
A receipt type 'Subscribed' confirms the subscription was successful.


## Account feed snapshot

```json
{
  "type": "Partial",
  "topic": "PositionUpdate",
  "data": [{
    "trader": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
    "strategyId": "main",
    "symbol": "ETHPERP",
    "side": "Long",
    "balance": "423.44",
    "avgPrice": "605",
    "lastEpoch": "325"
  }]
}

```
The current snapshot is returned immediately after the subscription receipt with a type of `Partial`.
`Partial` indicates that this is an initial snapshop following a subscription.


## Account feed update

```json
{
  "type": "Update",
  "topic": "PositionUpdate",
  "data": [{
    "trader": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
    "strategyId": "main",
    "symbol": "ETHPERP",
    "side": "Long",
    "balance": "423.44",
    "avgPrice": "605",
    "lastEpoch": "325"
  }]
}
```
Streaming updates follow, with the type `Update`. `Update` indicates that this is an incremental update.



## Market events subscription

>Subscribe to market

```json
{
  "subscribeMarket": {
    "symbols": ["ETHPERP"],
    "events": ["OrderBook", "Markprice"]
  }
}
```
Subscribe to Orderbook and Markprice events by sending a websocket message following the code snippet format at right.

## Market events subscription confirmation

>Market subscription confirmation

```json
{
   "receipt": {
      "type": "Subscribed",
      "msg": "Subscribed to OrderBook for ETHPERP"
   }
}
```
A successful subscription returns a confirmation message of type `Subscribed`.

## Market events partial orderbook snapshot

```json
{
   "type": "Partial",
   "topic": "OrderBook",
   "data": [{
     "bids": [ [ "460", "1.123455666" ] ],
     "asks": [ [ "470", "0.98" ] ],
     "timestamp": 1616112500667,
     "nonce": "0x816c8dd7b069a00a0f0a83cbd630b3b4069431a6eb141ddf3108a20d94c3f5f5",
     "aggregationType": "0.0001"
   }]
}
```
The current snapshot is returned immediately after the subscription receipt with a type of `Partial`.

## Market events streaming updates

>Streaming updates

```json
{
   "type": "Update",
   "topic": "OrderBook",
   "data": [{
     "bids": [ [ "580.56", "1.5" ] ]
     "asks": [],
     "timestamp": 1616087958750,
     "nonce": "0x1e8038f562948c0fc747cdab1ee8ed69c13f5ca34e194ef147c8d140ba040f14",
     "aggregationType": "0.0001"
   }]
}
```
Streaming updates follow, with the type `Update`.


## Mark price subscription confirmation

>Subscription confirmation for Mark Price

```json
{
   "receipt": {
     "type": "Subscribed",
     "msg": "Subscribed to Markprice for ETHPERP"
   }
}
```
For Mark Price, the subscription confirmation is of type `subscribed`.

## Partial mark price snapshot

>"type": "Partial mark price snapshot"

```json
{
   "topic": "Markprice",
   "data": [{
     "price": "472.96848799410645",
     "symbol": "ETHPERP",
     "createdAt": "2021-03-19T00:16:57.045Z",
     "updatedAt": "2021-03-19T00:16:57.045Z"
   }]
}
```
A mark price subscription is followed by an initial `Partial` snapshot. 

## Mark price streaming updates

> Mark price streaming updates

```json
{
    "type": "Update",
    "topic": "Markprice",
    "data": [{
        "createdAt": "2021-03-04T22:51:37.113985+00:00",
        "updatedAt": "2021-03-04T22:51:37.113985+00:00",
        "price": "458.40712784392935",
        "symbol": "ETHPERP"
    }]
}
```
Streaming updates follow the snapshot, with the type `Update`.


# Validation

TBD

# Rate Limits

The websocket API will (eventually) use tiers to determine rate limits for each ethereum account.
Rate limits restrict the number of "Commands" an account can place per minute.

Subscriptions are not rate limited.

