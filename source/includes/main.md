
# DerivaDEX API

DerivaDEX is a decentralized derivatives exchange that combines the performance of centralized exchanges with the security of decentralized exchanges.

DerivaDEX currently offers a public WebSocket API for traders and developers. In order to use this API, you must must first make a deposit to the DerivaDEX Ethereum contracts. The API will then enable you to open and manage your positions via commands, and subscribe to market data via `subscriptions`. 

Find us online [Discord](https://discord.gg/a54BWuG) | [Telegram](https://t.me/DerivaDEX) | [Medium](https://medium.com/derivadex)


# Getting started

To begin interacting with the DerivaDEX ecosystem programmatically, you generally will want to follow these steps:
1. Deposit funds via Ethereum via an Ethereum client
2. Authenticate and connect to the websocket API
3. Submit and cancel orders via `commands`
4. Subscribe to market and account data feeds via `subscriptions`

Additionally, you should familiarize yourself with the [DerivaDEX types terminology](#Types) and [hashing & signing schemes](#Signatures & hashing). 


## Types

The types labeled throughout this document in the request and response parameters may be familiar to those who have a background in Ethereum. In any case, please refer to the table below for additional information on the terminology used here. This reference in conjunction with the JSON samples should provide enough clarity:

type | description |example
-----| --------------------- | -----------
string | Literal of a sequence of characters surrounded by quotes | "ETHPERP"
address_s  | 20-byte "0x"-prefixed hexadecimal string literal (i.e. 40 digits long) corresponding to an `address` ETH type | "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
decimal | Numerical value with up to, but no more than 18 decimals of precision | 10.031500000000000000
decimal_s  | String representation of `decimal` | "10.031500000000000000"
bool  | Boolean value, either `true` or `false` | True
bytes32_s  | 32-byte "0x"-prefixed hexadecimal string literal (i.e. 64 digits long) corresponding to an `bytes32` ETH type | "0x0000000000000000000000000000000000000000000000000000000000000001"
timestamp_s_i | String representation of numerical UNIX timestamp representing the number of seconds since 1/1/1970 | "1616667513875"
timestamp_s | String representation representing the ISO 8601 UTC timestamp | "2021-03-25T10:38:09.503654+00:00"


## Signatures & hashing

All [`commands`](#Commands) on the API must be signed. The payload you will sign using an Ethereum wallet client of your choice (e.g. ethers, web3.js, web3.py, etc.) will need to be hashed as per the EIP-712 standard. We **highly recommend** referring to the [original proposal](https://eips.ethereum.org/EIPS/eip-712) for full context, but in short, this standard introduced a framework by which users can securely sign typed structured data. This greatly improves the crypto UX as users can now sign data they see and understand as opposed to unreadable byte-strings. While these benefits may not be readily apparent for programmatic traders, you will need to conform to this standard regardless.

EIP-712 hashing consists of three critical components - a `header`, `domain` struct hash, and `message` struct hash.


### Header

> Sample EIP-191 header definition
```plaintext
// solidity
bytes2 eip191_header = 0x1901;
```

```python
eip191_header = b"\x19\x01"
```

The `header` is simply the byte-string `\x19\x01`. You are welcome to do this however you like, but it must adhere to the standard eventually, otherwise the signature will not ultimately successfully recover. Example Solidity and Python reference implementations are displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.


### Domain

> Domain separator for mainnet. DO NOT modify these parameters.
```json
{
    "name": "DerivaDEX",
    "version": "1",
    "chainId":  1,
    "verifyingContract": "0x"
}
```

> Sample computation of domain struct hash
```plaintext
// solidity
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

def compute_eip712_domain_struct_hash(chain_id):
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
        + encode_single('uint256', chain_id)
        + encode_single('address', "0x")
    )
```

The `domain` is a mandatory field that allows for signature/hashing schemes on one dApp to be unique to itself from other dApps. All `commands` use the same `domain` specification. The parameters that comprise the domain are as follows:

type | field | description
-----|----- | ---------------------
name  | string | Name of the dApp or protocol
version  | string | Current version of the signing domain
chainId | uint256 | EIP-155 chain ID

To generate the `domain` struct hash, you must perform a series of encodings and hashings of the schema and contents of the `domain` specfication. You are welcome to do this however you like, but it must adhere to the standard eventually, otherwise the signature will not ultimately successfully recover. Example Solidity and Python reference implementations are displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.

### Message

The `message` field varies depending on the typed data you are signing, and is illustrated on a case-by-case basis below.

### Place order (message)

> Sample computation of order struct hash
```plaintext
// solidity
function compute_eip712_order_struct_hash(address _traderAddress, bytes32 _symbol, bytes32 _strategy, uint256 _side, uint256 _orderType, bytes32 _requestId, uint256 _amount, uint256 _price, uint256 _stopPrice) public view returns (bytes32) {
    // keccak-256 hash of the encoded schema for the order params struct
    bytes32 orderSchemaHash = keccak256(abi.encodePacked(
        "OrderParams(",
        "address traderAddress,",
        "bytes32 symbol,",
        "bytes32 strategy,",
        "uint256 side,",
        "uint256 orderType,",
        "bytes32 requestId,",
        "uint256 amount,",
        "uint256 price,",
        "uint256 stopPrice",
        ")"
    ));
    
    bytes32 orderStructHash = keccak256(abi.encodePacked(
        orderSchemaHash,
        uint256(_traderAddress),
        _symbol,
        _strategy,
        _side,
        _orderType,
        _requestId,
        _amount,
        _price,
        _stopPrice
    ));
    
    return orderStructHash;
}
```

```python
from eth_abi import encode_single
from eth_utils.crypto import keccak
from decimal import Decimal, ROUND_DOWN

def compute_eip712_order_struct_hash(trader_address: str, symbol: str, strategy: str, side: str, order_type: str, request_id: str, amount: Decimal, price: Decimal, stop_price: Decimal) -> bytes:
    # keccak-256 hash of the encoded schema for the place order command
    eip712_order_params_schema_hash = keccak(
        b"OrderParams("
        + b"address traderAddress,"
        + b"bytes32 symbol,"
        + b"bytes32 strategy,"
        + b"uint256 side,"
        + b"uint256 orderType,"
        + b"bytes32 requestId,"
        + b"uint256 amount,"
        + b"uint256 price,"
        + b"uint256 stopPrice"
        + b")"
    )
    
    # Ensure decimal value has no more than 18 decimals of precision
    def round_to_unit(val):
        return val.quantize(Decimal(".000000000000000001"), rounding=ROUND_DOWN)
    
    # Scale up to DDX grains format (i.e. multiply by 1e18)
    def to_base_unit_amount(val, decimals):
        return int(round_to_unit(val) * 10 ** decimals)

    # Convert order side string to int representation
    def order_side_to_int(order_side: str) -> int:
        if order_side == 'Bid':
            return 0
        return 1
    
    # Convert order type string to int representation
    def order_type_to_int(order_type: str) -> int:
        if order_type == 'Limit':
            return 0
        elif order_type == 'Market':
            return 1
        return 2

    return keccak(
        eip712_order_params_schema_hash
        + encode_single('address', trader_address)
        + encode_single('bytes32', symbol.encode('utf8'))
        + encode_single('bytes32', strategy.encode('utf8'))
        + encode_single('uint256', order_side_to_int(side))
        + encode_single('uint256', order_type_to_int(order_type))
        + encode_single('bytes32', bytes.fromhex(request_id[2:]))
        + encode_single('uint256', to_base_unit_amount(amount, 18))
        + encode_single('uint256', to_base_unit_amount(price, 18))
        + encode_single('uint256', to_base_unit_amount(stop_price, 18))
    )
```

The parameters that comprise the `message` for the `command` to place an order are as follows:

type | field | description
-----|----- | ---------------------
traderAddress  | address | Trader's ETH address used to sign order intent
symbol  | bytes32 | 32-byte encoding of the symbol this order is for. The `symbol` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly.
strategy | bytes32 | 32-byte encoding of the strategy this order belongs to. The `strategy` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly. The `strategy` refers to the cross-margined bucket this trade belongs to. Currently, there is only the default `main` strategy, but support for multiple strategies is coming soon!
side | uint256 | An integer value either `0` (Bid) or `1` (Ask)
orderType | uint256 | An integer value either `0` (Limit) or `1` (Market)
requestId | bytes32 | 32-byte nonce (an incrementing numeric identifier that is unique per user for all time) resulting in uniqueness of order
amount | uint256 | Order amount (scaled up by 18 decimals; e.g. 2.5 => 2500000000000000000). The `amount` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.
price | uint256 | Order price (scaled up by 18 decimals; e.g. 2001.37 => 2001370000000000000000). The `price` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.
stopPrice | uint256 | Stop price (scaled up by 18 decimals). The `stopPrice` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.

**Take special note of the transformations done on several fields as described in the table above. In other words, the order intent you submit to the API will have different representations for some fields than the order intent you hash.** You are welcome to do this however you like, but it must adhere to the standard eventually, otherwise the signature will not ultimately successfully recover. Example Solidity and Python reference implementations are displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.


### Tying it all together

> Computing the final EIP-712 hash
```plaintext
function compute_eip712_hash(bytes2 _eip191_header, bytes32 _domainStructHash, bytes32 _orderStructHash) public view returns (bytes32) {
    return keccak256(abi.encodePacked(
        _eip191_header,
        _domainStructHash,
        _orderStructHash
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
```plaintext
function deposit(
    address _collateralAddress,
    bytes32 _strategyId,
    uint128 _amount
) external;
```

> Sample API URL connection generation (Python)
```python
from web3 import Web3
from eth_abi import encode_single
import simplejson as json

# A Web3 instance
w3 = Web3(Web3.HTTPProvider("https://kovan.infura.io/v3/<your_api_key>"))

# Open up the ABI shown above saved at a file location, in this case `trader_abi.json` is the filename
with open('trader_abi.json') as f:
    trader_abi = json.load(f)
    
    # Create an trader_abi contract wrapper
    trader_contract = w3.eth.contract(address=Web3.toChecksumAddress('<derivadex_contract_address>'), abi=trader_abi)
    
    # Deposit 1000 USDC
    trader_contract.functions.deposit('<usdc_address>', encode_single("bytes32", 'main'.encode("utf8")), 1000000000).transact()
```

DerivaDEX is a decentralized exchange. As such, trading is non-custodial. Users are responsible for their own funds, which are deposited to the DerivaDEX smart contracts on Ethereum for trading. 

To deposit funds on DerivaDEX, first ensure that you have created an Ethereum account. The deposit interaction is between a user and the DerivaDEX smart contracts. To be more explicit, you will not be utilizing the WebSocket API to facilitate a deposit. The DerivaDEX Solidity smart contracts adhere to the [Diamond Standard](https://medium.com/derivadex/the-diamond-standard-a-new-paradigm-for-upgradeability-569121a08954). The `deposit` smart contract function you will need to use is located in the `Trader` facet, at the address of the main `DerivaDEX` proxy contract (insert address here).

Note: Valid deposit collateral is curated by the smart contract and is managed by governance.

field | description 
------ | -----------
_collateralAddress | ERC-20 token address deposited as collateral
_strategyId | Strategy ID (encoded as a 32-byte value) being funded. The `strategy` refers to the cross-margined bucket this trade belongs to. Currently, there is only the default `main` strategy, but support for multiple strategies is coming soon!
_amount | Amount deposited (be sure to use the grains format specific of the collateral token you are using (e.g. if you wanted to deposit 1 USDC, you would enter 1000000 since the USDC token contract has 6 decimal places)

An example Python implementation is displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.

## Connecting to the Websocket API

> Sample API URL connection generation (Python)
```python
from web3 import Web3
from eth_account.messages import encode_defunct
from eth_abi import encode_abi
import time

def get_url(self) -> str:
    # A Web3 instance
    w3 = Web3(Web3.HTTPProvider("https://kovan.infura.io/v3/<your_api_key>"))
    
    # Initialize a Web3 account from a private key
    web3_account = self.w3.eth.account.from_key("<private_key>")

    # Retrieve current UNIX time in nanoseconds to derive a unique, monotonically-increasing nonce
    nonce = time.time_ns()

    # keccak256(abi.encode(['bytes32', 'uint256'], [bytes_encoded('DerivaDEX API'), nonce]))
    connect_message = w3.keccak(
        encode_abi(["bytes32", "uint256"], ["DerivaDEX API".encode("utf8"), nonce])
    )

    # Obtain signable message
    encoded_message = encode_defunct(text=connect_message.hex())

    # Obtain signature
    signature = web3_account.sign_message(encoded_message)

    # Construct WS connection url with format
    return f"wss://alpha.derivadex.io/trader/v1?nonce={nonce}&signature={signature.signature.hex()}"
```

Addresses used to connect to the websocket API must _already_ have funds deposited. If you haven't, [do that first](#Making-a-deposit).

The steps to connect are:

1. Generate a numeric value (`nonce`) that is greater than the last value used for this address. Typically, [Unix time](https://en.wikipedia.org/wiki/Unix_time) (either milliseconds or nanoseconds) is used, but users may opt to maintain their own sequence counter. Requests with a nonce that are less than or equal to the previous value will be rejected.
2. `Keccak-256` hash the following ABI-encoded values with the specified types:
    - "DerivaDEX API" encoded as bytes (`bytes32`) 
    - `nonce` (`uint256`)
3. Generate a `signature` of the hash using eth_sign or equivalent signer.
4. Connect to the websocket with the url: `wss://api.derivadex.com?nonce=[nonce]&signature=[signature]`

An example Python implementation is displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.


# Commands
The websocket API offers commands for placing and canceling orders, as well as withdrawals. Since commands modify system state, these requests must include an [EIP-712 signature](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md). Examples are included in the [sample code](samples.md).

## Place order

### Request
> Request format (JSON)
```json
{
	"request": {
		"t": "Order",
		"c": {
			"traderAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
			"symbol": "ETHPERP",
			"strategy": "main",
			"side": "Bid",
			"orderType": "Limit",
			"requestId": "0x0000000000000000000000000000000000000000000000000000000000000001",
			"amount": 10,
			"price": 497.97,
			"stopPrice": 0,
			"signature": "0x"
		}
	}
}
```

You can place new orders by specifying specific attributes in the `Order` command's request payload. These requests are subject to a set of validations.

type | field | description 
------ | ---- | -------
address_s | traderAddress | Trader's Ethereum address (same as the one that facilitated the deposit)
string  | symbol | Name of the market to trade. Currently, this is limited to 'ETHPERP', but new symbols are coming soon! 
string | strategy | Name of the cross-margined strategy this trade belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon!
string | side | Side of trade, either `Bid` (buy/long) or an `Ask` (sell/short)
string | orderType | Order type, either `Limit` or `Market`. Other order types coming soon!
bytes32_s | requestId | An incrementing numeric identifier for this request that is unique per user for all time
decimal | amount | The order amount/size requested
decimal | price | The order price
decimal | stopPrice | Currently, always 0 as stops are not implemented.
bytes_s | signature | EIP-712 signature of the order placement intent


### Response
> Receipt (success) format (JSON)
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

A place order `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Received` or `Error`.
 
A successful `command` returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

type | field | description
------ | ---- | -----------
bytes32_s | requestId | The requestId supplied in the initial request - can be used to correlate requests with receipts
?? | requestIndex | A ticket number which guarantees fair sequencing
bytes_s | enclaveSignature | An Operator's signature which proves secure handling of the request 

> Receipt (error) format (JSON)
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
An erroneous command returns an `Error` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | msg | Error message


## Cancel order
> Request format (JSON)
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

You can cancel existing orders by specifying specific attributes in the `CancelOrder` command's request payload.

type | field | description
-----|---- | -----------------------
string | symbol | Currently always 'ETHPERP'. New symbols coming soon! 
bytes32_s | orderHash| The hash of the order being canceled.
bytes32_s | requestId | An incrementing numeric identifier for this request that is unique per user for all time
bytes_s | signature | EIP-712 signature

As described in the `Signatures & hashing` section, the `orderHash` is something that you construct client-side prior to submitting the order to the exchange. In this regard, you have the `orderHash` for each order you submit irrespective of acknowledgement from the exchange. You can determine which ones have actually been received by the exchange by filtering all the ones you have sent by using the `requestId` field in the successful/received responses from the place order `command`. As an alternate/additional reference, the `OrdersUpdate` `subscription` (see below for further information) includes the `orderHash` field for any open orders you have.

### Response
> Receipt (success) format (JSON)
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

A cancel order `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Received` or `Error`.
 
A successful `command` returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

type | field | description
------ | ---- | -----------
bytes32_s | requestId | The requestId supplied in the initial request - can be used to correlate requests with receipts
?? | requestIndex | A ticket number which guarantees fair sequencing
bytes_s | enclaveSignature | An Operator's signature which proves secure handling of the request 

> Receipt (error) format (JSON)
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
An erroneous command returns an `Error` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | msg | Error message


## Withdraw
> Request format (JSON)
```json
{
  "request": {
    "t": "Withdraw",
    "c": {
      "traderAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
      "strategyId": "main",
      "currency": "0x41082c820342539de44c1b404fead3b4b39e15d6",
      "amount": 440.32,
      "requestId": "0x0000000000000000000000000000000000000000000000000000000000000003",
      "signature": "0x89fb49d2d125adfef56328ee7367d23d1642d2fb6cfdf8843fa94ae7eac9ab23"
    }
  }
}
```

You can signal withdrawal intents to the Operators by specifying specific attributes in the `Withdraw` command's request payload. Withdrawal is a 2-step process: submitting a withdrawal intent, and performing a smart contract withdrawal. Once a withdrawal intent is initiated, you won't be able to trade with the collateral you are attempting to withdraw. You will only be able to formally initiate a smart contract withdrawal/token transfer once the epoch in which you signal your withdrawal desire has concluded. 

type | field | description
---- | --- | -----------------
address_s | traderAddress | Trader's Ethereum address (same as the one that facilitated the deposit)
string | strategyId | Name of the cross-margined strategy this withdrawal belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon!
address_s | currency | ERC-20 token address being withdrawn
decimal | amount | Amount withdrawn (be sure to use the grains format specific to the collateral token being used (e.g. if you wanted to withdraw 1 USDC, you would enter 1000000 since the USDC token contract has 6 decimal places)
bytes32_s | requestId | An incrementing numeric identifier for this request that is unique per user for all time
bytes_s | signature | EIP-712 signature

### Response
> Receipt (success) format (JSON)
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

A withdraw `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Received` or `Error`.
 
A successful `command` returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

type | field | description
------ | ---- | -----------
bytes32_s | requestId | The requestId supplied in the initial request - can be used to correlate requests with receipts
?? | requestIndex | A ticket number which guarantees fair sequencing
bytes_s | enclaveSignature | An Operator's signature which proves secure handling of the request 

> Receipt (error) format (JSON)
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
An erroneous command returns an `Error` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | msg | Error message

# Subscriptions

You can subscribe to two different kinds of feeds corresponding to a specific trader's account or market data with `SubscribeAccount` or `SubscribeMarket` requests, respectively.

## Account

### Request
> Request format (JSON)
```json
{
  "request": {
    "t": "SubscribeAccount",
    "c": {
      "trader": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
      "strategies": ["main"],
      "events": ["StrategyUpdate", "OrdersUpdate", "PositionUpdate"]
    }
  }
}
```

You can subscribe to data feeds corresponding to events for a particular trader's Ethereum address. The possible account events you can subscribe to are `StrategyUpdate`, `OrdersUpdate`, and `PositionUpdate`.

type | field | description
-----|----- | ---------------------
address_s  | trader | Trader's Ethereum address (same as the one that facilitated the deposit)
string[] | strategies | Strategies being subscribed to. Currently, the only supported strategy is `main`, but support for multiple strategies is coming soon!
string[] | events | Events being subscribed to. This can be one or more of `StrategyUpdate`, `OrdersUpdate`, and `PositionUpdate` 

### Response
> Receipt (success) format (JSON)
```json
{
  "receipt": {
    "t": "Subscribed",
    "c": {
      "msg": "Subscribed to [StrategyUpdate | OrdersUpdate | PositionUpdate] for ETHPERP"
    }
  }
}
```

Each `subscription` returns a receipt, which confirms that an Operator has received the `subscription` request. The receipt `type` will be either `Subscribed` or `Error`.

A successful `subscription` returns a `Subscribed` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | msg | Success message 

> Receipt (error) format (JSON)
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

An erroneous `subscription` returns an `Error` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | msg | Error message 

Each of the market `subscription` events will be discussed below individually. 

Each subscription event will be discussed below individually. 

## Strategy update (account)
> Partial response (JSON)
```json
{
	"t": "StrategyUpdate",
	"e": "Partial",
	"c": [{
		"trader": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
		"strategy": "main",
		"maxLeverage": "20",
		"freeCollateral": {
          "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48": "1000"
        },
		"frozenCollateral": {
          "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48": "0"
        },
		"createdAt": "2021-03-23T20:03:45.850Z"
	}]
}
```

> Update response (JSON)
```json
{
	"t": "StrategyUpdate",
	"e": "Update",
	"c": [{
		"trader": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
		"strategy": "main",
		"maxLeverage": "20",
		"freeCollateral": {
          "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48": "2000"
        },
		"frozenCollateral": {
          "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48": "0"
        },
		"frozen": false,
		"createdAt": "2021-03-23T20:03:45.850Z"
	}]
}
```

You can subscribe to updates to their strategy with the `StrategyUpdate` event. A strategy is a cross-margined account in which you can deposit/withdraw collateral and make trades. Strategies are siloed off from one another. Any updates to the free or frozen collaterals you have (whether it's from deposits, withdrawals, or realized PNL) will be registered in this feed.

Upon subscription, you will receive a `Partial` back, containing a snapshot of your strategy at that moment in time. From then on, you will receive streaming `Update` messages as appropriate. 

For the response, an array of updates per strategy you have subscribed to is emitted, with each update defined as follows:

type | field | description
-----|----- | ---------------------
address_s  | trader | Trader's Ethereum address (same as the one requested)
string_s   | strategy | Strategy being subscribed to. Currently, the only supported strategy is `main`, but support for multiple strategies is coming soon!
int_s   | maxLeverage | Maximum leverage for strategy, which impacts the maintenance margin ratio associated to any given trader
<address_s, decimal_s>  | freeCollateral | Collateral (on a per token basis) available for trading (not to be confused with the `Free Collateral` displayed on the UI, which is the collateral available to a trader wishing to signal a withdraw intent)
<address_s, decimal_s>  | frozenCollateral | Collateral (on a per token basis) available for a smart contract withdrawal, but not for trading, since the Operators have received the withdraw intent)
bool     | frozen | Whether the account and its collateral is frozen or not

## Orders update (account)
> Partial response (JSON)
```json
{
	"t": "OrdersUpdate",
	"e": "Partial",
	"c": [{
		"orderHash": "0x946e4bedf2dd87e5380f32a18d1af19adb4d7ecec3a8a346cb641adc5201e53e",
		"traderAddress": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
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

> Update response (JSON)
```json
{
	"t": "OrdersUpdate",
	"e": "Update",
	"c": [{
		"orderHash": "0x946e4bedf2dd87e5380f32a18d1af19adb4d7ecec3a8a346cb641adc5201e53e",
		"traderAddress": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
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

You can subscribe to updates to your open orders with the `OrdersUpdate` event. 

Upon subscription, you will receive a `Partial` back, containing a snapshot of all of your open orders at that moment in time. From then on, you will receive streaming/incremental `Update` messages with any changes to these orders (due to posting new orders, canceling existing orders, or orders that have been matched in partial or in full).

For the response, an array of updates per order is emitted, with each update defined as follows:

type | field | description 
------ | ---- | -------
bytes32_s | orderHash | Hash of the order that has changed or is new
address_s | traderAddress | Trader's Ethereum address associated with this order
string  | symbol | Name of the market this order belongs to. Currently, this is limited to 'ETHPERP', but new symbols are coming soon! 
string | strategy | Name of the cross-margined strategy this order belongs to. Currently, this is limited to the default `main` strategy, but support for multiple strategies is coming soon!
int_s | side | Side of order, either `0` (`Bid`) or an `1` (`Ask`)
int_s | orderType | Order type, either `0` (`Limit`) or `0` (`Market`). Other order types coming soon!
bytes32_s | requestId | Numeric identifier for the order. To clarify, this is NOT an identifier relating to this particular subscription/response, but rather the `requestId` field associated with this order at the time of placement.
decimal_s | amount | The original order amount/size requested
decimal_s | remainingAmount | The order amount/size remaining on the order book
decimal_s | price | The order price
decimal_s | stopPrice | Currently, always 0 as stops are not implemented.
bytes_s   | signature | EIP-712 signature
timestamp_s  | createdAt | Timestamp when order was initially created


## Positions update (account)

TBD


## Market

### Request
> Request format (JSON)
```json
{
  "request": {
    "t": "SubscribeMarket",
    "c": {
      "symbols": ["ETHPERP"],
      "events": ["OrderBookUpdate", "MarkPriceUpdate"]
    }
  }
}
```

You can subscribe to data feeds corresponding to events for the broader market. The possible market events you can subscribe to at this time are `OrderBookUpdate` and `MarkPriceUpdate`. 

type | field | description
-----|----- | ---------------------
string[] | symbols | Symbols being subscribed to. Currently, the only supported symbol is `ETHPERP`, but support for more symbols is coming soon!
string[] | events | Events being subscribed to. This can be one or more of `OrderBookUpdate` and `MarkPriceUpdate`

### Response
> Receipt (success) format (JSON)
```json
{
  "receipt": {
    "t": "Subscribed",
    "c": {
      "msg": "Subscribed to [OrderBookUpdate | MarkPriceUpdate] for ETHPERP"
    }
  }
}
```

Each `subscription` returns a receipt, which confirms that an Operator has received the `subscription` request. The receipt `type` will be either `Subscribed` or `Error`.

A successful `subscription` returns a `Subscribed` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | msg | Success message 

> Receipt (error) format (JSON)
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

An erroneous `subscription` returns an `Error` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | msg | Error message 

Each of the market `subscription` events will be discussed below individually. 

## Order book update (market)

> Partial response (JSON)
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

> Update response (JSON)
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

You can subscribe to updates to an L2-order book for any given market with the `OrderBookUpdate` event. As is typically seen, an order book is a price-time FIFO priority queue market place for you to interact with. Since this is an L2-aggregation, the response will collapse any given price level to the aggregate quantity at that level irrespective of the number of participants or the individual order details that comprise that price level.

Upon subscription, you will receive a `Partial` back, containing an L2-aggregate snapshot of the order book at that moment in time. From then on, you will receive streaming/incremental `Update` messages as appropriate containing only the aggregated price levels that are different (due to the placement of new orders, order cancellation, or liquidations/matches that have taken place). Updates will come in with a reference to the side of the order book and in the form of 2-item lists with the format `[price, new aggregate quantity]`. A new aggregate quantity of `0` indicates that the price level is now gone. 

For the response, an array of updates per order book is emitted, with each update defined as follows:

type | field | description
-----|----- | ---------------------
decimal_s[2][]  | bids | The price and corresponding updated aggregate quantity for the bids
decimal_s[2][]  | asks | The price and corresponding updated aggregate quantity for the asks
timestamp_i  | timestamp | Timestamp of order book partial/update
timestamp_s_i  | nonce | ???
decimal_s   | aggregationType | ???

### Maintaining local order book

To maintain a local order book, you can use a combination of the `Partial` snapshot and `Update` streaming messages. A sample algorithm is as follows:

1. Receive a `Partial` snapshot and use that as the starting point for the order book. Keep in mind that the response outputs the aggregated quantities at each price level for both the `bids` and `asks` sides of the order book.
2. For every `Update` streaming message, you will receive information for both sides of the order book regarding any price level that has changed. There will be no explicit indication if a level has been newly-formed, modified, or deleted. For any `[price, updated_aggregated_quantity]` you receive, update your local order book. If your local order book does not currently have an entry for this `price`, you know it is a new level and can set the quantity to `updated_aggregated_quantity`. If your local order book has the price level corresponding to `price`, adjust its quantity to `updated_aggregated_quantity` except in the scenario where `updated_aggregated_quantity` is `"0"`, in which case you can delete the level entirely.
3. If your connection drops or you believe you may have missed/mishandled an update, simply refetch the `Partial` as per Step 1, and proceed from there with Step 2.


## Mark price update (market)

> Partial response (JSON)
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

> Update response (JSON)
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

You can subscribe to updates to the mark price for any given market with the `MarkPriceUpdate` event. The mark price is an important concept on DerivaDEX as it is the price at which positions are marked when displaying unrealized PNL. Consequently, the mark price helps determine a strategy's margin fraction, thereby triggering liquidations when appropriate. The mark price is computed based on a 30s exponential moving average (often referred to as a 30s EMA) of the spread between the underlying price (a composite of several spot price feeds for the underlying asset) and the DerivaDEX order book itself. 

Upon subscription, you will receive a `Partial` back, containing a mark price snapshot at that moment in time. From then on, you will receive streaming/incremental `Update` messages as appropriate. 

For the response, an array of updates per symbol is emitted, with each update defined as follows:

type | field | description
-----|----- | ---------------------
timestamp_s  | createdAt | Timestamp of original mark price entry
timestamp_s  | updatedAt | Timestamp of mark price update
decimal_s | price | Mark price for the market
string  | symbol | Market subscribed to


# Validation

TBD


# Rate limits

The websocket API will (eventually) use tiers to determine rate limits for each ethereum account.
Rate limits restrict the number of "Commands" an account can place per minute.

Subscriptions are not rate limited.
