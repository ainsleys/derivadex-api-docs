# DerivaDEX API

DerivaDEX is a decentralized derivatives exchange that combines the performance of centralized exchanges with the security of decentralized exchanges.

Find us online [Discord](https://discord.gg/a54BWuG) | [Telegram](https://t.me/DerivaDEX) | [Medium](https://medium.com/derivadex)

# Getting started

To begin interacting with the DerivaDEX ecosystem programmatically, you generally will want to follow these steps:

1. Deposit funds to the exchange via Ethereum via an Ethereum client (these funds are controlled by you and only you, and are
   cryptographically secure)
2. Run the Auditor client locally (validates the system and serves a local REST and WebSocket API)
3. Connect to the local Auditor WebSocket API (subscribe to state and transaction updates propagated
   in real time from the exchange) & REST API (retrieve state entries)
4. Connect to the Operator REST API (submit and cancel orders via `commands`)
5. Connect to the Collections REST API (receive historical fills)

This set up provides you with the security, integrity, and trust model of
a robust decentralized system, with the performance and API experience of a centralized exchange.

Additionally, it's a good idea to familiarize yourself with the [DerivaDEX types terminology](#Types) and [hashing & signing schemes](#signatures--hashing), although the examples and sample client libraries abstract much of this complexity away.

## Auditor API

The pre-packaged Pythonic Auditor connects to the DerivaDEX Operators WebSocket API upon initialization and underpins
an extremely powerful, efficient, and easy-to-use setup for programmatic traders by allowing you to:

1. validate the honesty and integrity of the exchange's operations
2. use the Auditor as a backend datastore and API to ensure you are verifiably up-to-date with the latest exchange data

**Please be extra careful to always do your own testing and validation of any of this tooling.**

DerivaDEX's data model is powered by a Sparse Merkle Tree data store and a transaction log of state-modifying transitions. While these are emitted in raw formats by the DerivaDEX Operators API, they are not
the most easily-consumable. Thus, the Auditor provides a worthwhile abstraction that efficiently allows you to locally maintain your own data validation while exposing an API for your easy consumption.
If you would like to dive deeper into how the Auditor processes the state and raw transactions on DerivaDEX or would
like to write your own Auditor tooling (in a different language perhaps, or to expose an interface more to your liking), please check out the [Operator API <> Auditor](#trader-api--auditor) section.
Otherwise, you are more than welcome to just check out the Setup section to get the Python tooling running and serving your trading
needs.

## Setup

### From source

To run the Auditor from source, follow these steps from the DerivaDEX `trading_clients` repository (< 5 minutes):

1. If you don't already have it, we recommend setting up [Anaconda/Python(>3)](https://docs.anaconda.com/anaconda/install/index.html) on your machine

2. Initialize and activate a Conda environment (located at the root level of the repo) from which you will run the Auditor: `conda env create -f environment.yml && conda activate derivadex`

3. Set your PYTHONPATH: `export PYTHONPATH="${PYTHONPATH}:/your/full/path/upto/but/not/including/trading_clients"`

4. Navigate to the `auditor` subdirectory and create an `.auditor.conf.json` file using the template: `cp .auditor.conf.json.template .auditor.conf.json`

5. Run the Auditor with: `PYTHON_LOG=info python auditor_driver.py --config ".auditor.conf.json"`

You will almost immediately see logging messages (should be lots of green) with state initialized and transactions streaming through successfully.

Voila! You now have the Auditor running locally validating the **entire** DerivaDEX exchange, and serving as a local REST & WebSocket API
for whatever your trading needs may be!

# Command API (Operator)

Sending commands or requests (placing orders, canceling orders, and withdrawing funds) is something you can do vs. the Operator directly.

Since commands modify system state, these requests must include an [EIP-712 signature](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md). Examples are included in the [sample code](samples.md).

**Please keep in mind, all sample command requests displayed below are NOT encrypted. The reason for this is to more clearly display the formats of the various commands you may send. You MUST encrypt these messages prior to sending them to the exchange.**

## Place order

### Request

> Request format (JSON)

```json
{
    "t": "Order",
    "c": {
        "symbol": "ETHPERP",
        "strategy": "main",
        "side": "Bid",
        "orderType": "Limit",
        "nonce": "0x3136313839303336353634383833373230303000000000000000000000000000",
        "amount": 10,
        "price": 487.5,
        "stopPrice": 0,
        "signature": "0xd5a1ca6d40a030368710ab86d391e5d16164ea16d2c809894eefddd1658bb08c6898177aa492d4d45272ee41cb40f252327a23e8d1fc2af6904e8860d3f72b3b1b"
    }
}
```

You can place new orders by specifying specific attributes in the `Order` command's request payload. These requests are subject to a set of validations.

| type      | field       | description                                                                                                                                                               |
| --------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t           | Command type, which in this case will be `Order`                                                                                                                          |
| dict      | c           | Command contents containing the order being placed                                                                                                                        |
| string    | c.symbol    | Name of the market to trade. Currently, this is limited to `ETHPERP`, but new symbols are coming soon!                                                                    |
| string    | c.strategy  | Name of the cross-margined strategy this trade belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon! |
| string    | c.side      | Side of trade, either `Bid` (buy/long) or an `Ask` (sell/short)                                                                                                           |
| string    | c.orderType | Order type, either `Limit` or `Market`. Other order types coming soon!                                                                                                    |
| bytes32_s | c.nonce     | An incrementing numeric identifier for this request that is unique per user for all time                                                                                  |
| decimal   | c.amount    | The order amount/size requested                                                                                                                                           |
| decimal   | c.price     | The order price (If `orderType` is `Market`, this must be set to `0`)                                                                                                     |
| decimal   | c.stopPrice | Currently, always set to `0` as stops are not implemented.                                                                                                                |
| bytes_s   | c.signature | EIP-712 signature of the order placement intent                                                                                                                           |

To be more explicit, all of these fields must be passed in, even if not all of the fields apply due to certain functionalities not currently implemented (i.e. stops) or the fact that prices aren't applicable in the case of market orders. Please follow the guidelines specified in the table above around these conditions.

### Response

> Receipt (success) format (JSON)

```json
{
    "t": "Sequenced",
    "c": {
        "nonce": "0x3136323631383732373739373732303230303000000000000000000000000000",
        "requestHash": "0xdfcdd015a63119477e456e48ab16a6324cb4820b99c47d8a0520f1d43834d7ef",
        "requestIndex": 54913,
        "sender": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "enclaveSignature": "0x69865fca4901d21c84e1fd9b12b0e6ba4d7f7a885594fa8a1aa4dfdaf0dc7b6f433069b38c7822d215e55b3d46118ed012c874bf524fe73cb6aafe9adb42d3e81c"
    }
}
```

A place order `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Sequenced` or `Error`.

A successful `command` returns a `Sequenced` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

| type      | field              | description                                                                                                                              |
| --------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                  | Message type, which in this case will be `Sequenced`                                                                                     |
| dict      | c                  | Message contents                                                                                                                         |
| bytes32_s | c.nonce            | The same `nonce` that was passed in the initial request, which can be used to correlate your initial requests with receipts              |
| bytes32_s | c.requestHash      | Hash of the request                                                                                                                      |
| int       | c.requestIndex     | A ticket number which guarantees fair sequencing. All tickets are processed and handled by the exchange in order of this `requestIndex`. |
| address_s | c.sender           | The request sender"s Ethereum address                                                                                                    |
| bytes_s   | c.enclaveSignature | An Operator"s signature which proves secure handling of the request                                                                      |

> Receipt (error) format (JSON)

```json
{
    "t": "Error",
    "c": {
        "message": "Error: timeout of 2000ms exceeded"
    }
}
```

An erroneous command returns an `Error` receipt from the Operator.

| type   | field | description   |
| ------ | ----- | ------------- |
| string | msg   | Error message |

## Cancel order

### Request

> Request format (JSON)

```json
{
    "t": "CancelOrder",
    "c": {
        "symbol": "ETHPERP",
        "orderHash": "0xedee9c27b4fc64a481bd45c145eaf35806d6ad49ca0c68890a00000000000000",
        "nonce": "0x3136323635353034303835323735383330303000000000000000000000000000",
        "signature": "0xee6c271fc010e25fb28556b39f8999d832485b03335c9c4a5ceca84455ce6bb205483995f5240e62adeb50bda12ed8db67a990c0930e5512709b3bcff4a98ca01b"
    }
}
```

You can cancel existing orders by specifying specific attributes in the `CancelOrder` command"s request payload.

| type      | field       | description                                                                              |
| --------- | ----------- | ---------------------------------------------------------------------------------------- |
| string    | t           | Command type, which in this case will be `CancelOrder`                                   |
| dict      | c           | Command contents containing the order being canceled                                     |
| string    | c.symbol    | Currently always `ETHPERP`. New symbols coming soon!                                     |
| bytes32_s | c.orderHash | The first 25 bytes of the order"s unique hash that is being canceled.                    |
| bytes32_s | c.nonce     | An incrementing numeric identifier for this request that is unique per user for all time |
| bytes_s   | c.signature | EIP-712 signature of the order cancellation request                                      |

As described in the `Signatures & hashing` section, the `orderHash` is something that you construct client-side prior to submitting the order to the exchange. In this regard, you have the `orderHash` for each order you submit irrespective of acknowledgement from the exchange. However, you likely will fire order cancellations after you have already had acknowledgement of placement receipt from the exchange. You can obtain the order hashes of your open orders using appropriately defined REST requests or WebSocket subscriptions.

### Response

> Receipt (success) format (JSON)

```json
{
    "t": "Sequenced",
    "c": {
        "nonce": "0x3136323635353034303835323735383330303000000000000000000000000000",
        "requestHash": "0xdab45fe8ddac0cf1231f79bf4fcbfa847606f45341b446d143b0b0688aa7eed0",
        "requestIndex": 108095,
        "sender": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "enclaveSignature": "0x040d24750b994b3603cabb4097093bc310fbfcbe88246501fac4f1d9a441798b20a09eedfdbe52a576cba17ed3986984ba0bfe4a8a3141a525759e7c39f49a441b"
    }
}
```

A cancel order `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Sequenced` or `Error`.

A successful `command` returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

| type      | field              | description                                                                                                                              |
| --------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                  | Message type, which in this case will be `Sequenced`                                                                                     |
| dict      | c                  | Message contents                                                                                                                         |
| bytes32_s | c.nonce            | The same `nonce` that was passed in the initial request, which can be used to correlate your initial requests with receipts              |
| bytes32_s | c.requestHash      | Hash of the request                                                                                                                      |
| int       | c.requestIndex     | A ticket number which guarantees fair sequencing. All tickets are processed and handled by the exchange in order of this `requestIndex`. |
| address_s | c.sender           | The request sender"s Ethereum address                                                                                                    |
| bytes_s   | c.enclaveSignature | An Operator"s signature which proves secure handling of the request                                                                      |

> Receipt (error) format (JSON)

```json
{
    "t": "Error",
    "c": {
        "message": "Error: timeout of 2000ms exceeded"
    }
}
```

An erroneous command returns an `Error` receipt from the Operator.

| type   | field | description   |
| ------ | ----- | ------------- |
| string | msg   | Error message |

## Cancel all

### Request

> Request format (JSON)

```json
{
    "t": "CancelAll",
    "c": {
        "strategyId": "main",
        "nonce": "0x3136323635353034303835323735383330303000000000000000000000000000",
        "signature": "0xee6c271fc010e25fb28556b39f8999d832485b03335c9c4a5ceca84455ce6bb205483995f5240e62adeb50bda12ed8db67a990c0930e5512709b3bcff4a98ca01b"
    }
}
```

You can cancel all existing orders by specifying specific attributes in the `CancelAll` command's request payload.

| type      | field        | description                                                                                                                                                               |
| --------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t            | Command type, which in this case will be `CancelOrder`                                                                                                                    |
| dict      | c            | Command contents containing the order being canceled                                                                                                                      |
| string    | c.strategyId | Name of the cross-margined strategy this trade belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon! |
| bytes32_s | c.nonce      | An incrementing numeric identifier for this request that is unique per user for all time                                                                                  |
| bytes_s   | c.signature  | EIP-712 signature of the order cancellation request                                                                                                                       |

### Response

> Receipt (success) format (JSON)

```json
{
    "t": "Sequenced",
    "c": {
        "nonce": "0x3136323635353034303835323735383330303000000000000000000000000000",
        "requestHash": "0xdab45fe8ddac0cf1231f79bf4fcbfa847606f45341b446d143b0b0688aa7eed0",
        "requestIndex": 108095,
        "sender": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "enclaveSignature": "0x040d24750b994b3603cabb4097093bc310fbfcbe88246501fac4f1d9a441798b20a09eedfdbe52a576cba17ed3986984ba0bfe4a8a3141a525759e7c39f49a441b"
    }
}
```

A cancel order `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Sequenced` or `Error`.

A successful `command` returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

| type      | field              | description                                                                                                                              |
| --------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                  | Message type, which in this case will be `Sequenced`                                                                                     |
| dict      | c                  | Message contents                                                                                                                         |
| bytes32_s | c.nonce            | The same `nonce` that was passed in the initial request, which can be used to correlate your initial requests with receipts              |
| bytes32_s | c.requestHash      | Hash of the request                                                                                                                      |
| int       | c.requestIndex     | A ticket number which guarantees fair sequencing. All tickets are processed and handled by the exchange in order of this `requestIndex`. |
| address_s | c.sender           | The request sender"s Ethereum address                                                                                                    |
| bytes_s   | c.enclaveSignature | An Operator"s signature which proves secure handling of the request                                                                      |

> Receipt (error) format (JSON)

```json
{
    "t": "Error",
    "c": {
        "message": "Error: timeout of 2000ms exceeded"
    }
}
```

An erroneous command returns an `Error` receipt from the Operator.

| type   | field | description   |
| ------ | ----- | ------------- |
| string | msg   | Error message |

## Withdraw

### Request

> Request format (JSON)

```json
{
    "t": "Withdraw",
    "c": {
        "recipientAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
        "strategyId": "main",
        "currency": "0x41082c820342539de44c1b404fead3b4b39e15d6",
        "amount": 440.32,
        "nonce": "0x3136313839303336353634383833373230303000000000000000000000000000",
        "signature": "0xd5a1ca6d40a030368710ab86d391e5d16164ea16d2c809894eefddd1658bb08c6898177aa492d4d45272ee41cb40f252327a23e8d1fc2af6904e8860d3f72b3b1b"
    }
}
```

You can signal withdrawal intents to the Operators by specifying specific attributes in the `Withdraw` command"s request payload. Withdrawal is a 2-step process: submitting a withdrawal intent, and performing a smart contract withdrawal. Once a withdrawal intent is initiated, you won"t be able to trade with the collateral you are attempting to withdraw. You will only be able to formally initiate a smart contract withdrawal/token transfer once the epoch in which you signal your withdrawal desire has concluded.

| type      | field              | description                                                                                                                                                                                                        |
| --------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| string    | t                  | Command type, which in this case will be `Withdraw`                                                                                                                                                                |
| dict      | c                  | Command contents containing the withdrawal data                                                                                                                                                                    |
| address_s | c.recipientAddress | Trader's Ethereum address (same as the one that facilitated the deposit)                                                                                                                                           |
| string    | c.strategyId       | Name of the cross-margined strategy this withdrawal belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon!                                     |
| address_s | c.currency         | ERC-20 token address being withdrawn                                                                                                                                                                               |
| decimal   | c.amount           | Amount withdrawn (be sure to use the grains format specific to the collateral token being used (e.g. if you wanted to withdraw 1 USDC, you would enter 1000000 since the USDC token contract has 6 decimal places) |
| bytes32_s | c.nonce            | An incrementing numeric identifier for this request that is unique per user for all time                                                                                                                           |
| bytes_s   | c.signature        | EIP-712 signature for the withdrawal request                                                                                                                                                                       |

### Response

> Receipt (success) format (JSON)

```json
{
    "t": "Sequenced",
    "c": {
        "nonce": "0x3136323631383732373739373732303230303000000000000000000000000000",
        "requestHash": "0xdfcdd015a63119477e456e48ab16a6324cb4820b99c47d8a0520f1d43834d7ef",
        "requestIndex": 54913,
        "sender": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "enclaveSignature": "0x69865fca4901d21c84e1fd9b12b0e6ba4d7f7a885594fa8a1aa4dfdaf0dc7b6f433069b38c7822d215e55b3d46118ed012c874bf524fe73cb6aafe9adb42d3e81c"
    }
}
```

A withdraw `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Received` or `Error`.

A successful `command` returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

| type      | field              | description                                                                                                                              |
| --------- | ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                  | Message type, which in this case will be `Sequenced`                                                                                     |
| dict      | c                  | Message contents                                                                                                                         |
| bytes32_s | c.nonce            | The same `nonce` that was passed in the initial request, which can be used to correlate your initial requests with receipts              |
| bytes32_s | c.requestHash      | Hash of the request                                                                                                                      |
| int       | c.requestIndex     | A ticket number which guarantees fair sequencing. All tickets are processed and handled by the exchange in order of this `requestIndex`. |
| address_s | c.sender           | The request sender"s Ethereum address                                                                                                    |
| bytes_s   | c.enclaveSignature | An Operator"s signature which proves secure handling of the request                                                                      |

> Receipt (error) format (JSON)

```json
{
    "t": "Error",
    "c": {
        "message": "Error: timeout of 2000ms exceeded"
    }
}
```

An erroneous command returns an `Error` receipt from the Operator.

| type   | field | description   |
| ------ | ----- | ------------- |
| string | msg   | Error message |

# REST API (Auditor)

The local Auditor running exposes a simple REST API for your consumption. The currently
available endpoints are described below, but you are of course more than welcome to add your own or modify
the Auditor implementation to expose a REST interface that best suits your needs.

REST endpoint URL (if running using Docker): http://ddx-auditor:8766
REST endpoint URL (if running using source): http://localhost:8766

## Trader

### Request

> Sample Trader REST query

```json
GET /trader/topic={topic}

e.g. topic = STATE/TRADER/
e.g. topic = STATE/TRADER/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/
```

You may obtain the `Trader` leaf details for any trader address with the following query parameter definition:

| type   | field | description                                                                                                   |
| ------ | ----- | ------------------------------------------------------------------------------------------------------------- |
| string | topic | Must be of the format `STATE/TRADER/[<trader_address>/]`. `trader_address` should be an `address_pre_s` type. |

### Response

> Sample trader REST response

```json
{
    "t": "Success",
    "c": [
        {
            "t": "STATE/TRADER/0x00a8dda8d7f5310e4a9e24f8eba77e091ac264f872/",
            "c": {
                "freeDDXBalance": "2557.077625570776255707",
                "frozenDDXBalance": "0",
                "referralAddress": "0x0000000000000000000000000000000000000000",
                "payFeesInDDX": true
            }
        },
        {
            "t": "STATE/TRADER/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/",
            "c": {
                "freeDDXBalance": "1917.808219178082191778",
                "frozenDDXBalance": "0",
                "referralAddress": "0x0000000000000000000000000000000000000000",
                "payFeesInDDX": true
            }
        }
    ]
}
```

A sample response back is shown on the right, with fields defined as follows:

| type      | field                   | description                                                                                                                                                                                    |
| --------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                       | Whether HTTP request was successful or not                                                                                                                                                     |
| list      | c                       | HTTP response result returning a list of `Trader` leaves                                                                                                                                       |
| dict      | c[n]                    | An individual `Trader` leaf, containing the identifying key information and its contents                                                                                                       |
| string    | c[n].t                  | `Trader` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/TRADER/<trader_address>/`. `trader_address` should be an `address_pre_s` type. |
| dict      | c[n].c                  | `Trader` leaf contents                                                                                                                                                                         |
| decimal_s | c[n].c.freeDDXBalance   | Trader's free DDX collateral (available for use on the exchange)                                                                                                                               |
| decimal_s | c[n].c.frozenDDXBalance | Trader's frozen DDX collateral (available for withdrawal)                                                                                                                                      |
| address_s | c[n].c.referralAddress  | The Ethereum address of the trader who referred this one                                                                                                                                       |
| bool      | c[n].c.payFeesInDDX     | Whether trader has opted to pay fees in DDX by default or not                                                                                                                                  |

## Strategy

### Request

> Sample Strategy REST query

```json
GET /strategy/topic={topic}

e.g. topic = STATE/STRATEGY/
e.g. topic = STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/
e.g. topic = STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/
```

You may obtain the `Strategy` leaf details for any trader and strategy id with the following query parameter definition:

| type   | field | description                                                                                                                                                                                                                                                              |
| ------ | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| string | topic | Must be of the format `STATE/STRATEGY/[<trader_address>/][[abbrev_strategy_id_hash]]`. `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |

### Response

> Sample strategy REST response

```json
{
    "t": "Success",
    "c": [
        {
            "t": "STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
            "c": {
                "strategyId": "main",
                "freeCollateral": {
                    "0xb69e673309512a9d726f87304c6984054f87a93b": "1008.334242735906152015"
                },
                "frozenCollateral": {},
                "maxLeverage": 3,
                "frozen": false
            }
        }
    ]
}
```

A sample response back is shown on the right, with fields defined as follows:

| type                       | field                   | description                                                                                                                                                                                                                                                                                                                                                |
| -------------------------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string                     | t                       | Whether HTTP request was successful or not                                                                                                                                                                                                                                                                                                                 |
| list                       | c                       | HTTP response result returning a list of `Strategy` leaves                                                                                                                                                                                                                                                                                                 |
| dict                       | c[n]                    | An individual `Strategy` leaf, containing the identifying key information and its contents                                                                                                                                                                                                                                                                 |
| string                     | c[n].t                  | `Strategy` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/STRATEGY/<trader_address>/<abbrev_strategy_id_hash>/`. `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4 bytes of the hash of the strategy id (e.g. `main`). |
| dict                       | c[n].c                  | `Strategy` leaf contents                                                                                                                                                                                                                                                                                                                                   |
| string                     | c[n].c.strategyId       | Cross-margined strategy identifier. Currently only `main` is supported.                                                                                                                                                                                                                                                                                    |
| dict<address_s, decimal_s> | c[n].c.freeCollateral   | Strategy"s free collateral (available for trading) mapping from ERC-20 collateral token address to amount                                                                                                                                                                                                                                                  |
| dict<address_s, decimal_s> | c[n].c.frozenCollateral | Strategy"s frozen collateral (available for withdrawal) mapping from ERC-20 collateral token address to amount                                                                                                                                                                                                                                             |
| int                        | c[n].c.maxLeverage      | Strategy"s maximum leverage setting                                                                                                                                                                                                                                                                                                                        |
| bool                       | c[n].c.frozen           | Flag indicating whether strategy is tokenized (`true`) or not (`false`). Currently only `false` is supported                                                                                                                                                                                                                                               |

## Position

### Request

> Sample Position REST query

```json
GET /position/topic={topic}

e.g. topic = STATE/POSITION/
e.g. topic = STATE/POSITION/ETHPERP/
e.g. topic = STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/
e.g. topic = STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/
```

You may obtain the `Position` leaf details for any symbol, chain discriminant, trader, and strategy id with the following query parameter definition:

| type   | field | description                                                                                                                                                                                                                                                                                                                              |
| ------ | ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string | topic | Must be of the format `STATE/POSITION/[<symbol>/][[<trader_address>/]][[[<abbrev_strategy_id_hash>/]]]`. `symbol` is a `string` type (e.g. `ETHPERP`), `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4 bytes of the hash of the strategy id (e.g. `main`). |

### Response

> Sample Position REST response

```json
{
    "t": "Success",
    "c": [
        {
            "t": "STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
            "c": {
                "side": "Long",
                "balance": "0.1",
                "avgEntryPrice": "2300"
            }
        }
    ]
}
```

A sample response back is shown on the right, with fields defined as follows:

| type      | field                | description                                                                                                                                                                                                                                                                                                                                                                                                      |
| --------- | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                    | Whether HTTP request was successful or not                                                                                                                                                                                                                                                                                                                                                                       |
| list      | c                    | HTTP response result returning a list of `Position` leaves                                                                                                                                                                                                                                                                                                                                                       |
| dict      | c[n]                 | An individual `Position` leaf, containing the identifying key information and its contents                                                                                                                                                                                                                                                                                                                       |
| string    | c[n].t               | `Position` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/POSITION/<symbol>/<trader_address>/<abbrev_strategy_id_hash>`. `symbol` is a `string` type (e.g. `ETHPERP`), `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |
| dict      | c[n].c               | `Position` leaf contents                                                                                                                                                                                                                                                                                                                                                                                         |
| int       | c[n].c.side          | Position side (`Long`, `Short`)                                                                                                                                                                                                                                                                                                                                                                                  |
| decimal_s | c[n].c.balance       | Size of position                                                                                                                                                                                                                                                                                                                                                                                                 |
| decimal_s | c[n].c.avgEntryPrice | Average entry price of position                                                                                                                                                                                                                                                                                                                                                                                  |

## Book order

### Request

> Sample BookOrder REST query

```json
GET /book_order/topic={topic}

e.g. topic = STATE/BOOK_ORDER/
e.g. topic = STATE/BOOK_ORDER/ETHPERP/
e.g. topic = STATE/BOOK_ORDER/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/
e.g. topic = STATE/BOOK_ORDER/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/
```

You may obtain book order information (`BookOrder` leaves) for any symbol, trader, and strategy id with the following query parameter definition:

| type   | field | description                                                                                                                                                                                                                                                                                                                             |
| ------ | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string | topic | Must be of the format `STATE/BOOK_ORDER/[<symbol>/][[<trader_address>/]][[[<abbrev_strategy_id_hash>/]]]`. `symbol` is a `string` type (e.g. `ETHPERP`), `trader_address` should be an `address_pre_s` type, `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |

This endpoint can be used for a variety of use-cases:

-   L3 order book: An L3 order book is a granular, order-by-order view of the entire market for any given symbol (i.e. no
    aggregation). You are welcome to consume this data and aggregate as you see fit. You can obtain this order book view using
    a topic such as `STATE/BOOK_ORDER/ETHPERP/`, where `ETHPERP` is an example market you may be requesting.
-   Open orders: You can obtain your open orders for a given trader and strategy via this REST endpoint at any given time using a topic
    such as `STATE/BOOK_ORDER/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/`, where `0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84` is
    an example Ethereum address prefixed with the chain discriminant you are trading with and `0x2576ebd1` is the first 4 bytes of the hash of the strategy
    you are trading from (e.g. `main`)

### Response

> Sample BookOrder REST response

```json
{
    "t": "Success",
    "c": [
        {
            "t": "STATE/BOOK_ORDER/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/0x56de91db6731ea375171888896c87234f6b2a49513e7890d48/",
            "c": {
                "side": "Bid",
                "amount": "5.42",
                "price": "2381.92",
                "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
                "strategyIdHash": "0x2576ebd1",
                "bookOrdinal": 3
            }
        },
        {
            "t": "STATE/BOOK_ORDER/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/0x67069f8829cbdfa2a0c2b215e80c05482699ac1d1dfb6f4c20/",
            "c": {
                "side": "Ask",
                "amount": "2",
                "price": "2390",
                "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
                "strategyIdHash": "0x2576ebd1",
                "bookOrdinal": 4
            }
        }
    ]
}
```

A sample response back is shown on the right, with fields defined as follows:

| type          | field                 | description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| string        | t                     | Whether HTTP request was successful or not                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| list          | c                     | HTTP response result returning a list of open `BookOrder` leaves                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| dict          | c[n]                  | An individual `BookOrder` leaf, containing the identifying key information and its contents                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| string        | c[n].t                | Book order topic (corresponding to its key), containing identifying information. This key is of the format `STATE/BOOK_ORDER/<symbol>/<trader_address>/<abbrev_strategy_id_hash>/<order_hash>/`. `symbol` is a `string` type (e.g. `ETHPERP`), `trader_address` should be an `address_pre_s` type, `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`), and "order_hash" is a `bytes_s` type representing the first 25-bytes of the order"s unique hash. |
| dict          | c[n].c                | `BookOrder` leaf contents                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| int           | c[n].c.side           | Order side (`Bid`, `Ask`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| decimal_s     | c[n].c.amount         | Size of order                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| decimal_s     | c[n].c.price          | Price order is currently resting at                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| address_s_pre | c[n].c.traderAddress  | Trader"s Ethereum address (prefixed with the chain discriminant) responsible for this maker order                                                                                                                                                                                                                                                                                                                                                                                                                              |
| bytes_s       | c[n].c.strategyIdHash | Trader"s abbreviated strategy id hash (first 4 bytes of the hash of the strategy id) responsible for this maker order                                                                                                                                                                                                                                                                                                                                                                                                          |
| int           | c[n].c.bookOrdinal    | Numeric sequencing identifier for order, can be used to arrange multiple orders at the same price level in a locally-maintained order book                                                                                                                                                                                                                                                                                                                                                                                     |

# WebSocket API (Auditor)

The local Auditor running exposes a simple WebSocket API for your consumption. The currently available endpoints are described below, but you are of course more than welcome to add your own or modify the Auditor implementation to expose a WebSocket interface that best suits your needs.

You can use the WebSocket API to receive data per `subscriptions`. We broadly group them into two categories:

1. State events: for when a state/leaf item changes
2. Transaction log events: for when a state-changing transaction has been emitted. These events are what triggers state to change.

WS endpoint URL (if running using Docker): ws://localhost:8765
WS endpoint URL (if running using source): ws://ddx-auditor:8765

## State events

State events can be subscribed to track changes to specific state entries on DerivaDEX.

### Trader

You may subscribe to updates for any trader"s `Trader` data.

#### Request

> Sample Trader state subscription (JSON)

```json
// All traders
{
	"t": "Subscribe",
	"c": "STATE/TRADER/"
}

// Trader for a trader address
{
	"t": "Subscribe",
	"c": "STATE/TRADER/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/"
}
```

| type   | field | description                                                                                                              |
| ------ | ----- | ------------------------------------------------------------------------------------------------------------------------ |
| string | t     | WebSocket message type - must be `Subscribe`                                                                             |
| string | c     | Subscription topic of the format `STATE/TRADER/[<trader_address>/]`. `trader_address` should be an `address_pre_s` type. |

#### Response

##### Partial

> Sample Trader leaf event partial response (JSON)

```json
{
    "t": "STATE/TRADER/",
    "e": "Partial",
    "c": {
        "itemData": [
            {
                "t": "STATE/TRADER/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/",
                "c": {
                    "freeDDXBalance": "2557.077625570776255707",
                    "frozenDDXBalance": "0",
                    "referralAddress": "0x0000000000000000000000000000000000000000",
                    "payFeesInDDX": true
                }
            },
            {
                "t": "STATE/TRADER/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/",
                "c": {
                    "freeDDXBalance": "639.269406392694063926",
                    "frozenDDXBalance": "0",
                    "referralAddress": "0x0000000000000000000000000000000000000000",
                    "payFeesInDDX": false
                }
            }
        ],
        "eventTrigger": null
    }
}
```

Upon subscription, you will first receive an initial snapshot of the current trader data. A sample response is shown
on the right, with fields defined as follows:

| type      | field                            | description                                                                                                                                                                                    |
| --------- | -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                                | WebSocket message type, pertaining to the topic subscribed to                                                                                                                                  |
| string    | e                                | Websocket message event, which in the case of a snapshot, will be `Partial`                                                                                                                    |
| dict      | c                                | Websocket message content, containing a snapshot of `Trader` leaf data based on the topic subscribed to                                                                                        |
| list      | c.itemData                       | Contains a list of the snapshotted `Trader` leaf data                                                                                                                                          |
| dict      | c.itemData[n]                    | An individual `Trader` leaf as part of the snapshot                                                                                                                                            |
| string    | c.itemData[n].t                  | `Trader` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/TRADER/<trader_address>/`. `trader_address` should be an `address_pre_s` type. |
| dict      | c.itemData[n].c                  | `Trader` leaf contents                                                                                                                                                                         |
| decimal_s | c.itemData[n].c.freeDDXBalance   | Trader's free DDX collateral (available for use on the exchange)                                                                                                                               |
| decimal_s | c.itemData[n].c.frozenDDXBalance | Trader's frozen DDX collateral (available for withdrawal)                                                                                                                                      |
| address_s | c.itemData[n].c.referralAddress  | The Ethereum address of the trader who referred this one                                                                                                                                       |
| bool      | c.itemData[n].c.payFeesInDDX     | Whether trader has opted to pay fees in DDX by default or not                                                                                                                                  |

##### Update

> Sample Trader leaf event update response (JSON)

```json
{
    "t": "STATE/TRADER/",
    "e": "Update",
    "c": {
        "itemData": {
            "t": "STATE/TRADER/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/",
            "c": {
                "freeDDXBalance": "1278.538812785388127852",
                "frozenDDXBalance": "0",
                "referralAddress": "0x0000000000000000000000000000000000000000",
                "payFeesInDDX": false
            }
        },
        "eventTrigger": {
            "eventType": 11,
            "requestIndex": 16823,
            "tradeMiningEpochId": 21,
            "ddxDistributed": "3196.347031963470319633",
            "totalVolume": {
                "makerVolume": "0.1",
                "takerVolume": "0.1"
            },
            "timeValue": 2365
        }
    }
}
```

After the initial snapshot response, you will receive streaming `Update` messages with updates to the strategy leaf. A sample response is shown
on the right, with fields defined as follows:

| type      | field                         | description                                                                                                                                                                                                                                                                                                                                                             |
| --------- | ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                             | WebSocket message type, pertaining to the topic subscribed to                                                                                                                                                                                                                                                                                                           |
| string    | e                             | Websocket message event, which in the case of an update, will be `Update`                                                                                                                                                                                                                                                                                               |
| dict      | c                             | Websocket message content, containing the `Trader` leaf update event data                                                                                                                                                                                                                                                                                               |
| dict      | c.itemData                    | Contains the new `Trader` leaf data                                                                                                                                                                                                                                                                                                                                     |
| string    | c.itemData.t                  | `Trader` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/TRADER/<trader_address>/`. `trader_address` should be a `address_pre_s` type.                                                                                                                                                                           |
| string    | c.itemData.c                  | `Trader` leaf contents                                                                                                                                                                                                                                                                                                                                                  |
| decimal_s | c.itemData.c.freeDDXBalance   | Trader"s free DDX collateral (available for use on the exchange)                                                                                                                                                                                                                                                                                                        |
| decimal_s | c.itemData.c.frozenDDXBalance | Trader"s frozen DDX collateral (available for withdrawal)                                                                                                                                                                                                                                                                                                               |
| address_s | c.itemData.c.referralAddress  | The Ethereum address of the trader who referred this one                                                                                                                                                                                                                                                                                                                |
| address_s | c.itemData.c.payFeesInDDX     | Whether trader has opted to pay fees in DDX by default or not                                                                                                                                                                                                                                                                                                           |
| dict      | c.eventTrigger                | The transaction log event that triggered an update. Since this is an `Update` response, there will be a transaction log event that triggered it. The possible events that can result in a an update to a strategy are `StrategyUpdate`, `WithdrawDDX`, and `TradeMining`. Please check out each transaction type in these docs for additional information as necessary. |

### Strategy

You may subscribe to updates for any trader's `Strategy` data.

#### Request

> Sample Strategy state subscription (JSON)

```json
// All strategies
{
	"t": "Subscribe",
	"c": "STATE/STRATEGY/"
}

// All strategies for a trader address
{
	"t": "Subscribe",
	"c": "STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/"
}

// Strategy for a trader address and strategy id
{
	"t": "Subscribe",
	"c": "STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/"
}
```

| type   | field | description                                                                                                                                                                                                                                                                            |
| ------ | ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string | t     | WebSocket message type - must be `Subscribe`                                                                                                                                                                                                                                           |
| string | c     | Subscription topic of the format `STATE/STRATEGY/[<trader_address>/][[<abbrev_strategy_id_hash>/]]`. `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |

#### Response

##### Partial

> Sample Strategy leaf event partial response (JSON)

```json
{
    "t": "STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
    "e": "Partial",
    "c": {
        "itemData": [
            {
                "t": "STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
                "c": {
                    "strategyId": "main",
                    "freeCollateral": {
                        "0xb69e673309512a9d726f87304c6984054f87a93b": "108744.388472786563252582"
                    },
                    "frozenCollateral": {},
                    "maxLeverage": 20,
                    "frozen": false
                }
            }
        ],
        "eventTrigger": null
    }
}
```

Upon subscription, you will first receive an initial snapshot of the current strategy data. A sample response is shown
on the right, with fields defined as follows:

| type                       | field                            | description                                                                                                                                                                                                                                                                                                                                               |
| -------------------------- | -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string                     | t                                | WebSocket message type, pertaining to the topic subscribed to                                                                                                                                                                                                                                                                                             |
| string                     | e                                | Websocket message event, which in the case of a snapshot, will be `Partial`                                                                                                                                                                                                                                                                               |
| dict                       | c                                | Websocket message content, containing a snapshot of `Strategy` leaf data based on the topic subscribed to                                                                                                                                                                                                                                                 |
| list                       | c.itemData                       | Contains a list of the snapshotted `Strategy` leaf data                                                                                                                                                                                                                                                                                                   |
| dict                       | c.itemData[n]                    | An individual `Strategy` leaf as part of the snapshot                                                                                                                                                                                                                                                                                                     |
| string                     | c.itemData[n].t                  | `Strategy` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/STRATEGY/<trader_address>/<abbrev_strategy_id_hash>`. `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |
| dict                       | c.itemData[n].c                  | `Strategy` leaf contents                                                                                                                                                                                                                                                                                                                                  |
| string                     | c.itemData[n].c.strategyId       | Cross-margined strategy identifier. Currently only `main` is supported.                                                                                                                                                                                                                                                                                   |
| dict<address_s, decimal_s> | c.itemData[n].c.freeCollateral   | Strategy's free collateral (available for trading) mapping from ERC-20 collateral token address to amount                                                                                                                                                                                                                                                 |
| dict<address_s, decimal_s> | c.itemData[n].c.frozenCollateral | Strategy's free collateral (available for withdrawal) mapping from ERC-20 collateral token address to amount                                                                                                                                                                                                                                              |
| int                        | c.itemData[n].c.maxLeverage      | Strategy's maximum leverage setting                                                                                                                                                                                                                                                                                                                       |
| bool                       | c.itemData[n].c.frozen           | Flag indicating whether strategy is tokenized (`true`) or not (`false`). Currently only `false` is supported                                                                                                                                                                                                                                              |
| dict                       | c.eventTrigger                   | The transaction log event that triggered an update. Since this is a `Partial` response, there is no update, thus will always be null.                                                                                                                                                                                                                     |

##### Update

> Sample Strategy leaf event update response (JSON)

```json
{
    "t": "STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
    "e": "Update",
    "c": {
        "itemData": {
            "t": "STATE/STRATEGY/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
            "c": {
                "strategyId": "main",
                "freeCollateral": {
                    "0xb69e673309512a9d726f87304c6984054f87a93b": "113744.388472786563252582"
                },
                "frozenCollateral": {},
                "maxLeverage": 20,
                "frozen": false
            }
        },
        "eventTrigger": {
            "eventType": 5,
            "requestIndex": 10119,
            "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
            "collateralAddress": "0xb69e673309512a9d726f87304c6984054f87a93b",
            "strategyId": "main",
            "amount": "5000",
            "updateType": 0,
            "txHash": "0x2881c07c3f5da89feda42ba5529128d0ffb65c6bbe5459d28295d60748b35da5"
        }
    }
}
```

After the initial snapshot response, you will receive streaming `Update` messages with updates to the strategy leaf. A sample response is shown
on the right, with fields defined as follows:

| type                       | field                         | description                                                                                                                                                                                                                                                                                                                                                                                                                      |
| -------------------------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string                     | t                             | WebSocket message type, pertaining to the topic subscribed to                                                                                                                                                                                                                                                                                                                                                                    |
| string                     | e                             | Websocket message event, which in the case of an update, will be `Update`                                                                                                                                                                                                                                                                                                                                                        |
| dict                       | c                             | Websocket message content, containing the `Strategy` leaf update event data                                                                                                                                                                                                                                                                                                                                                      |
| dict                       | c.itemData                    | Contains the new `Strategy` leaf data                                                                                                                                                                                                                                                                                                                                                                                            |
| string                     | c.itemData.t                  | `Strategy` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/STRATEGY/<trader_address>/<abbrev_strategy_id_hash>`. `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`).                                                                        |
| string                     | c.itemData.c                  | `Strategy` leaf contents                                                                                                                                                                                                                                                                                                                                                                                                         |
| string                     | c.itemData.c.strategyId       | Cross-margined strategy identifier. Currently only `main` is supported.                                                                                                                                                                                                                                                                                                                                                          |
| dict<address_s, decimal_s> | c.itemData.c.freeCollateral   | Strategy's free collateral (available for trading) mapping from ERC-20 collateral token address to amount                                                                                                                                                                                                                                                                                                                        |
| dict<address_s, decimal_s> | c.itemData.c.frozenCollateral | Strategy's free collateral (available for withdrawal) mapping from ERC-20 collateral token address to amount                                                                                                                                                                                                                                                                                                                     |
| int                        | c.itemData.c.maxLeverage      | Strategy's maximum leverage setting                                                                                                                                                                                                                                                                                                                                                                                              |
| bool                       | c.itemData.c.frozen           | Flag indicating whether strategy is tokenized (`true`) or not (`false`). Currently only `false` is supported                                                                                                                                                                                                                                                                                                                     |
| dict                       | c.eventTrigger                | The transaction log event that triggered an update. Since this is an `Update` response, there will be a transaction log event that triggered it. The possible events that can result in a an update to a strategy are `StrategyUpdate`, `Withdraw`, `PartialFill` (`Fill`), `CompleteFill` (`Fill`), `Liquidation`, and `Funding`. Please check out each transaction type in these docs for additional information as necessary. |

### Position

You may subscribe to updates for any trader"s `Position` data.

#### Request

> Sample Position state subscription (JSON)

```json
// All positions
{
	"t": "Subscribe",
	"c": "STATE/POSITION/"
}

// All positions for a symbol
{
	"t": "Subscribe",
	"c": "STATE/POSITION/ETHPERP/"
}

// All positions for a symbol and trader address
{
	"t": "Subscribe",
	"c": "STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/"
}

// All positions for a symbol, trader address, and strategy id
{
	"t": "Subscribe",
	"c": "STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/"
}
```

| type   | field | description                                                                                                                                                                                                                                                                                                                                          |
| ------ | ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string | t     | WebSocket message type - must be `Subscribe`                                                                                                                                                                                                                                                                                                         |
| string | c     | Subscription topic of the format `STATE/POSITION/[<symbol>/][[<trader_address>/]][[[<abbrev_strategy_id_hash>/]]]`. `symbol` is of `string` type (e.g. `ETHPERP`), `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |

#### Response

##### Partial

> Sample Position leaf event partial response (JSON)

```json
{
    "t": "STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
    "e": "Partial",
    "c": {
        "itemData": [
            {
                "t": "STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
                "c": {
                    "side": "Short",
                    "balance": "10.13",
                    "avgEntryPrice": "1946.458727521116845949"
                }
            }
        ],
        "eventTrigger": null
    }
}
```

Upon subscription, you will first receive an initial snapshot of the current position data. A sample response is shown
on the right, with fields defined as follows:

| type      | field                         | description                                                                                                                                                                                                                                                                                                                                                                                      |
| --------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| string    | t                             | WebSocket message type, pertaining to the topic subscribed to                                                                                                                                                                                                                                                                                                                                    |
| string    | e                             | Websocket message event, which in the case of a snapshot, will be `Partial`                                                                                                                                                                                                                                                                                                                      |
| dict      | c                             | Websocket message content, containing a snapshot of `Position` leaf data based on the topic subscribed to                                                                                                                                                                                                                                                                                        |
| list      | c.itemData                    | Contains a list of the snapshotted `Position` leaf data                                                                                                                                                                                                                                                                                                                                          |
| dict      | c.itemData[n]                 | An individual `Position` leaf as part of the snapshot                                                                                                                                                                                                                                                                                                                                            |
| string    | c.itemData[n].t               | `Position` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/POSITION/<symbol>/<trader_address>/<abbrev_strategy_id_hash>`. `symbol` is of `string` type, `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |
| dict      | c.itemData[n].c               | `Position` leaf contents                                                                                                                                                                                                                                                                                                                                                                         |
| string    | c.itemData[n].c.side          | Position side (`Long`, `Short`)                                                                                                                                                                                                                                                                                                                                                                  |
| decimal_s | c.itemData[n].c.balance       | Position size                                                                                                                                                                                                                                                                                                                                                                                    |
| decimal_s | c.itemData[n].c.avgEntryPrice | Position average entry price                                                                                                                                                                                                                                                                                                                                                                     |
| dict      | c.eventTrigger                | The transaction log event that triggered an update. Since this is a `Partial` response, there is no update, thus will always be null.                                                                                                                                                                                                                                                            |

##### Update

> Sample Position leaf event update response (JSON)

```json
{
    "t": "STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
    "e": "Update",
    "c": {
        "itemData": {
            "t": "STATE/POSITION/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
            "c": {
                "side": "Short",
                "balance": "11.87",
                "avgEntryPrice": "1946.508753983901739634"
            }
        },
        "eventTrigger": {
            "eventType": 14,
            "requestIndex": 12289,
            "symbol": "ETHPERP",
            "takerOrderHash": "0x4a9a3e9b462ca1678df83819c9cd62d65306b967ce9879192a",
            "makerOrderHash": "0x4315e691b9b6e7046578823aa59ea0c8f4b96b5e48d2d07f2d",
            "makerOrderRemainingAmount": "6.94",
            "amount": "1.74",
            "price": "1946.8",
            "takerSide": "Bid",
            "makerOutcome": {
                "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
                "strategyId": "main",
                "fee": "0",
                "ddxFeeElection": false,
                "realizedPnl": "0",
                "newCollateral": "999155.746041827306396849",
                "newPositionAvgEntryPrice": "1946.508753983901739634",
                "newPositionBalance": "11.87",
                "positionSide": "Short"
            },
            "takerOutcome": {
                "traderAddress": "0x006ecbe1db9ef729cbe972c83fb886247691fb6beb",
                "strategyId": "main",
                "fee": "6.774864",
                "ddxFeeElection": false,
                "realizedPnl": "0",
                "newCollateral": "99994.506474276943985849",
                "newPositionAvgEntryPrice": "1945.889456858208289734",
                "newPositionBalance": "4.87",
                "positionSide": "Long"
            }
        }
    }
}
```

After the initial snapshot response, you will receive streaming `Update` messages with updates to the `Position` leaf. A sample response is shown
on the right, with fields defined as follows:

| type      | field                      | description                                                                                                                                                                                                                                                                                                                                                                                      |
| --------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| string    | t                          | WebSocket message type, pertaining to the topic subscribed to                                                                                                                                                                                                                                                                                                                                    |
| string    | e                          | Websocket message event, which in the case of an update, will be `Update`                                                                                                                                                                                                                                                                                                                        |
| dict      | c                          | Websocket message content containing the updated `Position` leaf data                                                                                                                                                                                                                                                                                                                            |
| list      | c.itemData                 | Contains a an entry for the `Position` leaf data                                                                                                                                                                                                                                                                                                                                                 |
| string    | c.itemData.t               | `Position` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/POSITION/<symbol>/<trader_address>/<abbrev_strategy_id_hash>`. `symbol` is of `string` type, `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |
| dict      | c.itemData.c               | `Position` leaf contents                                                                                                                                                                                                                                                                                                                                                                         |
| string    | c.itemData.c.side          | Position side (`Long`, `Short`)                                                                                                                                                                                                                                                                                                                                                                  |
| decimal_s | c.itemData.c.balance       | Position size                                                                                                                                                                                                                                                                                                                                                                                    |
| decimal_s | c.itemData.c.avgEntryPrice | Position average entry price                                                                                                                                                                                                                                                                                                                                                                     |
| dict      | c.eventTrigger             | The transaction log event that triggered an update. Since this is an `Update` response, there will be a transaction log event that triggered it. The possible events that can result in a an update to a strategy are `PartialFill` (`Fill`), `CompleteFill` (`Fill`), and `Liquidation`. Please check out each transaction type in these docs for additional information as necessary.          |

### Book order

You may subscribe to updates to orders in the order book.

#### Request

> Sample BookOrder state subscription (JSON)

```json
// All book orders
{
	"t": "Subscribe",
	"c": "STATE/BOOK_ORDER/"
}

// All book orders for a given symbol
{
	"t": "Subscribe",
	"c": "STATE/BOOK_ORDER/ETHPERP/"
}

// All book orders for a given symbol and trader
{
	"t": "Subscribe",
	"c": "STATE/BOOK_ORDER/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/"
}

// All book orders for a given symbol, trader, and strategy
{
	"t": "Subscribe",
	"c": "STATE/BOOK_ORDER/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/0x2576ebd1/"
}
```

| type   | field | description                                                                                                                                                                                                                                                                                                                                            |
| ------ | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| string | t     | WebSocket message type - must be `Subscribe`                                                                                                                                                                                                                                                                                                           |
| string | c     | Subscription topic of the format `STATE/BOOK_ORDER/[<symbol>/][[<trader_address>/]][[[<abbrev_strategy_id_hash>/]]]`. `symbol` is of `string` type (e.g. `ETHPERP`), `trader_address` should be an `address_pre_s` type and `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`). |

Like its REST counterpart, this is endpoint can be used for a variety of purposes. You can subscribe to `BookOrder` state updates to track:

-   L3 order book: An L3 order book is a granular, order-by-order view of the entire market for any given symbol (i.e. no aggregation). You can subscribe to a market"s order book using a topic such as `STATE/BOOK_ORDER/ETHPERP`, where `ETHPERP` is an example market you may be requesting. This will send an initial `Partial` response with all of the `BookOrder` leaf entries making up the order book, and subsequently, `Update` messages with any new, canceled, or updated orders in the book. You are welcome to process this data in any local order book data structure you see fit.
-   Open orders: You can obtain your open orders in a given strategy using a topic such as `STATE/BOOK_ORDER/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/0x2576ebd1/`, where `ETHPERP` is an example market, `0x006ecbe1db9ef729cbe972c83fb886247691fb6beb` is an example of your Ethereum address prefixed with the chain discriminant, and `0x2576ebd1` is an example of the first 4 bytes of the hash of your strategy"s name (in this case, `main`). This will send an initial `Partial` response with all of the `BookOrder` leaf entries corresponding to the open orders in your strategy, and subsequently, `Update` messages with any new, canceled, or updated orders of yours. You are welcome to process this data however you see fit.

#### Response

##### Partial

> Sample BookOrder partial response

```json
{
    "t": "STATE/BOOK_ORDER/ETHPERP/",
    "e": "Partial",
    "c": {
        "itemData": [
            {
                "t": "STATE/BOOK_ORDER/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/0x2576ebd1/0x8f9f1044d35dfef941e6af68cbaa183f0794d4dc4317206326/",
                "c": {
                    "side": "Short",
                    "amount": "1",
                    "price": "2057.9926",
                    "traderAddress": "0x006ecbe1db9ef729cbe972c83fb886247691fb6beb",
                    "strategyIdHash": "0x2576ebd1",
                    "bookOrdinal": 4
                }
            },
            {
                "t": "STATE/BOOK_ORDER/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/0xfe3094706dde23aeccba4639715d801c53b58807575efcf285/",
                "c": {
                    "side": "Bid",
                    "amount": "3.68",
                    "price": "2050.92",
                    "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
                    "strategyIdHash": "0x2576ebd1",
                    "bookOrdinal": 3
                }
            }
        ],
        "eventTrigger": null
    }
}
```

Upon subscription, you will first receive an initial snapshot of the orders you have subscribed to. A sample response is shown
on the right, with fields defined as follows:

| type      | field                          | description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| --------- | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                              | WebSocket message type, pertaining to the topic subscribed to                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| string    | e                              | Websocket message event, which in the case of a snapshot, will be `Partial`                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| dict      | c                              | Websocket message content, containing a snapshot of `BookOrder` leaf data based on the topic subscribed to                                                                                                                                                                                                                                                                                                                                                                                                       |
| list      | c.itemData                     | Contains a list of the snapshotted `BookOrder` leaf data                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| dict      | c.itemData[n]                  | An individual `BookOrder` leaf as part of the snapshot                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| string    | c.itemData[n].t                | `BookOrder` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/BOOK_ORDER/<symbol>/<trader_address>/<abbrev_strategy_id_hash>/<order_hash>/`. `symbol` is of `string` type, `trader_address` should be an `address_pre_s` type, `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`), and `order_hash` is of `bytes_s` type representing the first 25 bytes of the order"s unique hash. |
| dict      | c.itemData[n].c                | `BookOrder` leaf contents                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| string    | c.itemData[n].c.side           | Order side (`Bid`, `Ask`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| decimal_s | c.itemData[n].c.amount         | Order size                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| decimal_s | c.itemData[n].c.price          | Order price                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| address_s | c.itemData[n].c.traderAddress  | The trader"s Ethereum address responsible for this order                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| bytes_s   | c.itemData[n].c.strategyIdHash | The trader"s strategy identifier (first 4 bytes of the hash of the strategy, e.g. `main`) for which this order belongs                                                                                                                                                                                                                                                                                                                                                                                           |
| int       | c.itemData[n].c.bookOrdinal    | Numerical identifier signifying the order"s sequencing in the book, can be useful to arrange orders that are at the same level in priority                                                                                                                                                                                                                                                                                                                                                                       |
| dict      | c.eventTrigger                 | The transaction log event that triggered an update. Since this is a `Partial` response, there is no update, thus will always be null.                                                                                                                                                                                                                                                                                                                                                                            |

##### Update

> Sample book order leaf event update response (JSON)

```json
{
    "t": "STATE/BOOK_ORDER/ETHPERP/",
    "e": "Update",
    "c": {
        "itemData": {
            "t": "STATE/BOOK_ORDER/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/0x85f483c62fc5a2172db06a3dd322cd89a464bd31ad2408f6d6/",
            "c": {
                "side": "Bid",
                "amount": "14.37",
                "price": "1948.25",
                "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
                "strategyIdHash": "0x2576ebd1",
                "bookOrdinal": 6
            }
        },
        "eventTrigger": {
            "eventType": 2,
            "requestIndex": 13977,
            "symbol": "ETHPERP",
            "orderHash": "0x85f483c62fc5a2172db06a3dd322cd89a464bd31ad2408f6d6",
            "side": "Bid",
            "amount": "14.37",
            "price": "1948.25",
            "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
            "strategyId": "main",
            "bookOrdinal": 6
        }
    }
}
```

After the initial snapshot response, you will receive streaming `Update` messages with updates to the `BookOrder` leaf. A sample response is shown
on the right, with fields defined as follows:

| type      | field                       | description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| --------- | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                           | WebSocket message type, pertaining to the topic subscribed to                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| string    | e                           | Websocket message event, which in the case of a snapshot, will be `Partial`                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| dict      | c                           | Websocket message content, containing a snapshot of `BookOrder` leaf data based on the topic subscribed to                                                                                                                                                                                                                                                                                                                                                                                                       |
| dict      | c.itemData                  | Contains the updated `BookOrder` leaf data                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| string    | c.itemData.t                | `BookOrder` topic (corresponding to its key), containing identifying information. This key is of the format `STATE/BOOK_ORDER/<symbol>/<trader_address>/<abbrev_strategy_id_hash>/<order_hash>/`. `symbol` is of `string` type, `trader_address` should be an `address_pre_s` type, `abbrev_strategy_id_hash` is a `bytes_s` type representing the first 4-bytes of the hash of the strategy id (e.g. `main`), and `order_hash` is of `bytes_s` type representing the first 25 bytes of the order"s unique hash. |
| dict      | c.itemData.c                | `BookOrder` leaf contents                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| string    | c.itemData.c.side           | Order side (`Bid`, `Ask`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| decimal_s | c.itemData.c.amount         | Order size                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| decimal_s | c.itemData.c.price          | Order price                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| address_s | c.itemData.c.traderAddress  | The trader"s Ethereum address responsible for this order                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| bytes_s   | c.itemData.c.strategyIdHash | The trader"s strategy identifier (first 4 bytes of the hash of the strategy, e.g. `main`) for which this order belongs                                                                                                                                                                                                                                                                                                                                                                                           |
| int       | c.itemData.c.bookOrdinal    | Numerical identifier signifying the order"s sequencing in the book, can be useful to arrange orders that are at the same level in priority                                                                                                                                                                                                                                                                                                                                                                       |
| dict      | c.eventTrigger              | The transaction log event that triggered an update.                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

## Transaction log events

Transaction (oft-abbreviated tx) log events can be subscribed to when seeking to follow all the state-transitioning events taking place on DerivaDEX.

### Post

You may subscribe to new posted orders on DerivaDEX. A post transaction occurs when an order successfully places an order on the exchange"s order book (with some unmatched amount).

#### Request

> Sample post subscription (JSON)

```json
// All posts
{
	"t": "Subscribe",
	"c": "TX_LOG/POST/"
}

// All posts for a symbol
{
	"t": "Subscribe",
	"c": "TX_LOG/POST/ETHPERP/"
}

// All posts for a symbol and trader address
{
	"t": "Subscribe",
	"c": "TX_LOG/POST/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/"
}

// All posts for a symbol, trader address, and strategy id
{
	"t": "Subscribe",
	"c": "TX_LOG/POST/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/0x2576ebd1/"
}
```

| type   | field | description                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------ | ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string | t     | WebSocket message type - must be `Subscribe`                                                                                                                                                                                                                                                                                                                                                                           |
| string | c     | Subscription topic of the format `TX_LOG/POST/[<symbol>/][[<trader_address>/]][[[<abbrev_strategy_id_hash>/]]]`. `symbol` is of `string` type (e.g. `ETHPERP`), `trader_address` is of `address_pre_s` type corresponding to the trader"s Ethereum address and `abbrev_strategy_id_hash` is of `bytes_s` type corresponding to the first 4 bytes of the hash of the strategy to which this fill belongs (e.g. `main`). |

#### Response

##### Update

> Sample post transaction event update response (JSON)

```json
{
    "t": "TX_LOG/POST/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/",
    "e": "Update",
    "c": {
        "t": "TX_LOG/POST/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/0x2576ebd1/",
        "c": {
            "eventType": 2,
            "requestIndex": 52623,
            "symbol": "ETHPERP",
            "orderHash": "0x24b67dd08a9157b9431a087e892a2cbe1e3258034477af6e8b",
            "side": "Bid",
            "amount": "1.39",
            "price": "2030.25",
            "traderAddress": "0x006ecbe1db9ef729cbe972c83fb886247691fb6beb",
            "strategyId": "main",
            "bookOrdinal": 5920
        }
    }
}
```

After subscription, you will receive streaming `Update` messages with new strategy updates defined as follows:

| type      | field             | description                                                                                                                                               |
| --------- | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                 | WebSocket message type, pertaining to the topic subscribed to                                                                                             |
| string    | e                 | Websocket message event, which will be `Update`                                                                                                           |
| dict      | c                 | Websocket message content, containing the `Post` transaction event data                                                                                   |
| dict      | c.t               | `Post` event topic                                                                                                                                        |
| dict      | c.c               | `Post` event contents                                                                                                                                     |
| int       | c.c.eventType     | Enum value corresponding to this transaction log event type (`2=Post`)                                                                                    |
| int       | c.c.requestIndex  | Numerical sequencing identifier pertaining to this transaction. All transactions are processed in order of their `requestIndex`.                          |
| string    | c.c.symbol        | Market this order has been placed on (e.g. `ETHPERP`)                                                                                                     |
| bytes_s   | c.c.orderHash     | First 25 bytes of the order intent"s unique EIP-712 hash                                                                                                  |
| int       | c.c.side          | Side of order (`0=bid`, `1=ask`)                                                                                                                          |
| decimal_s | c.c.amount        | Size of order                                                                                                                                             |
| decimal_s | c.c.price         | Price the order has been placed at                                                                                                                        |
| address_s | c.c.traderAddress | Trader"s Ethereum address (prefixed with the chain discriminant) responsible for order placement                                                          |
| string    | c.c.strategyId    | Cross-margined strategy identifier this order belongs to                                                                                                  |
| int       | c.c.bookOrdinal   | Numerical identifier with sequencing implications. You can use this to arrange multiple orders at a given level in a local order book for prioritization. |

### Strategy update

You may subscribe to new strategy updates on DerivaDEX. A strategy update can take place when a trader successfully deposits or withdraws a supported collateral type on-chain.

#### Request

> Sample strategy update subscription (JSON)

```json
// All strategy updates
{
	"t": "Subscribe",
	"c": "TX_LOG/STRATEGY_UPDATE/"
}

// All strategy updates for a trader
{
	"t": "Subscribe",
	"c": "TX_LOG/STRATEGY_UPDATE/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/"
}

// All strategy updates for a trader and strategy id
{
	"t": "Subscribe",
	"c": "TX_LOG/STRATEGY_UPDATE/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/0x2576ebd1/"
}
```

| type   | field | description                                                                                                                                                                                                                                                                                                                                                         |
| ------ | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string | t     | WebSocket message type - must be `Subscribe`                                                                                                                                                                                                                                                                                                                        |
| string | c     | Subscription topic of the format `TX_LOG/STRATEGY_UPDATE/[<trader_address>/][[<abbrev_strategy_id_hash>/]]`. `trader_address` is of `address_pre_s` type corresponding to the trader"s Ethereum address and `abbrev_strategy_id_hash` is of `bytes_s` type corresponding to the first 4 bytes of the hash of the strategy to which this fill belongs (e.g. `main`). |

#### Response

##### Update

> Sample strategy update transaction event update response (JSON)

```json
{
    "t": "TX_LOG/STRATEGY_UPDATE/",
    "e": "Update",
    "c": {
        "t": "TX_LOG/STRATEGY_UPDATE/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/0x2576ebd1/",
        "c": {
            "eventType": 5,
            "requestIndex": 36006,
            "traderAddress": "0x006ecbe1db9ef729cbe972c83fb886247691fb6beb",
            "collateralAddress": "0xb69e673309512a9d726f87304c6984054f87a93b",
            "strategyId": "main",
            "amount": "1000",
            "updateType": 0,
            "txHash": "0x38d35506f2b75035c14e5ed9fab99edc1d6920bd6c06133b081dfe89c2cb90df"
        }
    }
}
```

After subscription, you will receive streaming `Update` messages with new strategy updates defined as follows:

| type      | field                 | description                                                                                                                      |
| --------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                     | WebSocket message type, pertaining to the topic subscribed to                                                                    |
| string    | e                     | Websocket message event, which will be `Update`                                                                                  |
| dict      | c                     | Websocket message content, containing the `StrategyUpdate` transaction event data                                                |
| dict      | c.t                   | `StrategyUpdate` event topic                                                                                                     |
| dict      | c.c                   | `StrategyUpdate` event contents                                                                                                  |
| int       | c.c.eventType         | Enum value corresponding to this transaction log event type (`5=StrategyUpdate`)                                                 |
| int       | c.c.requestIndex      | Numerical sequencing identifier pertaining to this transaction. All transactions are processed in order of their `requestIndex`. |
| address_s | c.c.traderAddress     | Trader"s Ethereum address (prefixed with the chain discriminant) to which this event belongs to                                  |
| address_s | c.c.collateralAddress | Ethereum address of the token being deposited/withdrawn                                                                          |
| string    | c.c.strategyId        | Cross-margined strategy identifier to which this event belongs                                                                   |
| decimal_s | c.c.amount            | Number of tokens being deposited/withdrawn                                                                                       |
| decimal_s | c.c.updateType        | Enum signifying the type of event (`0=deposit`, `1=withdraw`)                                                                    |
| decimal_s | c.c.txHash            | Ethereum transaction hash of the on-chain event                                                                                  |

### Price checkpoint

You may subscribe to new price updates on DerivaDEX.

#### Request

> Sample price checkpoint subscription (JSON)

```json
// All price checkpoints
{
	"t": "Subscribe",
	"c": "TX_LOG/PRICE_CHECKPOINT/"
}

// All price checkpoints for a market
{
	"t": "Subscribe",
	"c": "TX_LOG/PRICE_CHECKPOINT/ETHPERP/"
}
```

| type   | field | description                                                                                                                                              |
| ------ | ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string | t     | WebSocket message type - must be `Subscribe`                                                                                                             |
| string | c     | Subscription topic of the format `TX_LOG/PRICE_CHECKPOINT/[<symbol>/]`. `symbol` is of `string` type and corresponds to a given market (e.g. `ETHPERP`). |

#### Response

##### Update

> Sample price checkpoint transaction event update response (JSON)

```json
{
    "t": "TX_LOG/PRICE_CHECKPOINT/ETHPERP/",
    "e": "Update",
    "c": {
        "t": "TX_LOG/PRICE_CHECKPOINT/ETHPERP/",
        "c": {
            "eventType": 9,
            "requestIndex": 15957,
            "symbol": "ETHPERP",
            "ema": "0.562806061520606728",
            "indexPriceHash": "0xdf08ef420733ecf6055f279cbb296e70925d12ee4e04b945c9da42bfd5f0a3df",
            "indexPrice": "1950.64619047619047619"
        }
    }
}
```

After subscription, you will receive streaming `Update` messages with new price checkpoints defined as follows:

| type      | field              | description                                                                                                                      |
| --------- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                  | WebSocket message type, pertaining to the topic subscribed to                                                                    |
| string    | e                  | Websocket message event, which will be `Update`                                                                                  |
| dict      | c                  | Websocket message content, containing the `PriceCheckpoint` transaction event data                                               |
| dict      | c.t                | `PriceCheckpoint` event topic                                                                                                    |
| dict      | c.c                | `PriceCheckpoint` event contents                                                                                                 |
| int       | c.c.eventType      | Enum value corresponding to this transaction log event type (`9=PriceCheckpoint`)                                                |
| int       | c.c.requestIndex   | Numerical sequencing identifier pertaining to this transaction. All transactions are processed in order of their `requestIndex`. |
| string    | c.c.symbol         | Market for which this price checkpoint corresponds to                                                                            |
| decimal_s | c.c.ema            | Exponential moving average reflecting the spread between DerivaDEX"s order book and the underlying composite index price         |
| bytes32_s | c.c.indexPriceHash | Unique hash of the index price                                                                                                   |
| decimal_s | c.c.indexPrice     | Composite index price the perpetual is tracking                                                                                  |

### Fill

You may subscribe to new fill on DerivaDEX. A fill can take place as part of a `CompleteFill`, `PartialFill`, or `Liquidation` transaction.

#### Request

> Sample price checkpoint subscription (JSON)

```json
// All fills
{
	"t": "Subscribe",
	"c": "TX_LOG/FILL/"
}

// All fills for a market
{
	"t": "Subscribe",
	"c": "TX_LOG/FILL/ETHPERP/"
}

// All fills for a market and trader address
{
	"t": "Subscribe",
	"c": "TX_LOG/FILL/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/"
}

// All fills for a market, trader address, and strategy id
{
	"t": "Subscribe",
	"c": "TX_LOG/FILL/ETHPERP/0x006ecbe1db9ef729cbe972c83fb886247691fb6beb/0x2576ebd1/"
}
```

| type   | field | description                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------ | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string | t     | WebSocket message type - must be `Subscribe`                                                                                                                                                                                                                                                                                                                                                                                                              |
| string | c     | Subscription topic of the format `TX_LOG/FILL/[<symbol>/][[<trader_address>/]][[[<abbrev_strategy_id_hash>/]]]`. `symbol` is of `string` type and corresponds to a given market (e.g. `ETHPERP`), `trader_address` is of `address_pre_s` type corresponding to the trader"s Ethereum address, and `abbrev_strategy_id_hash` is of `bytes_s` type corresponding to the first 4 bytes of the hash of the strategy to which this fill belongs (e.g. `main`). |

#### Response

##### Update

> Sample price checkpoint transaction event update response (JSON)

```json
{
    "t": "TX_LOG/FILL/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
    "e": "Update",
    "c": {
        "t": "TX_LOG/FILL/ETHPERP/0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84/0x2576ebd1/",
        "c": {
            "eventType": 14,
            "requestIndex": 16298,
            "symbol": "ETHPERP",
            "takerOrderHash": "0xe8adb15982728639fa19a6d00ba6baa763cd7c7fbb339b16f5",
            "makerOrderHash": "0x20f5c6ddfae3950dd64b5915686342798bf57278cf8d793ce2",
            "makerOrderRemainingAmount": "10.08",
            "amount": "1.23",
            "price": "1951.6",
            "takerSide": "Ask",
            "makerOutcome": {
                "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
                "strategyId": "main",
                "fee": "0",
                "ddxFeeElection": false,
                "realizedPnl": "0",
                "newCollateral": "999149.891751616220046303",
                "newPositionAvgEntryPrice": "1951.6",
                "newPositionBalance": "1.23",
                "positionSide": "Long"
            },
            "takerOutcome": {
                "traderAddress": "0x006ecbe1db9ef729cbe972c83fb886247691fb6beb",
                "strategyId": "main",
                "fee": "4.800936",
                "ddxFeeElection": false,
                "realizedPnl": "0",
                "newCollateral": "99884.740517241403286802",
                "newPositionAvgEntryPrice": "1958.439229330850592231",
                "newPositionBalance": "8.23",
                "positionSide": "Short"
            }
        }
    }
}
```

After subscription, you will receive streaming `Update` messages with new price checkpoints defined as follows:

| type      | field                                     | description                                                                                                                      |
| --------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| string    | t                                         | WebSocket message type, pertaining to the topic subscribed to                                                                    |
| string    | e                                         | Websocket message event, which will be `Update`                                                                                  |
| dict      | c                                         | Websocket message content, containing the `Fill` transaction event data                                                          |
| dict      | c.t                                       | `Fill` event topic                                                                                                               |
| dict      | c.c                                       | `Fill` event contents                                                                                                            |
| int       | c.c.eventType                             | Enum value corresponding to this transaction log event type (`14=Fill`)                                                          |
| int       | c.c.requestIndex                          | Numerical sequencing identifier pertaining to this transaction. All transactions are processed in order of their `requestIndex`. |
| string    | c.c.symbol                                | Market for which this fill corresponds to                                                                                        |
| bytes_s   | c.c.takerOrderHash                        | First 25 bytes of the hash of the taker"s order intent                                                                           |
| bytes_s   | c.c.makerOrderHash                        | First 25 bytes of the hash of the maker"s order intent                                                                           |
| decimal_s | c.c.makerOrderRemainingAmount             | The remaining size on the maker order                                                                                            |
| decimal_s | c.c.amount                                | Size / number of contracts of the fill                                                                                           |
| decimal_s | c.c.price                                 | Price the fill took place                                                                                                        |
| string    | c.c.takerSide                             | Enum representing the side of the taker order (`Bid`, `Ask`)                                                                     |
| dict      | c.c.makerOutcome                          | Data containing the some relevant information pertaining to the maker in the trade                                               |
| address_s | c.c.makerOutcome.traderAddress            | Maker trader"s Ethereum address (prefixed with the chain discriminant)                                                           |
| string    | c.c.makerOutcome.strategyId               | Cross-margined strategy identifier for the maker trader                                                                          |
| decimal_s | c.c.makerOutcome.fee                      | Fees paid by the maker as a result of the trade                                                                                  |
| bool      | c.c.makerOutcome.ddxFeeElection           | Whether maker fees are being paid in DDX (`true`) or USDC (`false`)                                                              |
| decimal_s | c.c.makerOutcome.realizedPNL              | Realized profits and losses for the maker as a result of the trade (not including fees)                                          |
| decimal_s | c.c.makerOutcome.newCollateral            | New free collateral in maker trader"s strategy as a result of the trade                                                          |
| decimal_s | c.c.makerOutcome.newPositionAvgEntryPrice | New average entry price for maker"s position as a result of the trade                                                            |
| decimal_s | c.c.makerOutcome.newPositionBalance       | New size for maker"s position as a result of the trade                                                                           |
| string    | c.c.makerOutcome.positionSide             | New side for maker"s position (`Long`, `Short`) as a result of the trade                                                         |
| dict      | c.c.takerOutcome                          | Data containing the some relevant information pertaining to the taker in the trade                                               |
| address_s | c.c.takerOutcome.traderAddress            | Taker trader"s Ethereum address (prefixed with the chain discriminant)                                                           |
| string    | c.c.takerOutcome.strategyId               | Cross-margined strategy identifier for the taker trader                                                                          |
| decimal_s | c.c.takerOutcome.fee                      | Fees paid by the taker as a result of the trade                                                                                  |
| bool      | c.c.takerOutcome.ddxFeeElection           | Whether taker fees are being paid in DDX (`true`) or USDC (`false`)                                                              |
| decimal_s | c.c.takerOutcome.realizedPNL              | Realized profits and losses for the taker as a result of the trade (not including fees)                                          |
| decimal_s | c.c.takerOutcome.newCollateral            | New free collateral in taker trader"s strategy as a result of the trade                                                          |
| decimal_s | c.c.takerOutcome.newPositionAvgEntryPrice | New average entry price for taker"s position as a result of the trade                                                            |
| decimal_s | c.c.takerOutcome.newPositionBalance       | New size for taker"s position as a result of the trade                                                                           |
| int       | c.c.takerOutcome.positionSide             | New side for taker"s position (`Long`, `Short`) as a result of the trade                                                         |

Please keep in mind that due to the subscription topic methodology, in the scenario where
you don"t specify a particular `trader_address` and `abbrev_strategy_id_hash` (e.g. a topic of the format
`TX_LOG/FILL/ETHPERP/`), you will receive 2 `fill` event responses back on this WebSocket stream per `fill`, where
the two response topics correspond to both the maker and taker"s `trader_address` and `abbrev_strategy_id_hash`. You
will be able to determine that these `fill` events are in fact the same by looking at the `requestIndex` and/or the equivalent pair of `makerOutcome` and `takerOutcome` in both events.

# Collections API

The Collections API offers traders stats and data via a REST interface. All of this data can in theory
be reconstructed if you were to play the transaction log from genesis. However, for convenience,
you can fetch this data in this manner.

## Fills

### Request

> Sample fills REST query

```json
GET /stats/api/v1/account/{traderAddress}/fills?symbol={symbol}&limit={limit}&offset={offset}

e.g. /stats/api/v1/account/0xE834EC434DABA538cd1b9Fe1582052B880BD7e63/fills?symbol=ETHPERP&limit=500&offset=0
```

You may obtain the fills data for any trader address (NOT prefixed with the chain discriminant) with the following query parameter definition:

| type   | field  | description   |
| ------ | ------ | ------------- |
| string | symbol | Market symbol |
| string | limit  | Query limit   |
| string | offset | Query offset  |

### Response

> Sample fills REST response

```json
[
    {
        "fillId": "1",
        "epochId": "4",
        "amount": "0.600000000000000000",
        "symbol": "ETHPERP",
        "price": "2507.500000000000000000",
        "makerOrderHash": "0xff98f83e5ecbbae74ff7b4510421b545a4d07e37783a034cc4",
        "makerFee": "0",
        "makerRealizedPnl": "0",
        "takerOrderHash": "0xb3debba450d0f4699f17383e2d21921513e03bdd2c7caa6188",
        "takerFee": "3.009000000000000000",
        "takerRealizedPnl": "0",
        "reason": 0,
        "liquidationId": null,
        "timeValue": "4359",
        "createdAt": "2022-02-25T21:14:07.858Z",
        "updatedAt": "2022-02-25T21:14:07.858Z"
    }
]
```

# Appendix

The appendix covers some additional information for those of you wanting to dive a bit deeper into some of the
above topics.

## Types

The types labeled throughout this document in the request and response parameters may be familiar to those who have a background in Ethereum. In any case, please refer to the table below for additional information on the terminology used here. This reference in conjunction with the JSON samples should provide enough clarity:

| type          | description                                                                                                                                                          | example                                                              |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| string        | Literal of a sequence of characters surrounded by quotes                                                                                                             | "ETHPERP"                                                            |
| address_s     | 20-byte "0x"-prefixed hexadecimal string literal (i.e. 40 digits long) corresponding to an `address` ETH type                                                        | "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"                         |
| address_pre_s | 21-byte "0x"-prefixed hexadecimal string literal (i.e. 42 digits long) corresponding to an `address` ETH type prefixed with a single byte for the chain discriminant | "0x00A0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"                       |
| decimal       | Numerical value with up to, but no more than 18 decimals of precision                                                                                                | 10.031500000000000000                                                |
| decimal_s     | String representation of `decimal`                                                                                                                                   | "10.031500000000000000"                                              |
| bool          | Boolean value, either `true` or `false`                                                                                                                              | True                                                                 |
| bytes_s       | "0x"-prefixed hexadecimal string literal corresponding to a `bytes` ETH type                                                                                         | "0x00000001"                                                         |
| bytes32_s     | 32-byte "0x"-prefixed hexadecimal string literal (i.e. 64 digits long) corresponding to an `bytes32` ETH type                                                        | "0x0000000000000000000000000000000000000000000000000000000000000001" |
| timestamp_s_i | String representation of numerical UNIX timestamp representing the number of seconds since 1/1/1970                                                                  | "1616667513875"                                                      |
| timestamp_s   | String representation representing the ISO 8601 UTC timestamp                                                                                                        | "2021-03-25T10:38:09.503654"                                         |

## Making a deposit

> Deposit ABI (JSON)

```json
{
    "inputs": [
        {
            "internalType": "address",
            "name": "_collateralAddress",
            "type": "address"
        },
        {
            "internalType": "bytes32",
            "name": "_strategyId",
            "type": "bytes32"
        },
        {
            "internalType": "uint128",
            "name": "_amount",
            "type": "uint128"
        }
    ],
    "name": "deposit",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
}
```

> Smart contract deposit function (Solidity)

```solidity
function deposit(
    address _collateralAddress,
    bytes32 _strategyId,
    uint128 _amount
) external;
```

> Smart contract deposit function (Python)

```python
from web3 import Web3
from eth_abi import encode_single
import simplejson as json

# A Web3 instance
w3 = Web3(Web3.HTTPProvider("https://goerli.infura.io/v3/<your_api_key>"))

# Open up the ABI shown above saved at a file location, in this case `trader_abi.json` is the filename
with open("trader_abi.json") as f:
    trader_abi = json.load(f)

    # Create an trader_abi contract wrapper
    trader_contract = w3.eth.contract(address=Web3.toChecksumAddress("0x7b91a9ced1e4e5ce5855508ae3a5292d911b5b83"), abi=trader_abi)

    # Deposit 1000 USDC
    trader_contract.functions.deposit("0x0998ba4bd2ffbfaa894d8b616ae412813a79bcf9", encode_single("bytes32", "main".encode("utf8")), 1000000000).transact()
```

DerivaDEX is a decentralized exchange. As such, trading is non-custodial. Users are responsible for their own funds, which are deposited to the DerivaDEX smart contracts on Ethereum for trading.

To deposit funds on DerivaDEX, first ensure that you have created an Ethereum account. The deposit interaction is between a user and the DerivaDEX smart contracts. To be more explicit, you will not be utilizing the API to facilitate a deposit. The DerivaDEX Solidity smart contracts adhere to the [Diamond Standard](https://medium.com/derivadex/the-diamond-standard-a-new-paradigm-for-upgradeability-569121a08954). The `deposit` smart contract function you will need to use is located in the `Trader` facet, at the address of the main `DerivaDEX` proxy contract (`0x7b91a9ced1e4e5ce5855508ae3a5292d911b5b83`).

Note: Valid deposit collateral is curated by the smart contract and is managed by governance.

| field               | description                                                                                                                                                                                                                                |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| \_collateralAddress | ERC-20 token address deposited as collateral                                                                                                                                                                                               |
| \_strategyId        | Strategy ID (encoded as a 32-byte value) being funded. The `strategy` refers to the cross-margined bucket this trade belongs to. Currently, there is only the default `main` strategy, but support for multiple strategies is coming soon! |
| \_amount            | Amount deposited (be sure to use the grains format specific of the collateral token you are using (e.g. if you wanted to deposit 1 USDC, you would enter 1000000 since the USDC token contract has 6 decimal places)                       |

An example Python implementation is displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.

## Signatures & hashing

All [`commands`](#Commands) on the API must be signed. The payload you will sign using an Ethereum wallet client of your choice (e.g. ethers, web3.js, web3.py, etc.) will need to be hashed as per the EIP-712 standard. We **highly recommend** referring to the [original proposal](https://eips.ethereum.org/EIPS/eip-712) for full context, but in short, this standard introduced a framework by which users can securely sign typed structured data. This greatly improves the crypto UX as users can now sign data they see and understand as opposed to unreadable byte-strings. While these benefits may not be readily apparent for programmatic traders, you will need to conform to this standard regardless.

EIP-712 hashing consists of three critical components - a `header`, `domain` struct hash, and `message` struct hash.

### Header

> Sample EIP-191 header definition

```solidity
bytes2 eip191_header = 0x1901;
```

```python
eip191_header = b"\x19\x01"
```

The `header` is simply the byte-string `\x19\x01`. You are welcome to do this however you like, but it must adhere to the standard eventually, otherwise the signature will not ultimately successfully recover. Example Solidity and Python reference implementations are displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.

### Domain

> Domain separator for goerli. DO NOT modify these parameters.

```json
{
    "name": "DerivaDEX",
    "version": "1",
    "chainId": 5,
    "verifyingContract": "0x7b91a9ced1e4e5ce5855508ae3a5292d911b5b83"
}
```

> Sample computation of domain struct hash

```solidity
function compute_eip712_domain_struct_hash(string memory _name, string memory _version, uint256 _chainId, address _verifyingContract) public view returns (bytes32) {
    // keccak-256 hash of the encoded schema for the domain separator
    bytes32 domainSchemaHash = keccak256(abi.encodePacked(
        "EIP712Domain(",
        "string name,",
        "string version,",
        "uint256 chainId,",
        "address verifyingContract",
        ")"
    ));

    bytes32 domainStructHash = keccak256(abi.encodePacked(
        domainSchemaHash,
        keccak256(bytes(_name)),
        keccak256(bytes(_version)),
        _chainId,
        uint256(_verifyingContract)
    ));

    return domainStructHash;
}
```

```python
from eth_abi import encode_single
from eth_utils.crypto import keccak

def compute_eip712_domain_struct_hash(chain_id, verifying_contract):
    # keccak-256 hash of the encoded schema for the domain separator
    eip712_domain_separator_schema_hash = keccak(
        b"EIP712Domain("
        + b"string name,"
        + b"string version,"
        + b"uint256 chainId,"
        + b"address verifyingContract"
        + b")"
    )

    return keccak(
        eip712_domain_separator_schema_hash
        + keccak(b"DerivaDEX")
        + keccak(b"1")
        + encode_single("uint256", chain_id)
        + encode_single("address", verifying_contract)
    )
```

The `domain` is a mandatory field that allows for signature/hashing schemes on one dApp to be unique to itself from other dApps. All `commands` use the same `domain` specification. The parameters that comprise the domain are as follows:

| type    | field             | description                                 |
| ------- | ----------------- | ------------------------------------------- |
| string  | name              | Name of the dApp or protocol                |
| string  | version           | Current version of the signing domain       |
| uint256 | chainId           | EIP-155 chain ID                            |
| address | verifyingContract | DerivaDEX smart contract"s Ethereum address |

To generate the `domain` struct hash, you must perform a series of encodings and hashings of the schema and contents of the `domain` specfication. You are welcome to do this however you like, but it must adhere to the standard eventually, otherwise the signature will not ultimately successfully recover. Example Solidity and Python reference implementations are displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.

### Message

The `message` field varies depending on the typed data you are signing, and is illustrated on a case-by-case basis below.

#### Place order

> Sample computation of place order message struct hash

```solidity
function compute_eip712_message_struct_hash(bytes32 _symbol, bytes32 _strategy, uint256 _side, uint256 _orderType, bytes32 _nonce, uint256 _amount, uint256 _price, uint256 _stopPrice) public view returns (bytes32) {
    // keccak-256 hash of the encoded schema for the order params struct
    bytes32 eip712SchemaHash = keccak256(abi.encodePacked(
        "OrderParams(",
        "bytes32 symbol,",
        "bytes32 strategy,",
        "uint256 side,",
        "uint256 orderType,",
        "bytes32 nonce,",
        "uint256 amount,",
        "uint256 price,",
        "uint256 stopPrice",
        ")"
    ));

    bytes32 messageStructHash = keccak256(abi.encodePacked(
        eip712SchemaHash,
        _symbol,
        _strategy,
        _side,
        _orderType,
        _nonce,
        _amount,
        _price,
        _stopPrice
    ));

    return messageStructHash;
}
```

```python
from eth_abi import encode_single
from eth_utils.crypto import keccak
from ddx_python.decimal import Decimal

def compute_eip712_message_struct_hash(symbol: str, strategy: str, side: str, order_type: str, nonce: str, amount: Decimal, price: Decimal, stop_price: Decimal) -> bytes:
    # keccak-256 hash of the encoded schema for the place order command
    eip712_schema_hash = keccak(
        b"OrderParams("
        + b"bytes32 symbol,"
        + b"bytes32 strategy,"
        + b"uint256 side,"
        + b"uint256 orderType,"
        + b"bytes32 nonce,"
        + b"uint256 amount,"
        + b"uint256 price,"
        + b"uint256 stopPrice"
        + b")"
    )

    # Ensure decimal value has no more than 18 decimals of precision
    def round_to_unit(val: Decimal) -> Decimal:
        return val.quantize(18)

    # Scale up to DDX grains format (i.e. multiply by 1e18)
    def to_base_unit_amount(val: Decimal, decimals: int) -> int:
        return int(round_to_unit(val) * 10 ** decimals)

    # Convert order side string to int representation
    def order_side_to_int(order_side: str) -> int:
        if order_side == "Bid":
            return 0
        return 1

    # Convert order type string to int representation
    def order_type_to_int(order_type: str) -> int:
        if order_type == "Limit":
            return 0
        elif order_type == "Market":
            return 1
        return 2

    return keccak(
        eip712_schema_hash
        + len(symbol).to_bytes(1, byteorder="little")
        + encode_single("bytes32", symbol.encode("utf8"))[:-1]
        + len(strategy).to_bytes(1, byteorder="little")
        + encode_single("bytes32", strategy.encode("utf8"))[:-1]
        + encode_single("uint256", order_side_to_int(side))
        + encode_single("uint256", order_type_to_int(order_type))
        + encode_single("bytes32", bytes.fromhex(nonce[2:]))
        + encode_single("uint256", to_base_unit_amount(amount, 18))
        + encode_single("uint256", to_base_unit_amount(price, 18))
        + encode_single("uint256", to_base_unit_amount(stop_price, 18))
    )
```

The parameters that comprise the `message` for the `command` to place an order are as follows:

| type    | field     | description                                                                                                                                                                                                                                                                                                                                                                                           |
| ------- | --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bytes32 | symbol    | 32-byte encoding of the symbol length and symbol this order is for. The `symbol` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly.                                                                                                                                                                                               |
| bytes32 | strategy  | 32-byte encoding of the strategy length and strategy this order belongs to. The `strategy` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly. The `strategy` refers to the cross-margined bucket this trade belongs to. Currently, there is only the default `main` strategy, but support for multiple strategies is coming soon! |
| uint256 | side      | An integer value either `0` (Bid) or `1` (Ask)                                                                                                                                                                                                                                                                                                                                                        |
| uint256 | orderType | An integer value either `0` (Limit) or `1` (Market)                                                                                                                                                                                                                                                                                                                                                   |
| bytes32 | nonce     | 32-byte value (an incrementing numeric identifier that is unique per user for all time) resulting in uniqueness of order                                                                                                                                                                                                                                                                              |
| uint256 | amount    | Order amount (scaled up by 18 decimals; e.g. 2.5 => 2500000000000000000). The `amount` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.                                                                                                                                                                       |
| uint256 | price     | Order price (scaled up by 18 decimals; e.g. 2001.37 => 2001370000000000000000). The `price` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.                                                                                                                                                                  |
| uint256 | stopPrice | Stop price (scaled up by 18 decimals). The `stopPrice` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.                                                                                                                                                                                                       |

**Take special note of the transformations done on several fields as described in the table above. In other words, the order intent you submit to the API will have different representations for some fields than the order intent you hash.** You are welcome to do this however you like, but it must adhere to the standard eventually, otherwise the signature will not ultimately successfully recover. Example Solidity and Python reference implementations are displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.

#### Cancel order

> Sample computation of cancel order message struct hash

```solidity
function compute_eip712_message_struct_hash(bytes32 _symbol, bytes32 _orderHash, bytes32 _nonce) public view returns (bytes32) {
    // keccak-256 hash of the encoded schema for the cancel order params struct
    bytes32 eip712SchemaHash = keccak256(abi.encodePacked(
        "CancelOrderParams(",
        "bytes32 symbol,",
        "bytes32 orderHash,",
        "bytes32 nonce",
        ")"
    ));

    bytes32 messageStructHash = keccak256(abi.encodePacked(
        eip712SchemaHash,
        _symbol,
        _orderHash,
        _nonce,
    ));

    return messageStructHash;
}
```

```python
from eth_abi import encode_single
from eth_utils.crypto import keccak
from decimal import Decimal, ROUND_DOWN

def compute_eip712_message_struct_hash(symbol: str, order_hash: str, nonce: str) -> bytes:
    # keccak-256 hash of the encoded schema for the cancel order command
    eip712_schema_hash = keccak(
        b"CancelOrderParams("
        + b"bytes32 symbol,"
        + b"bytes32 orderHash,"
        + b"bytes32 nonce"
        + b")"
    )

    return keccak(
        eip712_schema_hash
        + len(symbol).to_bytes(1, byteorder="little")
        + encode_single("bytes32", symbol.encode("utf8"))[:-1]
        + encode_single("bytes32", bytes.fromhex(order_hash[2:]))
        + encode_single("bytes32", bytes.fromhex(nonce[2:]))
    )
```

The parameters that comprise the `message` for the `command` to cancel an order are as follows:

| type    | field     | description                                                                                                                                                                                             |
| ------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bytes32 | symbol    | 32-byte encoding of the symbol length and symbol this order is for. The `symbol` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly. |
| bytes32 | orderHash | 32-byte EIP-712 hash of the order at the time of placement                                                                                                                                              |
| bytes32 | nonce     | 32-byte value (an incrementing numeric identifier that is unique per user for all time) resulting in uniqueness of order cancellation                                                                   |

#### Cancel all

> Sample computation of cancel all message struct hash

```solidity
function compute_eip712_message_struct_hash(bytes32 _strategy, bytes32 _nonce) public view returns (bytes32) {
    // keccak-256 hash of the encoded schema for the cancel all params struct
    bytes32 eip712SchemaHash = keccak256(abi.encodePacked(
        "CancelAllParams(",
        "bytes32 strategy,",
        "bytes32 nonce",
        ")"
    ));

    bytes32 messageStructHash = keccak256(abi.encodePacked(
        eip712SchemaHash,
        _strategy,
        _nonce,
    ));

    return messageStructHash;
}
```

```python
from eth_abi import encode_single
from eth_utils.crypto import keccak
from decimal import Decimal, ROUND_DOWN

def compute_eip712_message_struct_hash(strategy: str, nonce: str) -> bytes:
    # keccak-256 hash of the encoded schema for the cancel all command
    eip712_schema_hash = keccak(
        b"CancelAllParams("
        + b"bytes32 strategy,"
        + b"bytes32 nonce"
        + b")"
    )

    return keccak(
        eip712_schema_hash
        + len(strategy).to_bytes(1, byteorder="little")
        + encode_single("bytes32", strategy.encode("utf8"))[:-1]
        + encode_single("bytes32", bytes.fromhex(nonce[2:]))
    )
```

The parameters that comprise the `message` for the `command` to cancel an order are as follows:

| type    | field    | description                                                                                                                                                                                                                                                                                                                                                                                           |
| ------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| bytes32 | strategy | 32-byte encoding of the strategy length and strategy this order belongs to. The `strategy` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly. The `strategy` refers to the cross-margined bucket this trade belongs to. Currently, there is only the default `main` strategy, but support for multiple strategies is coming soon! |     |
| bytes32 | nonce    | 32-byte value (an incrementing numeric identifier that is unique per user for all time) resulting in uniqueness of order cancellation                                                                                                                                                                                                                                                                 |

### Tying it all together

> Computing the final EIP-712 hash

```solidity
function compute_eip712_hash(bytes2 _eip191_header, bytes32 _domainStructHash, bytes32 _messageStructHash) public view returns (bytes32) {
    return keccak256(abi.encodePacked(
        _eip191_header,
        _domainStructHash,
        _messageStructHash
    ));
}
```

```python
from eth_utils.crypto import keccak

def compute_eip712_hash(eip191_header: bytes, eip712_domain_struct_hash: bytes, eip712_message_struct_hash: bytes) -> str:
    # Converting bytes result to a hexadecimal string
    return keccak(
        eip191_header
        + eip712_domain_struct_hash
        + eip712_message_struct_hash
    ).hex()
```

To derive the final EIP-712 hash of the typed data you will sign, you will need to `keccak256` hash the `header`, `eip712_domain_struct_hash`, and `eip712_message_struct_hash` (will vary depending on which `command` specifically you are sending). You are welcome to do this however you like, but it must adhere to the standard eventually, otherwise the signature will not ultimately successfully recover. Example Solidity and Python reference implementations are displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.

### Samples

**Please feel free to use these ground truth samples to validate your EIP-712 hashing implementation for correctness.** For the following samples, assume a `chainId = 42` and `verifyingContract = 0x7b91a9ced1e4e5ce5855508ae3a5292d911b5b83`.

#### Place order

The following sample order placement data results in an EIP-712 hash of: `0x57d38e5e8ebd21f773238437a61d09478a5c6699194cca6b233f54f0edc9c79e`.

| field     | value                                                                |
| --------- | -------------------------------------------------------------------- |
| symbol    | "ETHPERP"                                                            |
| strategy  | "main"                                                               |
| side      | "Bid"                                                                |
| orderType | "Limit"                                                              |
| nonce     | "0x3136343630373732323434323935363430303000000000000000000000000000" |
| amount    | 6.0                                                                  |
| price     | 2386.3                                                               |
| stopPrice | 0                                                                    |

#### Cancel order

The following sample cancellation data results in an EIP-712 hash of: `0xa8e33ee48cac9cf7baca677812254435047adb579bde631645c8283d6d7b5439`.

| field     | value                                                                |
| --------- | -------------------------------------------------------------------- |
| symbol    | "ETHPERP"                                                            |
| orderHash | "0x1c12a57d8a4a522f95680ae30413cd7f268850950e33a4b855"               |
| nonce     | "0x3136343630373933323431363338333130303000000000000000000000000000" |

#### Cancel all

The following sample cancellation data results in an EIP-712 hash of: `0x348b22fd655495b1c2e6284a037717c318ba35453556b3ed57a62c3ca72343e4`.

| field    | value                                                                |
| -------- | -------------------------------------------------------------------- |
| strategy | "main"                                                               |
| nonce    | "0x3136343630373933323431363338333130303000000000000000000000000000" |

## Encryption

> Sample encryption (JSON)

```json
// Sample unencrypted order placement request
{
	"t": "Order",
    "c": {
        "symbol": "ETHPERP",
        "strategy": "main",
        "side": "Ask",
        "orderType": "Limit",
        "nonce": "0x3136323737363235343138383430383030303000000000000000000000000000",
        "amount": 10.6,
        "price": 2472.1,
        "stopPrice": 0,
        "signature": "0x1b1f419961c742861a41396f14892ea4a665b7b89086637d37d53ec20364a7ef3aaf1e1472867f5a18fa3f27a2748ed16c603eccd13472cfd61d43df1c3ed22a1b"
    }
}

// Sample encrypted order placement request
"0x3bb4fec9257fb81c846c075fb3944151838c32ce41dadb585049fa7788ae591bb970ea295993d42b238e95595fde09d80d675aa960112533cf61ed4b91b9d77dd3f97d6ecf1cf809800854a11b43d27c5da51d69f550463bddeb471be94fb6a17de3e54cf99665be081ec7f8c24b7eebaef691baa27de167e448ed3a0facf6af0c4fec0e856f56515590c40479cf95f6e25918a42a60b4c15b4362614f91bfbd67eba6a6aad6de9edd45ba5fa7fc33e4473fb9e94a14f492c65bfecc08ff97d7c3126d6ce697aa955234e98ebb027fb042500122c299826267deb278b44a7dfc7f06fad6174f448f4fb41ce13e1003c2013b6de2e9a0bf3d9871658c84644fc41d7e5482bd4efad8370172e64b7220d2a2e596c9a5b3ce38997bae79200b3b62839e47bc6aa876128cb6d4430e18b1f51588d7741161e1a9734d0e203d725c9f4d7dae174a0e75fcdb3882c23590437fbc3330046b150bc90818e7b1c3a42571b62363343f2e46bc7cf6fcd0dfd2550ec83d453641d7a236a3a6677e806125b0c50246d5f26c2db3f61ebc78b955d5095d8b0cd7f4929f5bc2adf53c455b2170f546a09d2e11b463db6cc582a771c6aaad96c3c56123d5010643e584849e6f7cf998495e28b018b4c00bcb64a3aa2b7b7e0cb3faea33c7034887c86a15d5c0a01db2c8bc27a4b6ec2acc2e06ccabde4a64962ab3aec63f36"
```

> Sample encryption implementation (Python)

```python
def encrypt_with_nonce(msg: str) -> bytes:
    """
    Encrypt the JSON-stringified command contents

    Parameters
    ----------
    msg : str
        JSON-stringified command contents
    """

    network_public_key = PublicKey(
        bytes.fromhex(
            "0x0215cad31f884747d1c870458fea0c8e580044577be35276cbfbf6c212c813e722"[2:]
        )
    )
    my_secret_key = PrivateKey(get_random_bytes(32))
    my_public_key = my_secret_key.public_key
    shared_pub = network_public_key.multiply(my_secret_key.secret)
    keccak_256 = keccak.new(digest_bits=256)
    keccak_256.update(shared_pub.format())
    derived_key = keccak_256.digest()[:16]
    nonce = get_random_bytes(12)

    cipher = AES.new(derived_key, AES.MODE_GCM, nonce=nonce)
    encoded_message = msg.encode("utf8")
    ciphertext, tag = cipher.encrypt_and_digest(
        len(encoded_message).to_bytes(4, byteorder="big") + encoded_message
    )

    return ciphertext + tag + nonce + my_public_key.format()

```

DerivaDEX is a front-running resistant decentralized exchange, achieved with sending encrypted data that can only be
decrypted from inside the Operator"s trusted hardware. All `commands` must be encrypted using an AES-GCM128 scheme. The
encryption steps are as follows:

1. Generate n 16-byte (128-bit) ECDH shared key using the operator"s public key (`0x0215cad31f884747d1c870458fea0c8e580044577be35276cbfbf6c212c813e722`) and a 32-byte private key of your choosing (you can randomly generate this each time you are sending a `command`). Make sure you are using a `keccak256` ECDH key generation scheme, but since you need a 16-byte shared key, make sure you take only the first 16 bytes of the 32-byte key generated.

2. Generate a 12-byte random nonce

3. Setup the payload you want to encrypt, which is the bytes-encoded `command`"s contents prefixed with a 4-byte value indicating the length of this bytes-encoded `command`"s contents

4. Generate the ciphertext and MAC tag using AES-GCM128

5. Return the encrypted bytes in the format of [ciphertext][tag][nonce][client_public_key_compressed_format]

To validate your encryption implementation for correctness (assuming you are not using the DerivaDEX Python client library), please reference what follows and the display on the right:

-   Step 1: Assuming a random client secret key of `0xfa2751cb3d428e917226a46141f6d91448e73e07fba672dc2b80de492e6138da`, you should arrive at a ECDH shared key of `0xc0d7c330fe00314a35751f74260cec81`
-   Step 2: Assume a random nonce of `0x64a3aa2b7b7e0cb3faea33c7`
-   Step 3: Assuming a stringified unencrypted order placement message content of the following form `{"t": "Order", "c": {"traderAddress": "0xc10f3E99a255CC053b8E9A62c9C416485fdCd20b", "symbol": "ETHPERP", "strategy": "main", "side": "Ask", "orderType": "Limit", "nonce": "0x3136323737363235343138383430383030303000000000000000000000000000", "amount": 10.6, "price": 2472.1, "stopPrice": 0, "signature": "0x1b1f419961c742861a41396f14892ea4a665b7b89086637d37d53ec20364a7ef3aaf1e1472867f5a18fa3f27a2748ed16c603eccd13472cfd61d43df1c3ed22a1b"}}`, you should arrive at a bytes-encoded payload (prefixed with the length) to encrypt of: `0x000001b77b2274223a20224f72646572222c202263223a207b2274726164657241646472657373223a2022307863313066334539396132353543433035336238453941363263394334313634383566644364323062222c202273796d626f6c223a202245544850455250222c20227374726174656779223a20226d61696e222c202273696465223a202241736b222c20226f7264657254797065223a20224c696d6974222c20226e6f6e6365223a2022307833313336333233373337333633323335333433313338333833343330333833303330333033303030303030303030303030303030303030303030303030303030222c2022616d6f756e74223a2031302e362c20227072696365223a20323437322e312c202273746f705072696365223a20302c20227369676e6174757265223a2022307831623166343139393631633734323836316134313339366631343839326561346136363562376238393038363633376433376435336563323033363461376566336161663165313437323836376635613138666133663237613237343865643136633630336563636431333437326366643631643433646631633365643232613162227d7d`
-   Step 4: The above will generate a ciphertext of `0x3bb4fec9257fb81c846c075fb3944151838c32ce41dadb585049fa7788ae591bb970ea295993d42b238e95595fde09d80d675aa960112533cf61ed4b91b9d77dd3f97d6ecf1cf809800854a11b43d27c5da51d69f550463bddeb471be94fb6a17de3e54cf99665be081ec7f8c24b7eebaef691baa27de167e448ed3a0facf6af0c4fec0e856f56515590c40479cf95f6e25918a42a60b4c15b4362614f91bfbd67eba6a6aad6de9edd45ba5fa7fc33e4473fb9e94a14f492c65bfecc08ff97d7c3126d6ce697aa955234e98ebb027fb042500122c299826267deb278b44a7dfc7f06fad6174f448f4fb41ce13e1003c2013b6de2e9a0bf3d9871658c84644fc41d7e5482bd4efad8370172e64b7220d2a2e596c9a5b3ce38997bae79200b3b62839e47bc6aa876128cb6d4430e18b1f51588d7741161e1a9734d0e203d725c9f4d7dae174a0e75fcdb3882c23590437fbc3330046b150bc90818e7b1c3a42571b62363343f2e46bc7cf6fcd0dfd2550ec83d453641d7a236a3a6677e806125b0c50246d5f26c2db3f61ebc78b955d5095d8b0cd7f4929f5bc2adf53c455b2170f546a09d2e11b463db6cc582a771c6aaad96c3c56123d5010643e5` and a tag of `0x84849e6f7cf998495e28b018b4c00bcb`
-   Step 5: After deriving the public key from the secret key from Step 1 (`0x034887c86a15d5c0a01db2c8bc27a4b6ec2acc2e06ccabde4a64962ab3aec63f36`), you will arrive at the final encrypted message of: `0x3bb4fec9257fb81c846c075fb3944151838c32ce41dadb585049fa7788ae591bb970ea295993d42b238e95595fde09d80d675aa960112533cf61ed4b91b9d77dd3f97d6ecf1cf809800854a11b43d27c5da51d69f550463bddeb471be94fb6a17de3e54cf99665be081ec7f8c24b7eebaef691baa27de167e448ed3a0facf6af0c4fec0e856f56515590c40479cf95f6e25918a42a60b4c15b4362614f91bfbd67eba6a6aad6de9edd45ba5fa7fc33e4473fb9e94a14f492c65bfecc08ff97d7c3126d6ce697aa955234e98ebb027fb042500122c299826267deb278b44a7dfc7f06fad6174f448f4fb41ce13e1003c2013b6de2e9a0bf3d9871658c84644fc41d7e5482bd4efad8370172e64b7220d2a2e596c9a5b3ce38997bae79200b3b62839e47bc6aa876128cb6d4430e18b1f51588d7741161e1a9734d0e203d725c9f4d7dae174a0e75fcdb3882c23590437fbc3330046b150bc90818e7b1c3a42571b62363343f2e46bc7cf6fcd0dfd2550ec83d453641d7a236a3a6677e806125b0c50246d5f26c2db3f61ebc78b955d5095d8b0cd7f4929f5bc2adf53c455b2170f546a09d2e11b463db6cc582a771c6aaad96c3c56123d5010643e584849e6f7cf998495e28b018b4c00bcb64a3aa2b7b7e0cb3faea33c7034887c86a15d5c0a01db2c8bc27a4b6ec2acc2e06ccabde4a64962ab3aec63f36`

## Operator API <> Auditor

As mentioned in the [Auditor API](#auditor-api) section above, the Auditor is an abstraction that exposes a more convenient and familiar API for programmatic traders.
It"s connects more directly to the DerivaDEX Operator API"s transaction log WebSocket endpoint. This endpoint
provides a snapshot of the Sparse Merkle Tree and streaming transaction"s that transition
the state of this data store. Technically, to trade on DerivaDEX you won"t need to understand all
of these details as this has been abstracted away using the setup presented above. However, if you are
interested in 1) learning more about how the system works, 2) considering rewriting the Auditor, or 3) exploring connecting to the Operator API directly, you should read on to best understand
the data you will be working with.

### Sparse merkle tree (state)

DerivaDEX utilizes a [Sparse Merkle Tree (SMT)](https://medium.com/@kelvinfichter/whats-a-sparse-merkle-tree-acda70aeb837) in order to efficiently maintain the
state of the exchange at all times. Storing data on-chain (such as user"s balances and positions), something most
decentralized exchanges do, is very costly and one that is usually passed on to the user,
making exchange use prohibitively expensive to most. An SMT, however, allows the system to only
store a single root hash (the root of the tree) on-chain, while still allowing for any one to verify the integrity of the data (leaves) through
inclusion (or non-inclusion) proofs.

So what kind of data is stored and where in the SMT? The short answer is - everything.
This means any data pertaining to a trader (their account-level data, strategy-level data,
positions, and volume statistics), to a given market (the various open orders and the price feed information),
and to the system (the insurance fund capitalization) is stored as individual leaf entries in the SMT.
The DerivaDEX SMT consists of 2^256 possible leaves, each uniquely located at a specific leaf location
(as indicated by its `key`, which is a 32-byte value). As you might surmise, the _vast_ majority of the SMT
will be empty (2^256 is very, _very_, _**very**_ large number), but the data that does exist can exist at a location
anywhere within these large bounds. For efficient querying and data management
(while maintaining practical collision-resistance), the various types of leaf items are
prefixed to reside in the same general vicinity as one another. Each leaf item type is
described in detail below. The full set of leaf types can be seen in the following table,
along with their corresponding numeric discriminants.

| Item                      | Discriminant |
| ------------------------- | ------------ |
| Empty                     | 0            |
| Trader                    | 1            |
| Strategy                  | 2            |
| Position                  | 3            |
| BookOrder                 | 4            |
| Price                     | 5            |
| InsuranceFund             | 6            |
| Stats                     | 7            |
| Signer                    | 8            |
| Specs                     | 9            |
| InsuranceFundContribution | 10           |
| FeePool                   | 11           |
| EpochMetadata             | 12           |

Each item type is described in detail below.

#### Trader

A `Trader` contains information pertaining to a trader"s free and frozen DDX balances,
and referral address.

##### Key encoding / decoding

> Key encoding / decoding (Python)

```python
from eth_abi.utils.padding import zpad32_right

def encode_key(trader_address: str):
    # ItemType.TRADER == 1
    return zpad32_right(
        ItemType.TRADER.to_bytes(1, byteorder="little")
        + bytes.fromhex(trader_address[2:])
    )

def decode_key(trader_key: bytes):
    # trader_address
    return f"0x{trader_key[1:22].hex()}"
```

The location of a `Trader` leaf is determined by its key, which is encoded as
follows:

| Bytes    | Value                                                          |
| -------- | -------------------------------------------------------------- |
| 0        | Trader discriminant                                            |
| [1, 21]  | Trader's Ethereum address prefixed with the chain discriminant |
| [22, 31] | Zero-padding                                                   |

The following sample `Trader` materials generates the following encoded key: `0x0100603699848c84529987e14ba32c8a66def67e9ece00000000000000000000`.

| field          | value                                          |
| -------------- | ---------------------------------------------- |
| trader_address | "0x00603699848c84529987E14Ba32C8a66DEF67E9eCE" |

##### Value definition

> Value encoding / decoding (Python)

```python
from ddx_python.decimal import Decimal
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(6)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, free_ddx_balance: Decimal, frozen_ddx_balance: Decimal, referral_address: str):
    # Trader item discriminant
    item_type = 1

    # Scale collateral amounts to grains
    return encode_single(
        "(uint8,(uint128,uint128,address,bool))",
        [
            item_type,
            [
                to_base_unit_amount(free_ddx_balance, 6),
                to_base_unit_amount(frozen_ddx_balance, 6),
                referral_address,
            ],
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (
        item_type,
        (free_ddx_balance, frozen_ddx_balance, referral_address),
    ) = decode_single(
        "(uint8,(uint128,uint128,address,bool))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale collateral amounts from grains
    return (
        to_unit_amount(free_ddx_balance, 6),
        to_unit_amount(frozen_ddx_balance, 6),
        referral_address,
    )
```

A `Trader` leaf holds the following data:

| type      | field              | description                                                                                  |
| --------- | ------------------ | -------------------------------------------------------------------------------------------- |
| decimal   | free_ddx_balance   | DDX collateral available for staking/fees                                                    |
| decimal   | frozen_ddx_balance | DDX collateral available for on-chain withdrawal                                             |
| address_s | referral_address   | Referral address pertaining to the Ethereum address who referred this trader (if applicable) |
| bool      | pay_fees_in_ddx    | Whether trader has opted to pay fees in DDX by default or not                                |

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint128,uint128,address,bool))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the grains conversion for certain
variables, is shown on the right.

The following sample `Trader` materials generates the following ABI-encoded value: `0x0000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000000000000000003b9aca000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a8dda8d7f5310e4a9e24f8eba77e091ac264f8720000000000000000000000000000000000000000000000000000000000000000`.

| field              | value                                        |
| ------------------ | -------------------------------------------- |
| free_ddx_balance   | 1000                                         |
| frozen_ddx_balance | 0                                            |
| referral_address   | "0xA8dDa8d7F5310E4A9E24F8eBA77E091Ac264f872" |
| pay_fees_in_ddx    | "0xA8dDa8d7F5310E4A9E24F8eBA77E091Ac264f872" |

#### Strategy

A `Strategy` contains information pertaining to a trader"s cross-margined strategy, such as their free and frozen collaterals and max leverage.

##### Key encoding / decoding

> Key encoding / decoding (Python)

```python
from eth_abi.utils.padding import zpad32_right, encode_single, decode_single
from web3.auto import w3

def generate_strategy_id_hash(strategy_id: str) -> bytes:
    # Get the first 4 bytes of the hash of the strategy id
    return w3.keccak(
        len(strategy_id).to_bytes(1, byteorder="little")
        + encode_single("bytes32", strategy_id.encode("utf8"))[:-1]
    )[:4]

def encode_key(trader_address: str, strategy_id: str, chain_discriminant: int):
    # ItemType.STRATEGY == 2
    return zpad32_right(
            ItemType.STRATEGY.to_bytes(1, byteorder="little")
            + bytes.fromhex(trader_address[2:])
            + generate_strategy_id_hash(strategy_id)
        )

def decode_key(strategy_key: bytes):
    # trader_address, strategy_id_hash
    return (
            f"0x{strategy_key[1:22].hex()}",
            f"0x{strategy_key[22:26].hex()}",
        )
```

The location of a `Strategy` leaf is determined by its key, which is encoded as
follows:

| Bytes    | Value                                                          |
| -------- | -------------------------------------------------------------- |
| 0        | Strategy discriminant                                          |
| [1, 21]  | Trader"s Ethereum address prefixed with the chain discriminant |
| [22, 25] | Abbreviated hash of strategy ID                                |
| [26, 25] | Zero-padding                                                   |

The following sample `Strategy` materials generates the following encoded key: `0x0200603699848c84529987e14ba32c8a66def67e9ece2576ebd1000000000000`.

| field          | value                                          |
| -------------- | ---------------------------------------------- |
| trader_address | "0x00603699848c84529987E14Ba32C8a66DEF67E9eCE" |
| strategy_id    | "main"                                         |

##### Value definition

> Value encoding / decoding (Python)

```python
from typing import Dict
from ddx_python.decimal import Decimal
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(6)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_base_unit_amount_list(vals, decimals):
    return [to_base_unit_amount(val, decimals) for val in vals]

def to_unit_amount(val: int, decimals: int) -> Decimal:
    return Decimal(str(val)) / 10 ** decimals

def to_unit_amount_list(vals: List[int], decimals: int):
    return [to_unit_amount(val, decimals) for val in vals]

def to_adjusted_encoding_for_negative_val(val: int):
    return 16 ** 32 + abs(val) if val < 0 else val

def to_adjusted_encoding_for_negative_val_list(vals: List[int]):
    return [to_adjusted_encoding_for_negative_val(val) for val in vals]

def from_adjusted_encoding_for_negative_val(val: int):
    return val if val < 16 ** 32 else -(val - 16 ** 32)

def abi_encoded_value(self, strategy_id: str, free_collateral: Dict[str, Decimal], frozen_collateral: Dict[str, Decimal], max_leverage: int, frozen: bool):
    # Strategy item discriminant
    item_type = 2

    # Scale collateral amounts to grains
    return encode_single(
        "((uint8,(bytes32,(address[],uint256[]),(address[],uint128[]),uint64,bool)))",
        [
            [
                item_type,
                [
                    len(strategy_id).to_bytes(1, byteorder="little")
                    + encode_single("bytes32", strategy_id.encode("utf8"))[
                        :-1
                    ],
                    [
                        list(free_collateral.keys()),
                        to_adjusted_encoding_for_negative_val_list(
                            to_base_unit_amount_list(
                                list(free_collateral.values()), 6
                            )
                        ),
                    ],
                    [
                        list(frozen_collateral.keys()),
                        to_base_unit_amount_list(
                            list(frozen_collateral.values()), 6
                        ),
                    ],
                    max_leverage,
                    frozen,
                ],
            ]
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (
        (
            item_type,
            (
                strategy_id,
                (free_collateral_tokens, free_collateral_amounts),
                (frozen_collateral_tokens, frozen_collateral_amounts),
                max_leverage,
                frozen,
            ),
        ),
    ) = decode_single(
        "((uint8,(bytes32,(address[],uint256[]),(address[],uint128[]),uint64,bool)))",
        w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale collateral amounts from grains
    return (
        strategy_id[1 : 1 + strategy_id[0]].decode("utf8"),
        {
            k: to_unit_amount(from_adjusted_encoding_for_negative_val(v), 6)
            for k, v in zip(
                list(free_collateral_tokens), list(free_collateral_amounts)
            )
        },
        {
            k: to_unit_amount(v, 6)
            for k, v in zip(
                list(frozen_collateral_tokens), list(frozen_collateral_amounts)
            )
        },
        max_leverage,
        frozen,
    )
```

A `Strategy` leaf holds the following data:

| type                     | field             | description                                                       |
| ------------------------ | ----------------- | ----------------------------------------------------------------- |
| string                   | strategy_id       | Identifier for strategy (e.g. "main")                             |
| dict<address_s, decimal> | free_collateral   | Mapping of collateral address to free collateral amount           |
| dict<address_s, decimal> | frozen_collateral | Mapping of collateral address to frozen collateral amount         |
| int                      | max_leverage      | Maximum leverage strategy can take                                |
| bool                     | frozen            | Whether the strategy is frozen or not (relevant for tokenization) |

These contents are always stored in the tree in ABI-encoded form: `((uint8,(bytes32,(address[],uint128[]),(address[],uint128[]),uint64,bool)))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the grains conversion for certain
variables, is shown on the right.

The following sample `Strategy` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000040046d61696e00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000016000000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000001000000000000000000000000b69e673309512a9d726f87304c6984054f87a93b00000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000002e8f3487400000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000`.

| field             | value                                                     |
| ----------------- | --------------------------------------------------------- |
| strategy_id       | "main"                                                    |
| free_collateral   | {"0xb69e673309512a9d726f87304c6984054f87a93b": 199971.08} |
| frozen_collateral | {}                                                        |
| max_leverage      | 3                                                         |
| frozen            | False                                                     |

#### Position

A `Position` contains information pertaining to an open position.

##### Key encoding / decoding

> Key encoding / decoding (Python)

```python
from eth_abi.utils.padding import zpad32_right, encode_single, decode_single
from web3.auto import w3

def generate_strategy_id_hash(strategy_id: str) -> bytes:
    # Get the first 4 bytes of the hash of the strategy id
    return w3.keccak(
        len(strategy_id).to_bytes(1, byteorder="little")
        + encode_single("bytes32", strategy_id.encode("utf8"))[:-1]
    )[:4]

def pack_bytes(text: str):
    charset = "0ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    symbol_bits = ""
    for letter in text:
        for i, char in enumerate(charset):
            if char == letter:
                bits = format(i, "08b")[::-1]
                symbol_bits += bits[:5]

    symbol_bytes = int(symbol_bits[::-1], 2).to_bytes(
        (len(symbol_bits) + 7) // 8, byteorder="little"
    )
    return zpad_right(symbol_bytes, 6)

def unpack_bytes(packed_text: bytes):
    charset = "0ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    symbol_bits = format(int.from_bytes(packed_text, "little"), "030b")[::-1]
    symbol_char_bit_chunks = [
        symbol_bits[i : i + 5] for i in range(0, len(symbol_bits), 5)
    ]
    symbol = ""
    for symbol_char_bit_chunk in symbol_char_bit_chunks:
        reversed_bit_chunk = symbol_char_bit_chunk[::-1]
        char_index = int(reversed_bit_chunk, 2)
        symbol += charset[char_index]
    return symbol

def encode_key(trader_address: str, strategy_id: str, symbol: str, chain_discriminant: int):
    # ItemType.POSITION == 3
    return (
            ItemType.POSITION.to_bytes(1, byteorder="little")
            + pack_bytes(symbol)
            + bytes.fromhex(trader_address[2:])
            + generate_strategy_id_hash(strategy_id)
        )

def decode_key(position_key: bytes):
    # symbol, trader_address, strategy_id_hash
    return (
            unpack_bytes(position_key[1:7]),
            f"0x{position_key[7:28].hex()}",
            f"0x{position_key[28:].hex()}",
        )
```

The location of a `Position` leaf is determined by its key, which is encoded as
follows:

| Bytes    | Value                                                          |
| -------- | -------------------------------------------------------------- |
| 0        | Position discriminant                                          |
| [1, 6]   | Symbol (5-bit encoded/packed)                                  |
| [7, 27]  | Trader"s Ethereum address prefixed with the chain discriminant |
| [28, 31] | Abbreviated hash of strategy ID                                |

The following sample `Position` materials generates the following encoded key: `0x0385225824040000603699848c84529987e14ba32c8a66def67e9ece2576ebd1`.

| field          | value                                        |
| -------------- | -------------------------------------------- |
| trader_address | "0x603699848c84529987E14Ba32C8a66DEF67E9eCE" |
| strategy_id    | "main"                                       |
| symbol         | "ETHPERP"                                    |

##### Value definition

> Value encoding / decoding (Python)

```python
from typing import Dict
from ddx_python.decimal import Decimal
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(6)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(side: int, balance: Decimal, avg_entry_price: Decimal):
    # Position item discriminant
    item_type = 3

    # Scale balance and average entry price grains
    return encode_single(
        "(uint8,(uint8,uint128,uint128))",
        [
            item_type,
            [
                side,
                to_base_unit_amount(balance, 6),
                to_base_unit_amount(avg_entry_price, 6),
            ],
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (item_type, (side, balance, avg_entry_price)) = decode_single(
        "(uint8,(uint8,uint128,uint128))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale balance and average entry price from DDX grains
    return (
        side, to_unit_amount(balance, 6), to_unit_amount(avg_entry_price, 6)
    )
```

A `Position` leaf holds the following data:

| type    | field               | description                            |
| ------- | ------------------- | -------------------------------------- |
| int     | side                | Side of position (`Long=1`, `Short=2`) |
| decimal | balance             | Size of the position                   |
| decimal | average_entry_price | Average entry price of the position    |

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint8,uint128,uint128))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the grains conversion for certain
variables, is shown on the right.

The following sample `Position` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000068155a43676e0000000000000000000000000000000000000000000000000000d4eff354906660000`.

| field           | value |
| --------------- | ----- |
| side            | 1     |
| balance         | 120   |
| avg_entry_price | 245.5 |

#### Book order

A `BookOrder` contains information pertaining to a maker order in the order book.

##### Key encoding / decoding

> Key encoding / decoding (Python)

```python
from eth_abi.utils.padding import zpad32_right, encode_single, decode_single
from web3.auto import w3

def generate_strategy_id_hash(strategy_id: str) -> bytes:
    # Get the first 4 bytes of the hash of the strategy id
    return w3.keccak(
        len(strategy_id).to_bytes(1, byteorder="little")
        + encode_single("bytes32", strategy_id.encode("utf8"))[:-1]
    )[:4]

def pack_bytes(text: str):
    charset = "0ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    symbol_bits = ""
    for letter in text:
        for i, char in enumerate(charset):
            if char == letter:
                bits = format(i, "08b")[::-1]
                symbol_bits += bits[:5]

    symbol_bytes = int(symbol_bits[::-1], 2).to_bytes(
        (len(symbol_bits) + 7) // 8, byteorder="little"
    )
    return zpad_right(symbol_bytes, 6)

def unpack_bytes(packed_text: bytes):
    charset = "0ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    symbol_bits = format(int.from_bytes(packed_text, "little"), "030b")[::-1]
    symbol_char_bit_chunks = [
        symbol_bits[i : i + 5] for i in range(0, len(symbol_bits), 5)
    ]
    symbol = ""
    for symbol_char_bit_chunk in symbol_char_bit_chunks:
        reversed_bit_chunk = symbol_char_bit_chunk[::-1]
        char_index = int(reversed_bit_chunk, 2)
        symbol += charset[char_index]
    return symbol

def encode_key(symbol: str, order_hash: str):
    # ItemType.BOOK_ORDER == 3
    return (
            ItemType.BOOK_ORDER.to_bytes(1, byteorder="little")
            + pack_bytes(symbol)
            + bytes.fromhex(order_hash[2:52])
        )

def decode_key(book_order_key: bytes):
    # symbol, first 25 bytes of order_hash
    return (
            unpack_bytes(book_order_key[1:7]),
            f"0x{book_order_key[7:].hex()}",
        )
```

The location of a `BookOrder` leaf is determined by its key, which is encoded as
follows:

| Bytes   | Value                                 |
| ------- | ------------------------------------- |
| 0       | Book order discriminant               |
| [1, 6]  | Symbol (5-bit encoded/packed)         |
| [7, 31] | First 25 bytes of order"s unique hash |

The following sample `BookOrder` materials generates the following encoded key: `0x048522582404009fcf480aa9606f5bbb64ab36e642566bd893f715b346df244e`.

| field      | value                                                                |
| ---------- | -------------------------------------------------------------------- |
| symbol     | "ETHPERP"                                                            |
| order_hash | "0x9fcf480aa9606f5bbb64ab36e642566bd893f715b346df244eABABABABABABAB" |

##### Value definition

> Value encoding / decoding (Python)

```python
from typing import Dict
from ddx_python.decimal import Decimal
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(6)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, side: int, amount: Decimal, price: Decimal, trader_address: str, strategy_id_hash: str, book_ordinal: int, time_value: int):
    # BookOrder item discriminant
    item_type = 4

    # Scale amount and price to grains
    return encode_single(
        "(uint8,(uint8,uint128,uint128,bytes21,bytes32,uint64,uint64))",
        [
            item_type,
            [
                side,
                to_base_unit_amount(amount, 6),
                to_base_unit_amount(price, 6),
                bytes.fromhex(trader_address[2:]),
                bytes.fromhex(strategy_id_hash[2:]),
                book_ordinal,
                time_value,
            ],
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (
        item_type,
        (side, amount, price, trader_address, strategy_id_hash, book_ordinal, time_value),
    ) = decode_single(
        "(uint8,(uint8,uint128,uint128,bytes21,bytes32,uint64,uint64))",
        w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale amount and price from grains
    return (
        side,
        to_unit_amount(amount, 6),
        to_unit_amount(price, 6),
        f"0x{trader_address.hex()}",
        f"0x{strategy_id_hash[:4].hex()}",
        book_ordinal,
        time_value,
    )
```

A `BookOrder` leaf holds the following data:

| type    | field            | description                                                                                                |
| ------- | ---------------- | ---------------------------------------------------------------------------------------------------------- |
| str     | side             | Side of order (`Bid`, `Ask`)                                                                               |
| decimal | amount           | Amount/size of order                                                                                       |
| decimal | price            | Price the order has been placed at                                                                         |
| str     | trader_address   | The order creator"s Ethereum address prefixed with the chain discriminant                                  |
| str     | strategy_id_hash | First 4 bytes of strategy ID hash this order belongs to                                                    |
| int     | book_ordinal     | Numerical sequencing identifier for a BookOrder, which can be used to sort orders at any given price level |
| int     | time_value       | Time value of order placement since genesis (s)                                                            |

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint8,uint128,uint128,address,bytes32,uint64))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the grains conversion for certain
variables, is shown on the right.

The following sample `BookOrder` materials generates the following ABI-encoded value: `00000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001158e460913d0000000000000000000000000000000000000000000000000000d8d726b7177a80000603699848c84529987e14ba32c8a66def67e9ece0000000000000000000000002576ebd100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000300000000000000000000000000000000000000000000000000000000000003e8`

| field            | value                                        |
| ---------------- | -------------------------------------------- |
| side             | "Bid"                                        |
| amount           | 20                                           |
| price            | 250                                          |
| trader_address   | "0x603699848c84529987E14Ba32C8a66DEF67E9eCE" |
| strategy_id_hash | "0x2576ebd1"                                 |
| book_ordinal     | 3                                            |
| book_ordinal     | 1000                                         |

#### Price

A `Price` contains information pertaining to a market"s price checkpoint.

##### Key encoding / decoding

> Key encoding / decoding (Python)

```python
from eth_abi.utils.padding import zpad32_right, encode_single, decode_single
from web3.auto import w3

def pack_bytes(text: str):
    charset = "0ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    symbol_bits = ""
    for letter in text:
        for i, char in enumerate(charset):
            if char == letter:
                bits = format(i, "08b")[::-1]
                symbol_bits += bits[:5]

    symbol_bytes = int(symbol_bits[::-1], 2).to_bytes(
        (len(symbol_bits) + 7) // 8, byteorder="little"
    )
    return zpad_right(symbol_bytes, 6)

def unpack_bytes(packed_text: bytes):
    charset = "0ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    symbol_bits = format(int.from_bytes(packed_text, "little"), "030b")[::-1]
    symbol_char_bit_chunks = [
        symbol_bits[i : i + 5] for i in range(0, len(symbol_bits), 5)
    ]
    symbol = ""
    for symbol_char_bit_chunk in symbol_char_bit_chunks:
        reversed_bit_chunk = symbol_char_bit_chunk[::-1]
        char_index = int(reversed_bit_chunk, 2)
        symbol += charset[char_index]
    return symbol

def encode_key(symbol: str, index_price_hash: str):
    # ItemType.PRICE == 5
    return zpad32_right(
            ItemType.PRICE.to_bytes(1, byteorder="little")
            + pack_bytes(symbol)
            + bytes.fromhex(index_price_hash[2:])
        )

def decode_key(price_key: bytes):
    # symbol, index price hash
    return (
            unpack_bytes(price_key[1:7]),
            f"0x{leaf_key[7:].hex()}",
        )
```

The location of a `Price` leaf is determined by its key, which is encoded as
follows:

| Bytes   | Value                                 |
| ------- | ------------------------------------- |
| 0       | Price discriminant                    |
| [1, 6]  | Symbol (5-bit encoded/packed)         |
| [7, 31] | 25 bytes of index price"s unique hash |

The following sample `Price` materials generates the following encoded key: `0x0585225824040023c2b28e94b9dfcb8630a369fed134cdb6689dcb2528578779`.

| field            | value                                                                |
| ---------------- | -------------------------------------------------------------------- |
| symbol           | "ETHPERP"                                                            |
| index_price_hash | "0x23c2b28e94b9dfcb8630a369fed134cdb6689dcb2528578779ABABABABABABAB" |

##### Value definition

> Value encoding / decoding (Python)

```python
from typing import Dict
from ddx_python.decimal import Decimal
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(6)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, index_price: Decimal, ema: Decimal, ordinal: int):
    # Price item discriminant
    item_type = 5

    # Scale index price and ema to grains
    price_encoding = encode_single(
        "(uint8,(uint128,uint256,uint64))",
        [
            item_type,
            [
                to_base_unit_amount(index_price, 6),
                to_base_unit_amount(abs(ema), 6),
                ordinal,
            ],
        ],
    )
    if self.ema < 0:
        price_encoding_byte_array = bytearray(price_encoding)
        price_encoding_byte_array[-49] = 1
        price_encoding = bytes(price_encoding_byte_array)

    return price_encoding

def abi_decoded_value(abi_encoded_value: str):
    price_encoding_byte_array = bytearray(bytes.fromhex(abi_encoded_value[2:]))
    multiplier = -1 if price_encoding_byte_array[-49] == 1 else 1
    price_encoding_byte_array[-49] = 0
    abi_encoded_value = bytes(price_encoding_byte_array).hex()

    (item_type, (index_price, ema, ordinal)) = decode_single(
        "(uint8,(uint128,uint256,uint64))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale index price and ema from grains
    return (
        to_unit_amount(index_price, 6),
        to_unit_amount(ema * multiplier, 6),
        ordinal,
    )
```

A `Price` leaf holds the following data:

| type    | field       | description                                                                                                                                   |
| ------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| decimal | index_price | Composite index price perpetual is tracking                                                                                                   |
| decimal | ema         | EMA component of price, tracking the difference between the DerivaDEX order book price and the underlying                                     |
| int     | ordinal     | Numerical sequencing identifier for PriceCheckpoint that created this Price state leaf, can be used to arrange Price leaves in order of entry |

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint128,uint256,uint64))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the grains conversion for certain
variables, is shown on the right. It"s demonstrated in the code sample, but it"s worth highlighting that
the `ema` component, which can be a negative value, is ABI-encoded to a 32-byte value where the first 16 bytes will either be
`0x00000000000000000000000000000000` (positive) or `0x00000000000000000000000000000001` (negative), and the next 16 bytes
is the absolute value of the `ema`.

The following sample `Price` materials generates the following ABI-encoded value: `0x00000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000ff439f867f838f4000000000000000000000000000000000000000000000000004d94aaad5556854d300000000000000000000000000000000000000000000000000000000000020e2`.

| field       | value                 |
| ----------- | --------------------- |
| index_price | 4708.7925             |
| ema         | 89.444491182582813907 |
| ordinal     | 8418                  |

#### Insurance fund

An `InsuranceFund` contains information pertaining to the insurance fund"s capitalization.

##### Key encoding / decoding

> Key encoding / decoding (Python)

```python
from eth_abi.utils.padding import zpad32_right

def encode_key():
    # ItemType.INSURANCE_FUND == 6
    return zpad32_right(
        ItemType.INSURANCE_FUND.to_bytes(1, byteorder="little")
        + "OrganicInsuranceFund".encode("utf8")
    )
```

The location of an `InsuranceFund` leaf is determined by its key, which is encoded as
follows:

| Bytes    | Value                                |
| -------- | ------------------------------------ |
| 0        | Insurance fund discriminant          |
| [1, 20]  | "OrganicInsuranceFund" bytes-encoded |
| [21, 31] | Zero-padding                         |

The `InsuranceFund` leaf is located at the encoded key: `0x064f7267616e6963496e737572616e636546756e640000000000000000000000`.

##### Value definition

> Value encoding / decoding (Python)

```python
from typing import Dict
from ddx_python.decimal import Decimal
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(6)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, capitalization: Dict[str, Decimal]):
    # InsuranceFund item discriminant
    item_type = 6

    # Scale collateral amounts to grains
    return encode_single(
        "((uint8,(address[],uint128[])))",
        [
            [
                self.item_type,
                [
                    list(capitalization.keys()),
                    to_base_unit_amount_list(
                        list(capitalization.values()), 6
                    ),
                ],
            ]
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (
        (item_type, (capitalization_tokens, capitalization_amounts,),),
    ) = decode_single(
        "((uint8,(address[],uint128[])))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale collateral amounts from grains
    return cls(
        {
            k: to_unit_amount(v, 6)
            for k, v in zip(
                list(capitalization_tokens), list(capitalization_amounts)
            )
        },
    )
```

An `InsuranceFund` leaf holds the following data:

| type                     | field          | description                                                            |
| ------------------------ | -------------- | ---------------------------------------------------------------------- |
| dict<address_s, decimal> | capitalization | Mapping of collateral address to organic insurance fund capitalization |

These contents are always stored in the tree in ABI-encoded form: `((uint8,(address[],uint128[])))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the grains conversion for certain
variables, is shown on the right.

The following sample `InsuranceFund` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000050000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000001000000000000000000000000cfc18cec799fbd1793b5c43e773c98d4d61cc2db000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000532d4737cf8c716f0000`.

| field          | value                                                        |
| -------------- | ------------------------------------------------------------ |
| capitalization | {"0xcfc18cec799fbd1793b5c43e773c98d4d61cc2db": 392791.65336} |

#### Stats

A `Stats` contains data such as volume info for trade mining for any given trader.

##### Key encoding / decoding

> Key encoding / decoding (Python)

```python
from eth_abi.utils.padding import zpad32_right

def encode_key(trader_address: str):
    # ItemType.STATS == 7
    return zpad32_right(
            ItemType.STATS.to_bytes(1, byteorder="little")
            + bytes.fromhex(trader_address[2:])
        )

def decode_key(stats_key: bytes):
    # chain_discriminant, trader_address
    return f"0x{stats_key[1:22].hex()}"
```

The location of a `Stats` leaf is determined by its key, which is encoded as
follows:

| Bytes    | Value                                                          |
| -------- | -------------------------------------------------------------- |
| 0        | Stats discriminant                                             |
| [1, 21]  | Trader"s Ethereum address prefixed with the chain discriminant |
| [22, 31] | Zero-padding                                                   |

The following sample `Stats` materials generates the following encoded key: `0x0700603699848c84529987e14ba32c8a66def67e9ece00000000000000000000`.

| field          | value                                          |
| -------------- | ---------------------------------------------- |
| trader_address | "0x00603699848c84529987E14Ba32C8a66DEF67E9eCE" |

##### Value definition

> Value encoding / decoding (Python)

```python
from typing import Dict
from ddx_python.decimal import Decimal
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(6)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, maker_volume: Decimal, taker_volume: Decimal):
    # Stats item discriminant
    item_type = 6

    # Scale volume amounts to grains
    return encode_single(
        "(uint8,(uint128,uint128))",
        [
            item_type,
            [
                to_base_unit_amount(maker_volume, 6),
                to_base_unit_amount(taker_volume, 6),
            ],
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (item_type, (maker_volume, taker_volume)) = decode_single(
        "(uint8,(uint128,uint128))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale volumes from DDX grains
    return (to_unit_amount(maker_volume, 6), to_unit_amount(taker_volume, 6))
```

A `Stats` leaf holds the following data:

| type    | field        | description                                           |
| ------- | ------------ | ----------------------------------------------------- |
| decimal | maker_volume | Maker volume of trader during this trade mining epoch |
| decimal | taker_volume | Taker volume of trader during this trade mining epoch |

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint128,uint128))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the grains conversion for certain
variables, is shown on the right.

The following sample `Stats` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000032d26d12e980b60000000000000000000000000000000000000000000000000030fe0cfcba2f4700000`.

| field        | value |
| ------------ | ----- |
| maker_volume | 15000 |
| taker_volume | 14460 |

### Transactions

Transactions refer to the state-changing events that modify the SMT and its leaves (thus the root hash)
described above.

There are 12 various types of transactions on DerivaDEX. The full set of transactions along with their corresponding numeric discrimants can be seen in the table below:

| Event                 | Discriminant |
| --------------------- | ------------ |
| PartialFill           | 0            |
| CompleteFill          | 1            |
| PostOrder             | 2            |
| Cancel                | 3            |
| CancelAll             | 30           |
| Liquidation           | 4            |
| StrategyUpdate        | 5            |
| TraderUpdate          | 6            |
| Withdraw              | 7            |
| WithdrawDDX           | 8            |
| PriceCheckpoint       | 9            |
| PnlSettlement         | 10           |
| Funding               | 11           |
| TradeMining           | 12           |
| SpecsUpdate           | 13           |
| InsuranceFundUpdate   | 14           |
| InsuranceFundWithdraw | 15           |
| DisasterRecovery      | 16           |
| SignerRegistered      | 60           |
| EpochMarker           | 100          |
| NoTransition          | 999          |

Each of these transaction types as received from the Operator WebSocket API are described at length below.

#### Partial fill

> Sample PartialFill (JSON)

```json
{
    "event": [
        {
            "side": "Ask",
            "price": "53008",
            "amount": "7.8",
            "symbol": "BTCPERP",
            "orderHash": "0xcd69dd0341444be5bc46c0a7e9b305bc4a3a322fae3eedd70e",
            "strategyId": "main",
            "bookOrdinal": 41,
            "traderAddress": "0x00aa6676733e2e259d879127519d973ac71135f797"
        },
        {},
        [
            {
                "Fill": {
                    "price": "53163",
                    "amount": "0.1",
                    "reason": "Trade",
                    "symbol": "BTCPERP",
                    "takerSide": "Ask",
                    "makerOutcome": {
                        "fee": "0",
                        "trader": "0x000e646446bfd163dc1cb3010754a7e1a5dd73a6de",
                        "strategy": "main",
                        "ddxFeeElection": false
                    },
                    "takerOutcome": {
                        "fee": "10.6326",
                        "trader": "0x00aa6676733e2e259d879127519d973ac71135f797",
                        "strategy": "main",
                        "ddxFeeElection": false
                    },
                    "makerOrderHash": "0xc7aa4937f79ee9fbc3f8c5f00f46157ac0c0ab6a485227e920",
                    "takerOrderHash": "0xcd69dd0341444be5bc46c0a7e9b305bc4a3a322fae3eedd70e",
                    "makerOrderRemainingAmount": "0"
                }
            }
        ]
    ]
}
```

A `PartialFill` transaction is a scenario where the taker order has been partially filled across 1 or more
maker orders and thus has a remaining order that enters the order book. The event portion of the transaction response consists of a 3-item array. The
first item is the remaining `Post` event, the second item is a dictionary of any relevant price updates, and the third item is a list of trade outcomes, which can either be `TradeFill` or `Cancel` events pertaining to
maker orders that may have been canceled.

#### Complete fill

> Sample CompleteFill (JSON)

```json
{
    "event": [
        {},
        [
            {
                "Fill": {
                    "price": "53163",
                    "amount": "0.1",
                    "reason": "Trade",
                    "symbol": "BTCPERP",
                    "takerSide": "Ask",
                    "makerOutcome": {
                        "fee": "0",
                        "trader": "0x000e646446bfd163dc1cb3010754a7e1a5dd73a6de",
                        "strategy": "main",
                        "ddxFeeElection": false
                    },
                    "takerOutcome": {
                        "fee": "10.6326",
                        "trader": "0x00aa6676733e2e259d879127519d973ac71135f797",
                        "strategy": "main",
                        "ddxFeeElection": false
                    },
                    "makerOrderHash": "0xc7aa4937f79ee9fbc3f8c5f00f46157ac0c0ab6a485227e920",
                    "takerOrderHash": "0xcd69dd0341444be5bc46c0a7e9b305bc4a3a322fae3eedd70e",
                    "makerOrderRemainingAmount": "0"
                }
            }
        ]
    ]
}
```

A `CompleteFill` is a scenario where the taker order has been completely filled across 1 or more maker orders.
The event portion of the transaction response consists of a consists of a trade outcomes, which can either be `TradeFill` or
`Cancel` events pertaining to maker orders that may have been canceled.

#### PostOrder

> Sample PostOrder (JSON)

```json
{
    "event": [
        {
            "side": "Bid",
            "price": "1870.25",
            "amount": "1.39",
            "symbol": "ETHPERP",
            "orderHash": "0x860b1065b629df495de7a0d97432a48b098cadcbb37a1fdbd9",
            "strategyId": "main",
            "bookOrdinal": 0,
            "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
            "timeValue": 1024
        },
        []
    ]
}
```

A `PostOrder` is a 2-item array where the first item is a `Post` order that enters the order book and the second item
is a list of `Cancel` events pertaining to maker orders that may have been canceled. The `Post` portion of the transaction has attributes defined as follows:

#### Cancel

> Sample Cancel (JSON)

```json
{
    "event": {
        "amount": "1.39",
        "symbol": "ETHPERP",
        "orderHash": "0x860b1065b629df495de7a0d97432a48b098cadcbb37a1fdbd9"
    }
}
```

A `Cancel` is when an existing order is canceled and removed from the order book. The event portion of the transaction has attributes defined as follows:

| type      | field           | description                                                                                        |
| --------- | --------------- | -------------------------------------------------------------------------------------------------- |
| dict      | event           | The contents of the transaction event                                                              |
| decimal_s | event.amount    | Size of order canceled from the order book                                                         |
| string    | event.symbol    | Symbol for the market this order has been canceled (e.g. `ETHPERP`)                                |
| bytes32_s | event.orderHash | Hexstr representation of the first 25 bytes of the unique EIP-712 hash of the order being canceled |

#### Liquidation

```json
{
    "event": [
        {
            "positions": [
                [
                    "ETHPERP",
                    {
                        "price": {
                            "ema": "-212.602242612626064356",
                            "ordinal": 157,
                            "indexPrice": "2718.2459",
                            "indexPriceHash": "0x16f0d2a4abfd17f788c900a392cf4297439bae04d0f5a13cd1"
                        },
                        "amount": "6.1",
                        "adlOutcomes": [],
                        "tradeOutcomes": [
                            {
                                "Fill": {
                                    "price": "2322",
                                    "amount": "6.1",
                                    "reason": "Liquidation",
                                    "symbol": "ETHPERP",
                                    "takerSide": "Ask",
                                    "makerOutcome": {
                                        "fee": "0",
                                        "trader": "0x00e834ec434daba538cd1b9fe1582052b880bd7e63",
                                        "strategy": "main",
                                        "realizedPnl": "595.832921348314606732",
                                        "positionSide": "Short",
                                        "newCollateral": "942709.042921348314605958",
                                        "ddxFeeElection": false,
                                        "newPositionBalance": "2.8",
                                        "newPositionAvgEntryPrice": "2419.677528089887640448"
                                    },
                                    "indexPriceHash": "0x16f0d2a4abfd17f788c900a392cf4297439bae04d0f5a13cd1",
                                    "makerOrderHash": "0x63d8e48a6536760deaf29e5f2f2e0039356fc77a437ccc47f7",
                                    "makerOrderRemainingAmount": "4.7"
                                }
                            }
                        ],
                        "newInsuranceFundCap": "734.512240000000000005"
                    }
                ]
            ],
            "strategyId": "main",
            "traderAddress": "0x00d9a605a32d920d90d8119ffd5fe1fd247c500479",
            "canceledOrders": []
        }
    ]
}
```

A `Liquidation` is when a set of strategies are under-collateralized and closed out forcibly closed out:

| type      | field                                     | description                                                              |
| --------- | ----------------------------------------- | ------------------------------------------------------------------------ |
| list      | event                                     | The contents of the transaction event                                    |
| dict      | event[n]                                  | Liquidation entry                                                        |
| address_s | event[n].traderAddress                    | Liquidated trader"s prefixed Ethereum address                            |
| string    | event[n].strategyId                       | Liquidated trader"s strategy id                                          |
| dict      | event[n].strategyId                       | Liquidated trader"s strategy id                                          |
| list      | event.canceledOrders                      | `Cancel` events pertaining to canceled orders for the liquidated trader  |
| list      | event.positions                           | List of tuples pertaining to liquidated positions                        |
| string    | event.positions[n][0]                     | Liquidated position symbol                                               |
| dict      | event.positions[n][1]                     | Liquidated position details                                              |
| dict      | event.positions[n][1].price               | Latest `PriceCheckpoint` corresponding to this position being liquidated |
| decimal_s | event.positions[n][1].amount              | Amount of position being liquidated                                      |
| list      | event.positions[n][1].adlOutcomes         | ADL outcomes as a result of liquidation                                  |
| list      | event.positions[n][1].tradeOutcomes       | Trade outcomes as a result of liquidation                                |
| decimal_s | event.positions[n][1].newInsuranceFundCap | New insurance fund capitalization after position has been liquidated     |

#### Strategy update

> Sample StrategyUpdate (JSON)

```json
{
    "event": {
        "amount": "1000000",
        "trader": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "txHash": "0x239a452c6abc453e153c8c53e4bdb995c558619b3aa1980c8977cbc4c283618f",
        "strategyId": "main",
        "updateType": "Deposit",
        "collateralAddress": "0xb69e673309512a9d726f87304c6984054f87a93b"
    }
}
```

A `StrategyUpdate` is an update to a trader's strategy (such as depositing or withdrawing collateral). The event portion of the transaction has attributes defined as follows:

| type          | field                   | description                                                     |
| ------------- | ----------------------- | --------------------------------------------------------------- |
| dict          | event                   | The contents of the transaction event                           |
| decimal_s     | event.amount            | Amount deposited or withdrawn                                   |
| address_pre_s | event.trader            | Trader"s Ethereum address prefixed with the chain discriminant  |
| bytes32_s     | event.txHash            | Ethereum transaction hash for the on-chain deposit / withdrawal |
| string        | event.strategyId        | Strategy ID deposited to or withdrawn from (e.g. "main")        |
| string        | event.updateType        | Action corresponding to either "Deposit" or "Withdraw"          |
| address_s     | event.collateralAddress | Deposited / withdrawn collateral token"s Ethereum address       |

#### Trader update

> Sample TraderUpdate (JSON)

```json
{
    "event": {
        "amount": "1000000",
        "trader": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "txHash": "0x239a452c6abc453e153c8c53e4bdb995c558619b3aa1980c8977cbc4c283618f",
        "updateType": "Deposit"
    }
}
```

A `TraderUpdate` is an update to a trader"s DDX account (such as depositing or withdrawing DDX). The event portion of the transaction has attributes defined as follows:

| type          | field            | description                                                     |
| ------------- | ---------------- | --------------------------------------------------------------- |
| dict          | event            | The contents of the transaction event                           |
| decimal_s     | event.amount     | Amount of DDX deposited or withdrawn                            |
| address_pre_s | event.trader     | Trader"s Ethereum address prefixed with the chain discriminant  |
| bytes32_s     | event.txHash     | Ethereum transaction hash for the on-chain deposit / withdrawal |
| string        | event.updateType | Action corresponding to either "Deposit" or "Withdraw"          |

#### Withdraw

> Sample Withdraw (JSON)

```json
{
    "event": {
        "amount": "1000",
        "currency": "0xb69e673309512a9d726f87304c6984054f87a93b",
        "strategy": "main",
        "signerAddress": "0x6ecbe1db9ef729cbe972c83fb886247691fb6beb",
        "traderAddress": "0x006ecbe1db9ef729cbe972c83fb886247691fb6beb"
    }
}
```

A `Withdraw` is when a withdrawal of a collateral token is signaled. The event portion of the transaction has attributes defined as follows:

| type          | field               | description                                                                                     |
| ------------- | ------------------- | ----------------------------------------------------------------------------------------------- |
| dict          | event               | The contents of the transaction event                                                           |
| decimal_s     | event.amount        | Amount of collateral token withdrawal has been signaled for                                     |
| address_s     | event.currency      | Collateral ERC-20 token address                                                                 |
| string        | event.strategy      | Strategy ID the withdrawal applies to                                                           |
| address_s     | event.signerAddress | Signer"s Ethereum address withdrawal is taking place from                                       |
| address_pre_s | event.traderAddress | Trader"s Ethereum address prefixed with the chain discriminant collateral is being withdrawn to |

#### Withdraw DDX

> Sample WithdrawDDX (JSON)

```json
{
    "event": {
        "amount": "100",
        "signerAddress": "0xa8dda8d7f5310e4a9e24f8eba77e091ac264f872",
        "traderAddress": "0x00603699848c84529987e14ba32c8a66def67e9ece"
    }
}
```

A `WithdrawDDX` is when a withdrawal of DDX is signaled. The event portion of the transaction has attributes defined as follows:

| type          | field               | description                                                                              |
| ------------- | ------------------- | ---------------------------------------------------------------------------------------- |
| dict          | event               | The contents of the transaction event                                                    |
| decimal_s     | event.amount        | Amount of DDX withdrawal has been signaled for                                           |
| address_s     | event.signerAddress | Signer"s Ethereum address withdrawal is taking place from                                |
| address_pre_s | event.traderAddress | Trader"s Ethereum address prefixed with the chain discriminant DDX is being withdrawn to |

#### Price checkpoint

> Sample PriceCheckpoint (JSON)

```json
{
    "event": {
        "ETHPERP": {
            "ema": "0",
            "indexPrice": "1874.075454545454545454",
            "indexPriceHash": "0xa35d3dac391cc18671daacab1c393554aac1546a2cbfe630da1e4e86fb44f542"
        }
    }
}
```

A `PriceCheckpoint` is when a market registers an update to the composite index price a perpetual is tracking along with the ema component. The event portion of the transaction essentially emits the latest `Price` leaf contents, and is defined as follows:

| type      | field                       | description                                                                                                              |
| --------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| dict      | event                       | The contents of the transaction event                                                                                    |
| string    | event.symbol                | The market name for which this `PriceCheckpoint` transaction applies                                                     |
| decimal_s | event.symbol.ema            | Captures a smoothed average of the spread between the underlying index price and the DerivaDEX order book for the market |
| decimal_s | event.symbol.indexPrice     | Composite index price (a weighted average across several price feed sources)                                             |
| bytes32_s | event.symbol.indexPriceHash | Index price hash                                                                                                         |

#### PNL Settlement

> Sample PnlSettlement (JSON)

```json
{
    "event": {
        "settlementEpochId": 19
    }
}
```

A `PNLSettlement` is when a there is a PNL settlement event. This event realizes the PNL of all open positions and adjusts the average entry price to the current mark price. The event portion of the transaction has attributes defined as follows:

| type | field                   | description                               |
| ---- | ----------------------- | ----------------------------------------- |
| dict | event                   | The contents of the transaction event     |
| int  | event.settlementEpochId | The epoch id for the PNL settlement event |

#### Funding

> Sample Funding (JSON)

```json
{
    "event": [
        {
            "fundingRates": {
                "ETHPERP": "0.005"
            },
            "settlementEpochId": 9
        },
        []
    ]
}
```

A `Funding` is when a there is a funding rate distribution. At this time, all strategies are either credited or debited an amount based on the current funding rate and their position notional values and side. The event portion of the transaction has attributes defined as follows:

| type                    | field                      | description                                                                     |
| ----------------------- | -------------------------- | ------------------------------------------------------------------------------- |
| list                    | event                      | The contents of the transaction event                                           |
| int_s                   | event[0].settlementEpochId | The epoch id for the funding event                                              |
| dict<string, decimal_s> | event[0].fundingRates      | Mapping of market symbol to funding rate                                        |
| list                    | event[1]                   | `Liquidation` events that have taken place as a result of funding distributions |

#### Trade mining

> Sample TradeMining (JSON)

```json
{
    "event": {
        "totalVolume": {
            "makerVolume": "41249.393239489105059823",
            "takerVolume": "40101.194995030184814398"
        },
        "ddxDistributed": "3192.388588391993848193",
        "tradeMiningEpochId": 15
    }
}
```

A `TradeMining` is when there is a trade mining distribution. Traders will receive a portion of the overall DDX distributed proportional to their volume traded that interval. Makers will receive 20% of the DDX allocation and takers will receive 80% of the allocation. The event portion of the transaction has attributes defined as follows:

| type      | field                         | description                                                                                                                                                                                                                                                          |
| --------- | ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dict      | event                         | The contents of the transaction event                                                                                                                                                                                                                                |
| dict      | event.totalVolume             | Holds the total volume information for this trade mining interval                                                                                                                                                                                                    |
| decimal_s | event.totalVolume.makerVolume | Maker volume total for this trade mining interval                                                                                                                                                                                                                    |
| decimal_s | event.totalVolume.takerVolume | Taker volume total for this trade mining interval. Note that this number may be <= the `makerTotalVolume`, and any discrepancy is due to liquidations only counting towards maker allocation (i.e. the liquidation engine does not receive any DDX per trade mining) |
| decimal_s | event.ddxDistributed          | Total DDX distributed for trade mining for this interval                                                                                                                                                                                                             |
| int       | event.tradeMiningEpochId      | Interval counter for when trade mining occurred                                                                                                                                                                                                                      |

#### Post

> Sample Post (JSON)

```json
{
    "event": {
        "side": "Bid",
        "price": "1870.25",
        "amount": "1.39",
        "symbol": "ETHPERP",
        "orderHash": "0x860b1065b629df495de7a0d97432a48b098cadcbb37a1fdbd9",
        "strategyId": "main",
        "bookOrdinal": 0,
        "traderAddress": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "timeValue": 1024
    }
}
```

A `Post` is an order that enters the order book. The transaction has attributes defined as follows:

| type          | field               | description                                                                                                                                                                 |
| ------------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| dict          | event               | The contents of the transaction event                                                                                                                                       |
| decimal_s     | event.amount        | Size of order posted to the order book                                                                                                                                      |
| bytes32_s     | event.orderHash     | Hexstr representation of the first 25 bytes of the unique EIP-712 hash of the order being placed                                                                            |
| decimal_s     | event.price         | Price the order has been placed at                                                                                                                                          |
| string        | event.side          | Side of order (`Bid` or `Ask`)                                                                                                                                              |
| string        | event.strategyId    | Strategy ID this order belongs to (e.g. "main")                                                                                                                             |
| string        | event.symbol        | Symbol for the market this order has been placed (e.g. `ETHPERP`)                                                                                                           |
| int           | event.bookOrdinal   | Numerical value signifying sequence of order placement, which can be used to arrange multiple orders at the same price level to achieve FIFO priority in a local order book |
| address_pre_s | event.traderAddress | Trader"s Ethereum address prefixed with the chain discriminant                                                                                                              |
| timeValue     | event.timeValue     | Time value                                                                                                                                                                  |

#### TradeFill

> Sample TradeFill (JSON)

```json
{
    "event": {
        "price": "1860",
        "amount": "1.23",
        "reason": "Trade",
        "symbol": "ETHPERP",
        "takerSide": "Ask",
        "makerOutcome": {
            "fee": "0",
            "trader": "0x006ecbe1db9ef729cbe972c83fb886247691fb6beb",
            "strategy": "main",
            "realizedPnl": "0",
            "positionSide": "Long",
            "newCollateral": "1000000",
            "ddxFeeElection": false,
            "newPositionBalance": "2.46",
            "newPositionAvgEntryPrice": "1860"
        },
        "takerOutcome": {
            "fee": "4.5756",
            "trader": "0x00e36ea790bc9d7ab70c55260c66d52b1eca985f84",
            "strategy": "main",
            "realizedPnl": "0",
            "positionSide": "Short",
            "newCollateral": "999990.8488",
            "ddxFeeElection": false,
            "newPositionBalance": "2.46",
            "newPositionAvgEntryPrice": "1860"
        },
        "makerOrderHash": "0x542ad7f2640447d3e93723253099fd45ae721925e0df64702e",
        "takerOrderHash": "0x100d1edd45edede9eedc0e3aac28e7df17168fbef9b459ffc8",
        "makerOrderRemainingAmount": "2.54"
    }
}
```

A `TradeFill` is when there is a fill / match that has taken place. Technically, a `TradeFill` is not a transaction in it of itself, but rather a part of `CompleteFill` and `PartialFill` transactions. We often look to these individual `TradeFill` events as their own dedicated transaction type, however, for Traders to subscribe to and follow as they would other transactions. The event portion of the transaction has attributes defined as follows:

| type          | field                                       | description                                                                             |
| ------------- | ------------------------------------------- | --------------------------------------------------------------------------------------- |
| dict          | event                                       | The contents of the transaction event                                                   |
| decimal       | event.price                                 | Price the fill took place                                                               |
| decimal       | event.amount                                | Size of the fill                                                                        |
| string        | event.reason                                | Reason for fill (`Trade`, `Liquidation`)                                                |
| string        | event.symbol                                | Market symbol for which this fill took place                                            |
| string        | event.takerSide                             | Taker / aggressor side (`Bid`, `Ask`)                                                   |
| dict          | event.makerOutcome                          | Data containing the some relevant information pertaining to the maker in the trade      |
| address_pre_s | event.makerOutcome.traderAddress            | Maker trader"s Ethereum address prefixed with the chain discrimimant                    |
| string        | event.makerOutcome.strategyId               | Cross-margined strategy identifier for the maker trader                                 |
| decimal_s     | event.makerOutcome.fee                      | Fees paid by the maker as a result of the trade                                         |
| bool          | event.makerOutcome.ddxFeeElection           | Whether maker fees are being paid in DDX (`true`) or USDC (`false`)                     |
| decimal_s     | event.makerOutcome.realizedPNL              | Realized profits and losses for the maker as a result of the trade (not including fees) |
| decimal_s     | event.makerOutcome.newCollateral            | New free collateral in maker trader"s strategy as a result of the trade                 |
| decimal_s     | event.makerOutcome.newPositionAvgEntryPrice | New average entry price for maker"s position as a result of the trade                   |
| decimal_s     | event.makerOutcome.newPositionBalance       | New size for maker"s position as a result of the trade                                  |
| int           | event..makerOutcome.positionSide            | New side for maker"s position (`long=1`, `short=2`) as a result of the trade            |
| dict          | event.takerOutcome                          | Data containing the some relevant information pertaining to the taker in the trade      |
| address_pre_s | event.takerOutcome.traderAddress            | Taker trader"s Ethereum address prefixed with the chain discrimimant                    |
| string        | event.takerOutcome.strategyId               | Cross-margined strategy identifier for the taker trader                                 |
| decimal_s     | event..takerOutcome.fee                     | Fees paid by the taker as a result of the trade                                         |
| bool          | event.takerOutcome.ddxFeeElection           | Whether taker fees are being paid in DDX (`true`) or USDC (`false`)                     |
| decimal_s     | event.takerOutcome.realizedPNL              | Realized profits and losses for the taker as a result of the trade (not including fees) |
| decimal_s     | event.takerOutcome.newCollateral            | New free collateral in taker trader"s strategy as a result of the trade                 |
| decimal_s     | event.takerOutcome.newPositionAvgEntryPrice | New average entry price for taker"s position as a result of the trade                   |
| decimal_s     | event.takerOutcome.newPositionBalance       | New size for taker"s position as a result of the trade                                  |
| int           | event.takerOutcome.positionSide             | New side for taker"s position (`long=1`, `short=2`) as a result of the trade            |

### Subscription

You may subscribe to the SMT state leaves and transaction log in raw fashion (much like how the Auditor does behind the scenes prior to exposing a clean API for your consumption).

Operator transaction log WS endpoint URL: wss://api.derivadex.io/node0/v2/txlog

#### Response

> Snapshot response (JSON)

```json
{
    "t": "Snapshot",
    "c": {
        "epochId": 169,
        "leaves": {
            "0x04820e582404007b98abbb2526d07a2cdc0d67f35907d62f0e21bbbe45e9961a": "0x0000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000e576eaa6c85d2cd80000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000147a",
            "0x03852258240400788b9b1a5b0add97f65952f7cbe9a86b2a8e5f6bfeb5bbf534": "0x000000000000000000000000000000000000000000000000000000000000000300000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000004563918244f400000000000000000000000000000000000000000000000001064092f6c4533a000000e36ea790bc9d7ab70c55260c66d52b1eca985f8400000000000000000000002576ebd10000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000149ab",
            "0x04820e582404001de62df182b9019c994b3d82deb541ddca7c086a8ff691902d": "0x0000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000e1b23e43485b035c00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001281",
            "0x04820e58240400f1771fba7a7e4a9365b0e06d5ac603bd60f7f871edbcf1e0ab": "0x0000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000e54354372e48b48400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000144"
        },
        "offset": 0
    }
}
```

Upon subscription, you will first receive a set of `Snapshot` entries back, containing a snapshot of the DerivaDEX state as of the latest checkpoint.
DerivaDEX has on-chain checkpoints roughly every 10 minutes. Say you were to connect to the API and subscribe to this endpoint 7 minutes after Checkpoint 168 had completed (i.e., Checkpoint 169 had begun).
You would first receive the entire state as of Checkpoint 169.

| type                     | field   | description                                                                                          |
| ------------------------ | ------- | ---------------------------------------------------------------------------------------------------- |
| string                   | t       | WebSocket message type, pertaining to the topic subscribed to, which in this case will be `Snapshot` |
| list                     | c       | `Snapshot` WebSocket message contents                                                                |
| int                      | epochId | Current epoch ID                                                                                     |
| dict                     | leaves  | Snapshot leaf data                                                                                   |
| dict<bytes32_s, bytes_s> | leaves  | Snapshot leaf data mapping leaf key to leaf value (ABI-encoding of SMT leaf"s value)                 |

> Head response (JSON)

```json
{
    "t": "Head",
    "c": {
        "epochId": 169,
        "ordinal": 14,
        "stateRootHash": "0xe0870028a6d8f2dd98961a15b27279d5a1e7deedea3b55e8342e06ce81ea5d9c",
        "requestIndex": 282626,
        "batchId": 282626,
        "event": {
            "t": "Post",
            "c": {
                "orderHash": "0x773ac90b2eefd0b7e772b6f59c41d2ae8b2f7ad17985d94fd3",
                "traderAddress": "0x00e834ec434daba538cd1b9fe1582052b880bd7e63",
                "strategyId": "main",
                "symbol": "ETHPERP",
                "side": "Bid",
                "price": "4831.9",
                "amount": "5.4",
                "bookOrdinal": 95749
            }
        },
        "signature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
    }
}
```

After the snapshot messages, you will next receive a set of `Head` entries back, containing all of the transactions that have already
occurred from the beginning of this epoch to now. Say you were to connect to the API and subscribe to this endpoint 7 minutes after Checkpoint 168 had completed (i.e., Checkpoint 169 had begun).
You would receive all the transactions that have occurred during these 7 minutes.

| type      | field           | description                                                                                                                                                                                                                                                                                                                                    |
| --------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t               | WebSocket message type, pertaining to the topic subscribed to, which in this case will be `Head`                                                                                                                                                                                                                                               |
| list      | c               | `Head` WebSocket message contents                                                                                                                                                                                                                                                                                                              |
| int       | c.epochId       | Current epoch ID                                                                                                                                                                                                                                                                                                                               |
| int_s     | c.ordinal       | Monotonically increasing numerical value representing when a transaction took place in a given epoch. This will necessarily increase by 1 in order of being processed, and will only be reset to 0 when a new epoch has begun.                                                                                                                 |
| int_s     | c.requestIndex  | Sequenced identifier for the request initially made. This will be the same as the `requestIndex` you receive in a successful `command`.                                                                                                                                                                                                        |
| bytes32_s | c.stateRootHash | SMT"s state root hash prior to the transaction being applied                                                                                                                                                                                                                                                                                   |
| bytes32_s | c.batchId       | Monotonically increasing numerical value representing when a batch of transactions took place in a given epoch. This is different from the transaction"s ordinal value, which may actually be the same across transactions that are batched (e.g., multiple deposits in the same Ethereum block, or settlement transactions bunched together). |
| bytes_s   | c.signature     | Enclave"s signature of transaction data                                                                                                                                                                                                                                                                                                        |
| dict      | c.event         | Event data (structure and contents are different depending on the kind of event). Examples for each transaction type are shown above in the [Transactions](#transactions) section.                                                                                                                                                             |

> Tail response (JSON)

```json
{
    "t": "Tail",
    "c": {
        "epochId": 169,
        "ordinal": 691,
        "stateRootHash": "0xb359790e85a5f4d9b0a9b5317ec0f28a1c28fab368d155eeedd29fb96c2999f2",
        "requestIndex": 283687,
        "batchId": 283687,
        "event": {
            "t": "Post",
            "c": {
                "orderHash": "0x97824f71decfaba632ea794fa648f772e64d0bc90ddd397995",
                "traderAddress": "0x00e834ec434daba538cd1b9fe1582052b880bd7e63",
                "strategyId": "main",
                "symbol": "ETHPERP",
                "side": "Ask",
                "price": "4803.9",
                "amount": "6.2",
                "bookOrdinal": 95936
            }
        },
        "signature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
    }
}
```

After the head messages, you will next receive `Tail` entries one by one streaming through with every new transaction
processed.

| type      | field           | description                                                                                                                                                                                                                                                                                                                                    |
| --------- | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| string    | t               | WebSocket message type, pertaining to the topic subscribed to, which in this case will be `Tail`                                                                                                                                                                                                                                               |
| list      | c               | `Tail` WebSocket message contents                                                                                                                                                                                                                                                                                                              |
| int       | c.epochId       | Current epoch ID                                                                                                                                                                                                                                                                                                                               |
| int_s     | c.ordinal       | Monotonically increasing numerical value representing when a transaction took place in a given epoch. This will necessarily increase by 1 in order of being processed, and will only be reset to 0 when a new epoch has begun.                                                                                                                 |
| int_s     | c.requestIndex  | Sequenced identifier for the request initially made. This will be the same as the `requestIndex` you receive in a successful `command`.                                                                                                                                                                                                        |
| bytes32_s | c.stateRootHash | SMT"s state root hash prior to the transaction being applied                                                                                                                                                                                                                                                                                   |
| bytes32_s | c.batchId       | Monotonically increasing numerical value representing when a batch of transactions took place in a given epoch. This is different from the transaction"s ordinal value, which may actually be the same across transactions that are batched (e.g., multiple deposits in the same Ethereum block, or settlement transactions bunched together). |
| bytes_s   | c.signature     | Enclave"s signature of transaction data                                                                                                                                                                                                                                                                                                        |
| dict      | c.event         | Event data (structure and contents are different depending on the kind of event). Examples for each transaction type are shown above in the [Transactions](#transactions) section.                                                                                                                                                             |

# Validation

## Order notional

You will not be able to place orders exceeding $1,000,000 in notional. The notional value of an order
is computed as follows: `notional = order_amount * mark_price`.

## OMF < IMF

You will not be able to place orders if the resulting open margin fraction (OMF) would become less
than the initial margin fraction (IMF).

## Max taker price deviation

You will not be able to place a limit order at a price more than 2% through the best level on the other side of the
book. In other words, you cannot place a limit buy order at a price greater than 2% higher than the lowest available offer.
Conversely, you cannot place a limit sell order at a price less than 2% lower than the highest available bid. If there
is no liquidity on the other side of the book, the mark price is used instead.

## Self-match prevention

You will not be able to trade against yourself. Although this is technically not an API validation, it is worth mentioning here
regardless. The self-match rules are such that the remainder of the incoming order will be canceled if it were to self-match.

# Rate limits

The websocket API will (eventually) use tiers to determine rate limits for each ethereum account.
Rate limits restrict the number of "Commands" an account can place per minute.

Subscriptions are not rate limited.
