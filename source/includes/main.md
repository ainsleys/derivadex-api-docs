
# DerivaDEX API

DerivaDEX is a decentralized derivatives exchange that combines the performance of centralized exchanges with the security of decentralized exchanges.

DerivaDEX currently offers a public WebSocket API for traders and developers. You must first make a deposit to the DerivaDEX ethereum contracts. The API will then enable you to open and manage your positions via commands, and subscribe to market data via subscriptions. 

Find us online [Discord](https://discord.gg/a54BWuG) | [Telegram](https://t.me/DerivaDEX) | [Medium](https://medium.com/derivadex)

# Getting Started

DerivaDEX is a decentralized exchange. As such, trading is non-custodial. Users are responsible for their own funds, which are deposited to the DerivaDEX smart contract on Ethereum for trading. 

To begin interacting with the DerivaDEX ecosystem programmatically, you generally will want to follow these steps:
1. Deposit funds via Ethereum via an Ethereum client
2. Authenticate and connect to the websocket API
3. Submit and cancel orders via `commands`
4. Subscribe to market and account data feeds via `subscriptions` 

Additionally, you should familiarize yourself with [EIP 712 signatures](https://eips.ethereum.org/EIPS/eip-712), which is how you will sign `command` data prior to submission. 

## Making a deposit

To deposit funds on DerivaDEX, first ensure that you have created an Ethereum account. The deposit interaction is between a user and the DerivaDEX smart contract. To be more explicit, you will not be utilizing the WebSocket API to facilitate a deposit. The DerivaDEX Solidity smart contracts adhere to the [Diamond Standard](https://medium.com/derivadex/the-diamond-standard-a-new-paradigm-for-upgradeability-569121a08954). The `deposit` smart contract function you will need to use is located in the `Trader` facet, at the address of the main `DerivaDEX` proxy contract.

```text
// solidity
function deposit(
    address _collateralAddress,
    bytes32 _strategyId,
    uint128 _amount
) external;
```

field | description 
------ | -----------
_collateralAddress | ERC-20 token address deposited as collateral
_strategyId | Strategy ID (encoded as a 32-byte value) being funded 
_amount | Amount deposited (be sure to use the grains format specific to the collateral token being used (e.g. if you wanted to deposit 1 USDC, you would enter 1000000 since the USDC token contract has 6 decimal places)

## Connecting to the Websocket API

Addresses used to connect to the websocket API must _already_ have funds deposited. If you haven't, [do that first](#Making a deposit).

The steps to connect are:

1. Generate a numeric value (`nonce`) that is greater than the last value used for this address. Typically, [Unix time](https://en.wikipedia.org/wiki/Unix_time) (either milliseconds or nanoseconds) is used, but users may opt to maintain their own sequence counter. Requests with a nonce that are less than or equal to the previous value will be rejected.
2. `Keccak-256` hash the following ABI-encoded values with the specified types(using `Keccak-256`):
    - "DerivaDEX API" (`bytes32`) 
    - `nonce` (`uint256`)
3. Generate a `signature` of the hash using eth_sign or equivalent signer.
4. Connect to the websocket with the url: `wss://api.derivadex.com?nonce=[nonce]&signature=[signature]`

For more information, see the [code samples](samples.md).

# Commands
The websocket API offers commands for placing and canceling orders, as well as withdrawals. Since commands modify system state, these requests must include an [EIP712 signature](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md). Examples are included in the [sample code](samples.md).

## Place order

Traders can place new orders by specifying specific attributes in the `Order` command's request payload. These requests are subject to a set of validations.

> Request format:

```json
{
  "request": {
    "t": "Order",
    "c": {
      "makerAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
      "symbol": "ETHPERP",
      "strategy": "main",
      "side": "Bid",
      "orderType": "Market",
      "requestId": "0x0000000000000000000000000000000000000000000000000000000000000001",
      "amount": "123.45",
      "price": "234.56",
      "stopPrice": "0",
      "signature": "0x89fb49d2d125adfef56328ee7367d23d1642d2fb6cfdf8843fa94ae7eac9ab23"
    }
  }
}
```

type | field | description 
------ | ---- | -------
address | makerAddress | Trader's Ethereum address (same as the one that facilitated the deposit)
string  | symbol | Name of the market to trade. Currently, this is limited to 'ETHPERP', but new symbols are coming soon! 
bytes32 | strategy | Name of the cross-margined strategy this trade belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon!
uint256 | side | Side of trade, either `Bid` (buy/long) or an `Ask` (sell/short)
uint256 | orderType | Order type, either `Limit` or `Market`. Other order types coming soon!
bytes32 | requestID | An incrementing numeric identifier for this request
uint256 | amount | The order amount/size requested
uint256 | price | The order price
uint256 | stopPrice | Currently, always 0 as stops are not implemented.
bytes | signature | EIP-712 signature


## Cancel order

Traders can cancel existing orders by specifying specific attributes in the `CancelOrder` command's request payload.

>Request format:

```json
{
  "request": {
    "t": "CancelOrder",
    "c": {
      "symbol": "ETHPERP",
      "orderHash": "0xa2e47f2d462c4368fdae25028447c78118ba349141d0ba945c6f100dfd149029",
      "requestId": "0x0000000000000000000000000000000000000000000000000000000000000002",
      "signature": "0x89fb49d2d125adfef56328ee7367d23d1642d2fb6cfdf8843fa94ae7eac9ab23"
    }
  }
}
```

type | field | description
-----|---- | -----------------------
string | symbol | Currently always 'ETHPERP'. New symbols coming soon! 
bytes32 | orderHash| The hash of the order being canceled
bytes32 | requestID | An incrementing numeric identifier for this request
bytes | signature | EIP-712 signature

## Withdraw

Traders can signal withdrawal intents to the Operators by specifying specific attributes in the `Withdraw` command's request payload. While you won't be able to trade with this collateral you are attempting to withdraw, you will only be able to formally initiate a smart contract withdrawal/token transfer once the epoch in which you signal your withdrawal desire has concluded. 

>Request format:

```json
{
  "request": {
    "t": "Withdraw",
    "c": {
      "traderAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
      "strategyId": "main",
      "currency": "0x41082c820342539de44c1b404fead3b4b39e15d6",
      "amount": "440.32",
      "requestId": "0x0000000000000000000000000000000000000000000000000000000000000003",
      "signature": "0x89fb49d2d125adfef56328ee7367d23d1642d2fb6cfdf8843fa94ae7eac9ab23"
    }
  }
}
```

type | field | description
---- | --- | -----------------
address | traderAddress | Trader's Ethereum address (same as the one that facilitated the deposit)
bytes32 | strategyId | Name of the cross-margined strategy this trade belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon!
address | currency | ERC-20 token address being withdrawn
uint128 | amount | Amount withdrawn (be sure to use the grains format specific to the collateral token being used (e.g. if you wanted to withdraw 1 USDC, you would enter 1000000 since the USDC token contract has 6 decimal places)
bytes32 | requestId | An incrementing numeric identifier for this request.
bytes | signature | EIP-712 signature


## Receipts

Each command returns a receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Received` or `Error`.

### Received (successful) receipts

A successful command returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the guaranteed associated with Intel SGX TEEs.

```json
{
  "receipt": {
    "t": "Received",
    "c": {
      "requestId": "0x0000000000000000000000000000000000000000000000000000000000000003",
      "requestIndex": "0x5",
      "enclaveSignature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
    }
  }
}
```

type | field | description
------ | ---- | -----------
bytes32 | requestId | The requestId supplied in the initial request - can be used to correlate requests with receipts
?? | requestIndex | A ticket number which guarantees fair sequencing
bytes | enclaveSignature | An Operator's signature which proves secure handling of the request 

### Error (failed) receipts

An erroneous command returns an `Error` receipt from the Operator.

```json
{
  "receipt": {
    "t": "Error",
    "c": {
      "msg": "Nonces cannot go backwards"
    }
  }
}
```

type | field | description
------ | ---- | -----------
string | msg | Error message


# Subscriptions

Traders can subscribe to two different kinds of feeds corresponding to a specific trader's account or market data with `SubscribeAccount` or `SubscribeMarket` requests, respectively.

## Account

Traders can subscribe to data feeds corresponding to events for a particular trader's Ethereum address. The possible account events traders can subscribe to are `StrategyUpdate`, `OrdersUpdate`, and `PositionUpdate`. These will be discussed below individually. 

## Strategy update (account)

Traders can subscribe to updates to their strategy with the `StrategyUpdate` event. A strategy is a cross-margined account in which a trader can deposit/withdraw collateral and make trades. Strategies are siloed off from one another. Any updates to the free or frozen collaterals you have (whether it's from deposits, withdrawals, or realized PNL) will be registered in this feed.

Upon subscription, you will receive a `Partial` back, containing a snapshot of your strategy at that moment in time. From then on, you will receive streaming `Update` messages as appropriate.

> Request
```json
{
  "request": {
    "t": "SubscribeAccount",
    "c": {
      "trader": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
      "strategies": ["main"],
      "events": ["StrategyUpdate"]
    }
  }
}
```

> Partial response
```json
{
	"t": "StrategyUpdate",
	"e": "Partial",
	"c": [{
		"trader": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
		"strategy": "main",
		"maxLeverage": "20",
		"freeCollateral": "1000",
		"frozenCollateral": "0",
		"createdAt": "2021-03-23T20:03:45.850Z"
	}]
}
```

> Update response
```json
{
	"t": "StrategyUpdate",
	"e": "Update",
	"c": [{
		"trader": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
		"strategy": "main",
		"maxLeverage": "20",
		"freeCollateral": "2000",
		"frozenCollateral": "0",
		"frozen": false,
		"createdAt": "2021-03-23T20:03:45.850Z"
	}]
}
```

**Request parameters**

type | field | description
-----|----- | ---------------------
address  | trader | Trader's Ethereum address (same as the one that facilitated the deposit)
string[] | strategies | Strategies being subscribed to. Currently, the only supported strategy is `main`, but support for multiple strategies is coming soon!
string[] | events | Events being subscribed to. This can be one or more of `OrdersUpdate`, `PositionUpdate`, and `StrategyUpdate`

**Response parameters** - an array of updates per strategy is emitted, with each update defined as follows:

type | field | description
-----|----- | ---------------------
address  | trader | Trader's Ethereum address (same as the one requested)
string   | strategy | Strategy being subscribed to. Currently, the only supported strategy is `main`, but support for multiple strategies is coming soon!
int   | maxLeverage | Maximum leverage for strategy, which impacts the maintenance margin ratio associated to any given trader
Decimal  | freeCollateral | Collateral available for trading (not to be confused with the `Free Collateral` displayed on the UI, which is the collateral available to a trader wishing to signal a withdraw intent)
Decimal  | frozenCollateral | Collateral available for a smart contract withdrawal, but not for trading, since the Operators have received the withdraw intent)
bool     | frozen | Whether the account and its collateral is frozen or not

## Orders update (account)

You can subscribe to updates to your open orders with the `OrdersUpdate` event. 
Upon subscription, you will receive a `Partial` back, containing a snapshot of all of your open orders at that moment in time. From then on, you will receive streaming/incremental `Update` messages with any changes to these orders (due to posting new orders, canceling existing orders, or orders that have been matched in partial or in full).

```json
{
  "request": {
    "t": "SubscribeAccount",
    "c": {
      "trader": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
      "strategies": ["main"],
      "events": ["OrdersUpdate"]
    }
  }
}
```

> Partial response
```json
{
	"t": "OrdersUpdate",
	"e": "Partial",
	"c": [{
		"orderHash": "0x946e4bedf2dd87e5380f32a18d1af19adb4d7ecec3a8a346cb641adc5201e53e",
		"makerAddress": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
		"symbol": "ETHPERP",
		"side": "0",
		"orderType": "0",
		"requestId": "0x0000000000000000000000000000000000000000000000000000000000000001",
		"amount": "10.000000000000000000",
		"remainingAmount": "10.000000000000000000",
		"price": "521.800000000000000000",
		"stopPrice": "0",
		"signature": "0x",
		"createdAt": "2021-03-24T23:25:41.467Z"
	}]
}
```

> Update response
```json
{
	"t": "OrdersUpdate",
	"e": "Update",
	"c": [{
		"orderHash": "0x946e4bedf2dd87e5380f32a18d1af19adb4d7ecec3a8a346cb641adc5201e53e",
		"makerAddress": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
		"symbol": "ETHPERP",
		"side": "0",
		"orderType": "0",
		"requestId": "0x0000000000000000000000000000000000000000000000000000000000000001",
		"amount": "10",
		"remainingAmount": "0",
		"price": "521.8",
		"stopPrice": "0",
		"signature": "0x",
		"createdAt": "2021-03-24T23:25:41.467Z"
	}]
}
```

**Request parameters**

type | field | description
-----|----- | ---------------------
address  | trader | Trader's Ethereum address (same as the one that facilitated the deposit)
string[] | strategies | Strategies being subscribed to. Currently, the only supported strategy is `main`, but support for multiple strategies is coming soon!
string[] | events | Events being subscribed to. This can be one or more of `OrdersUpdate`, `PositionUpdate`, and `StrategyUpdate`

**Response parameters** - an array of updates per order is emitted, with each update defined as follows:

type | field | description 
------ | ---- | -------
address | orderHash | Hash of the order that has changed or is new
address | makerAddress | Trader's Ethereum address associated with this order
string  | symbol | Name of the market this order belongs to. Currently, this is limited to 'ETHPERP', but new symbols are coming soon! 
bytes32 | strategy | Name of the cross-margined strategy this order belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon!
uint256 | side | Side of order, either `Bid` (buy/long) or an `Ask` (sell/short)
uint256 | orderType | Order type, either `Limit` or `Market`. Other order types coming soon!
bytes32 | requestID | Numeric identifier for the order
Decimal | amount | The original order amount/size requested
Decimal | remainingAmount | The order amount/size remaining on the order book
Decimal | price | The order price
Decimal | stopPrice | Currently, always 0 as stops are not implemented.
bytes   | signature | EIP-712 signature
string  | createdAt | Timestamp when order was initially created


## Positions update (account)

TBD


## Market

Traders can subscribe to data feeds corresponding to events for the broader market. The possible market events traders can subscribe to at this time are `OrderBookUpdate` and `MarkPriceUpdate`. These will be discussed below individually. 

## Order book update (market)

You can subscribe to updates to an L2-order book for any given market with the `OrderBookUpdate` event. As is typically seen, an order book is a price-time FIFO priority queue market place for traders to interact with. Since this is an L2-aggregation, the response will collapse any given price level to the aggregate quantity at that level irrespective of the number of participants or the individual order details that comprise that price level.

Upon subscription, you will receive a `Partial` back, containing an L2-aggregate snapshot of the order book at that moment in time. From then on, you will receive streaming/incremental `Update` messages as appropriate containing only the aggregated price levels that are different (due to the placement of new orders, order cancellation, or liquidations/matches that have taken place). Updates will come in with a reference to the side of the order book and in the form of 2-item lists with the format `[price, new aggregate quantity]`. A new aggregate quantity of `0` indicates that the price level is now gone. 

> Request
```json
{
  "request": {
    "t": "SubscribeMarket",
    "c": {
      "symbols": ["ETHPERP"],
      "events": ["OrderBookUpdate"]
    }
  }
}
```

> Partial response
```json
{
	"t": "OrderBookUpdate",
	"e": "Partial",
	"c": [{
		"bids": [
			["516.58", "10"],
			["511.36", "10"]
		],
		"asks": [],
		"timestamp": "1616664308163",
		"nonce": "0x1848dbc26865f118d0ea56ecf776ee68b5a9b39e0223d4e322374c087a66a201",
		"aggregationType": "0.0001"
	}]
}
```

> Update response
```json
{
	"t": "OrderBookUpdate",
	"e": "Partial",
	"c": [{
		"bids": [
			["516.58", "0"]
		],
		"asks": [
			["527.01", "10"]
		],
		"timestamp": "1616667513875",
		"nonce": "0xc423f94e3ba40527b7bed2925cfedabf8a68388bde06f090a772b8bf3c791f5f",
		"aggregationType": "0.0001"
	}]
}
```

**Request parameters**

type | field | description
-----|----- | ---------------------
string[] | symbols | Symbols being subscribed to. Currently, the only supported symbol is `ETHPERP`, but support for more symbols is coming soon!
string[] | events | Events being subscribed to. This can be one or more of `OrderBookUpdate` and `MarkPriceUpdate`

**Response parameters** - an array of updates per order book is emitted, with each update defined as follows:

type | field | description
-----|----- | ---------------------
Decimal[2][]  | bids | The price and corresponding updated aggregate quantity for the bids
Decimal[2][]  | asks | The price and corresponding updated aggregate quantity for the asks
bytes32  | timestamp | Timestamp of order book partial/update
bytes32  | nonce | ???
Decimal   | aggregationType | ???


## Mark price update (market)

You can subscribe to updates to the mark price for any given market with the `MarkPriceUpdate` event. The mark price is an important concept on DerivaDEX as it is the price at which positions are marked when displaying unrealized PNL. Consequently, the mark price helps determine a strategy's margin fraction, thereby triggering liquidations when appropriate. The mark price is computed based on a 30s exponential moving average (often referred to as a 30s EMA) of the spread between the underlying price (a composite of several spot price feeds for the underlying asset) and the DerivaDEX order book itself. 

Upon subscription, you will receive a `Partial` back, containing a mark price snapshot at that moment in time. From then on, you will receive streaming/incremental `Update` messages as appropriate. 

> Request
```json
{
  "request": {
    "t": "SubscribeMarket",
    "c": {
      "symbols": ["ETHPERP"],
      "events": ["MarkPriceUpdate"]
    }
  }
}
```

> Partial response
```json
{
	"t": "MarkPriceUpdate",
	"e": "Partial",
	"c": [{
		"price": "508.36236848846397",
		"symbol": "ETHPERP",
		"createdAt": "2021-03-25T10:37:38.124Z",
		"updatedAt": "2021-03-25T10:37:38.124Z"
	}]
}
```

> Update response
```json
{
	"t": "MarkPriceUpdate",
	"e": "Update",
	"c": [{
		"createdAt": "2021-03-25T10:38:09.503654+00:00",
		"updatedAt": "2021-03-25T10:38:09.503654+00:00",
		"price": "508.95189310211146",
		"symbol": "ETHPERP"
	}]
}
```

**Request parameters**

type | field | description
-----|----- | ---------------------
string[] | symbols | Symbols being subscribed to. Currently, the only supported symbol is `ETHPERP`, but support for more symbols is coming soon!
string[] | events | Events being subscribed to. This can be one or more of `OrderBookUpdate` and `MarkPriceUpdate`

**Response parameters** - an array of updates per order book is emitted, with each update defined as follows:

type | field | description
-----|----- | ---------------------
string  | createdAt | Timestamp of original mark price entry
string  | updatedAt | Timestamp of mark price update
Decimal | price | Mark price for the market
string  | symbol | Market subscribed to


## Receipts

Each subscription returns a receipt, which confirms that an Operator has received the subscription request. The receipt `type` will be either `Subscribed` or `Error`.

### Subscribed (successful) receipts

A successful subscription returns a `Subscribed` receipt from the Operator.

```json
{
  "receipt": {
    "t": "Subscribed",
    "c": {
      "msg": "Subscribed to OrderBookUpdate for ETHPERP"
    }
  }
}
```

type | field | description
------ | ---- | -----------
string | msg | Success message 

### Error (failed) receipts

An erroneous subscription returns an `Error` receipt from the Operator.

```json
{
  "receipt": {
    "t": "Error",
    "c": {
      "msg": "Invalid subscription"
    }
  }
}
```

type | field | description
------ | ---- | -----------
string | msg | Error message 


# Validation

TBD


# Rate Limits

The websocket API will (eventually) use tiers to determine rate limits for each ethereum account.
Rate limits restrict the number of "Commands" an account can place per minute.

Subscriptions are not rate limited.

