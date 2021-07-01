
# DerivaDEX API

DerivaDEX is a decentralized derivatives exchange that combines the performance of centralized exchanges with the security of decentralized exchanges.

DerivaDEX currently offers a public WebSocket API for traders and developers. The API will enable you to open and manage your positions via `commands`, and subscribe to market data via `subscriptions`. 

Find us online [Discord](https://discord.gg/a54BWuG) | [Telegram](https://t.me/DerivaDEX) | [Medium](https://medium.com/derivadex)


# Getting started

To begin interacting with the DerivaDEX ecosystem programmatically, you generally will want to follow these steps:
1. Deposit funds via Ethereum via an Ethereum client
2. Authenticate and connect to the websocket API
3. Submit and cancel orders via `commands`
4. Subscribe to the DerivaDEX state snapshot and transaction log

Additionally, you should familiarize yourself with the [DerivaDEX types terminology](#Types) and [hashing & signing schemes](#Signatures & hashing). 


## Types

The types labeled throughout this document in the request and response parameters may be familiar to those who have a background in Ethereum. In any case, please refer to the table below for additional information on the terminology used here. This reference in conjunction with the JSON samples should provide enough clarity:

type | description | example
-----| --------------------- | -----------
string | Literal of a sequence of characters surrounded by quotes | "ETHPERP"
address_s  | 20-byte "0x"-prefixed hexadecimal string literal (i.e. 40 digits long) corresponding to an `address` ETH type | "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
decimal | Numerical value with up to, but no more than 18 decimals of precision | 10.031500000000000000
decimal_s  | String representation of `decimal` | "10.031500000000000000"
bool  | Boolean value, either `true` or `false` | True
bytes32_s  | 32-byte "0x"-prefixed hexadecimal string literal (i.e. 64 digits long) corresponding to an `bytes32` ETH type | "0x0000000000000000000000000000000000000000000000000000000000000001"
timestamp_s_i | String representation of numerical UNIX timestamp representing the number of seconds since 1/1/1970 | "1616667513875"
timestamp_s | String representation representing the ISO 8601 UTC timestamp | "2021-03-25T10:38:09.503654"


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

> Domain separator for kovan. DO NOT modify these parameters.
```json
{
    "name": "DerivaDEX",
    "version": "1",
    "chainId":  42,
    "verifyingContract": "0x80ead6c2d69acc72dad76fb3151820a9b5d6a9e9"
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
        + encode_single('uint256', chain_id)
        + encode_single('address', verifying_contract)
    )
```

The `domain` is a mandatory field that allows for signature/hashing schemes on one dApp to be unique to itself from other dApps. All `commands` use the same `domain` specification. The parameters that comprise the domain are as follows:

type | field | description
-----|----- | ---------------------
name  | string | Name of the dApp or protocol
version  | string | Current version of the signing domain
chainId | uint256 | EIP-155 chain ID
verifyingContract | address | DerivaDEX smart contract's Ethereum address

To generate the `domain` struct hash, you must perform a series of encodings and hashings of the schema and contents of the `domain` specfication. You are welcome to do this however you like, but it must adhere to the standard eventually, otherwise the signature will not ultimately successfully recover. Example Solidity and Python reference implementations are displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.

### Message

The `message` field varies depending on the typed data you are signing, and is illustrated on a case-by-case basis below.

#### Place order

> Sample computation of place order message struct hash
```solidity
function compute_eip712_message_struct_hash(address _traderAddress, bytes32 _symbol, bytes32 _strategy, uint256 _side, uint256 _orderType, bytes32 _nonce, uint256 _amount, uint256 _price, uint256 _stopPrice) public view returns (bytes32) {
    // keccak-256 hash of the encoded schema for the order params struct
    bytes32 eip712SchemaHash = keccak256(abi.encodePacked(
        "OrderParams(",
        "address traderAddress,",
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
        uint256(_traderAddress),
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
from decimal import Decimal, ROUND_DOWN

def compute_eip712_message_struct_hash(trader_address: str, symbol: str, strategy: str, side: str, order_type: str, nonce: str, amount: Decimal, price: Decimal, stop_price: Decimal) -> bytes:
    # keccak-256 hash of the encoded schema for the place order command
    eip712_schema_hash = keccak(
        b"OrderParams("
        + b"address traderAddress,"
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
        eip712_schema_hash
        + encode_single('address', trader_address)
        + len(symbol).to_bytes(1, byteorder="little")
        + encode_single("bytes32", symbol.encode("utf8"))[:-1]
        + len(strategy).to_bytes(1, byteorder="little")
        + encode_single("bytes32", strategy.encode("utf8"))[:-1]
        + encode_single('uint256', order_side_to_int(side))
        + encode_single('uint256', order_type_to_int(order_type))
        + encode_single('bytes32', bytes.fromhex(nonce[2:]))
        + encode_single('uint256', to_base_unit_amount(amount, 18))
        + encode_single('uint256', to_base_unit_amount(price, 18))
        + encode_single('uint256', to_base_unit_amount(stop_price, 18))
    )
```

The parameters that comprise the `message` for the `command` to place an order are as follows:

type | field | description
-----|----- | ---------------------
traderAddress  | address | Trader's ETH address used to sign order intent
symbol  | bytes32 | 32-byte encoding of the symbol length and symbol this order is for. The `symbol` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly.
strategy | bytes32 | 32-byte encoding of the strategy length and strategy this order belongs to. The `strategy` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly. The `strategy` refers to the cross-margined bucket this trade belongs to. Currently, there is only the default `main` strategy, but support for multiple strategies is coming soon!
side | uint256 | An integer value either `0` (Bid) or `1` (Ask)
orderType | uint256 | An integer value either `0` (Limit) or `1` (Market)
nonce | bytes32 | 32-byte value (an incrementing numeric identifier that is unique per user for all time) resulting in uniqueness of order
amount | uint256 | Order amount (scaled up by 18 decimals; e.g. 2.5 => 2500000000000000000). The `amount` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.
price | uint256 | Order price (scaled up by 18 decimals; e.g. 2001.37 => 2001370000000000000000). The `price` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.
stopPrice | uint256 | Stop price (scaled up by 18 decimals). The `stopPrice` of the order you send to the API is a decimal, however for signing purposes, you must scale up by 18 decimals and convert to an integer.

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

type | field | description
-----|----- | ---------------------
symbol  | bytes32 | 32-byte encoding of the symbol length and symbol this order is for. The `symbol` of the order you send to the API is a string, however for signing purposes, you must bytes-encode and pad accordingly.
orderHash | bytes32 | 32-byte EIP-712 hash of the order at the time of placement
nonce | bytes32 | 32-byte value (an incrementing numeric identifier that is unique per user for all time) resulting in uniqueness of order cancellation


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

**Please feel free to use these ground truth samples to validate your EIP-712 hashing implementation for correctness.** For the following samples, assume a `chainId = 42` and `verifyingContract = 0x80ead6c2d69acc72dad76fb3151820a9b5d6a9e9`.

#### Place order

The following sample order placement data results in an EIP-712 hash of: `0xe1eaae57e5c637b25e588c7e6b92f34288a989e6a5cb6deab44db297d12fb852`.

field | value
-----|------
traderAddress  |  "0x4DbaEb213F91B022D0f238a2510380BE149b091a"
symbol  |  "ETHPERP"
strategy |  "main"
side |  "Bid"
orderType |  "Limit"
nonce |  "0x3136323338333730373432343739363630303000000000000000000000000000"
amount |  7.11
price |  2497.69
stopPrice |  0

#### Cancel order

The following sample cancellation data results in an EIP-712 hash of: `0x1eb6b329caf5f45b2402453f713c016abee1fa8a4df722ee5bafce58f0b8d053`.

field | value
-----|------
symbol  |  "ETHPERP"
orderHash |  "0xbc9ea45d017a031120db603f40ff3dc6003f41e531e69a7fdfe2ebc438032b7d"
nonce |  "0x3136323338333734323633363939323230303000000000000000000000000000"

## Sparse merkle tree (state)

DerivaDEX utilizes a [Sparse Merkle Tree (SMT)](https://medium.com/@kelvinfichter/whats-a-sparse-merkle-tree-acda70aeb837) in order to efficiently maintain the
state of the exchange at all times. Storing data on-chain (such as user's balances and positions), something most
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

Item | Discriminant
-----| ------------
Trader | 0
Strategy | 1
Position | 2
BookOrder | 3
Price | 4
InsuranceFund | 5
Stats | 6
Empty | 7

Each item type is described in detail below.

### Trader

A `Trader` contains information pertaining to a trader's free and frozen DDX balances,
and referral address.

#### Key encoding / decoding

> Key encoding / decoding (Python)
```python
from eth_abi.utils.padding import zpad32_right

def encode_key(trader_address: str, chain_discriminant: int):
    # ItemType.TRADER == 0
    return zpad32_right(
        ItemType.TRADER.to_bytes(1, byteorder="little")
        + chain_discriminant.to_bytes(1, byteorder="little")
        + bytes.fromhex(trader_address[2:])
    )

def decode_key(trader_key: bytes):
    # chain_discriminant, trader_address
    return int.from_bytes(trader_key[1:2], "little"), f"0x{trader_key[2:22].hex()}"
```

The location of a `Trader` leaf is determined by its key, which is encoded as
follows:

Bytes | Value
-----| ------------
0 | Trader discriminant
1 | Blockchain discriminant
[2, 21] | Trader's Ethereum address
[22, 31] | Zero-padding

The following sample `Trader` materials generates the following encoded key: `0x0000603699848c84529987e14ba32c8a66def67e9ece00000000000000000000`.

field | value
-----|------
trader_address  |  "0x603699848c84529987E14Ba32C8a66DEF67E9eCE"
chain_discriminant  |  0

#### Value definition

> Value encoding / decoding (Python)
```python
from decimal import Decimal, ROUND_DOWN
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(Decimal(".000000000000000001"), rounding=ROUND_DOWN)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, free_ddx_balance: Decimal, frozen_ddx_balance: Decimal, referral_address: str):
    # Trader item discriminant
    item_type = 0

    # Scale collateral amounts to DDX grains (to_base_unit_amount
    return encode_single(
        "(uint8,(uint128,uint128,address))",
        [
            self.item_type,
            [
                to_base_unit_amount(free_ddx_balance, 18),
                to_base_unit_amount(frozen_ddx_balance, 18),
                self.referral_address,
            ],
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (
        item_type,
        (free_ddx_balance, frozen_ddx_balance, referral_address),
    ) = decode_single(
        "(uint8,(uint128,uint128,address))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale collateral amounts from DDX grains
    return (
        to_unit_amount(free_ddx_balance, 18),
        to_unit_amount(frozen_ddx_balance, 18),
        referral_address,
    )
```

A `Trader` leaf holds the following data:

type | field | description
----- | ------------ | ---------
decimal | free_ddx_balance | DDX collateral available for staking/fees
decimal | frozen_ddx_balance | DDX collateral available for on-chain withdrawal
address_s | referral_address | Referral address pertaining to the Ethereum address who referred this trader (if applicable)

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint128,uint128,address))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the DDX grains conversion (10 ** 18) for certain
variables, is shown on the right.

The following sample `Trader` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000003635c9adc5dea000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a8dda8d7f5310e4a9e24f8eba77e091ac264f872`.

field | value
-----|------
free_ddx_balance  |  1000
frozen_ddx_balance  |  0
referral_address  |  "0xA8dDa8d7F5310E4A9E24F8eBA77E091Ac264f872"


### Strategy

A `Strategy` contains information pertaining to a trader's cross-margined strategy, such as their free and frozen collaterals and max leverage.

#### Key encoding / decoding

> Key encoding / decoding (Python)
```python
from eth_abi.utils.padding import zpad32_right, encode_single, decode_single
from web3.auto import w3

def generate_strategy_id_hash(strategy_id: str) -> bytes:
    # Get the last 4 bytes of the hash of the strategy id
    return w3.keccak(
        len(strategy_id).to_bytes(1, byteorder="little")
        + encode_single("bytes32", strategy_id.encode("utf8"))[:-1]
    )[:4]

def encode_key(trader_address: str, strategy_id: str, chain_discriminant: int):
    # ItemType.STRATEGY == 1
    return zpad32_right(
            ItemType.STRATEGY.to_bytes(1, byteorder="little")
            + chain_discriminant.to_bytes(1, byteorder="little")
            + bytes.fromhex(trader_address[2:])
            + generate_strategy_id_hash(strategy_id)
        )

def decode_key(strategy_key: bytes):
    # chain_discriminant, trader_address, strategy_id_hash
    return (
            int.from_bytes(strategy_key[1:2], "little"),
            f"0x{strategy_key[2:22].hex()}",
            f"0x{strategy_key[22:26].hex()}",
        )
```

The location of a `Strategy` leaf is determined by its key, which is encoded as
follows:

Bytes | Value
-----| ------------
0 | Strategy discriminant
1 | Blockchain discriminant
[2, 21] | Trader's Ethereum address
[22, 25] | Abbreviated hash of strategy ID
[26, 25] | Zero-padding

The following sample `Strategy` materials generates the following encoded key: `0x0100603699848c84529987e14ba32c8a66def67e9ece2576ebd1000000000000`.

field | value
-----|------
trader_address  |  "0x603699848c84529987E14Ba32C8a66DEF67E9eCE"
strategy_id  |  "main"
chain_discriminant  |  0

#### Value definition

> Value encoding / decoding (Python)
```python
from typing import Dict
from decimal import Decimal, ROUND_DOWN
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(Decimal(".000000000000000001"), rounding=ROUND_DOWN)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_base_unit_amount_list(vals, decimals):
    return [to_base_unit_amount(val, decimals) for val in vals]

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def to_unit_amount_list(vals, decimals):
    return [to_unit_amount(val, decimals) for val in vals]

def abi_encoded_value(self, strategy_id: str, free_collateral: Dict[str, Decimal], frozen_collateral: Dict[str, Decimal], max_leverage: int, frozen: bool):
    # Strategy item discriminant
    item_type = 1

    # Scale collateral amounts to DDX grains
    return encode_single(
        "((uint8,(bytes32,(address[],uint128[]),(address[],uint128[]),uint64,bool)))",
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
                        to_base_unit_amount_list(
                            list(free_collateral.values()), 18
                        ),
                    ],
                    [
                        list(frozen_collateral.keys()),
                        to_base_unit_amount_list(
                            list(frozen_collateral.values()), 18
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
        "((uint8,(bytes32,(address[],uint128[]),(address[],uint128[]),uint64,bool)))",
        w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale collateral amounts from DDX grains
    return (
        strategy_id[1 : 1 + strategy_id[0]].decode("utf8"),
        {
            k: to_unit_amount(v, 18)
            for k, v in zip(
                list(free_collateral_tokens), list(free_collateral_amounts)
            )
        },
        {
            k: to_unit_amount(v, 18)
            for k, v in zip(
                list(frozen_collateral_tokens), list(frozen_collateral_amounts)
            )
        },
        max_leverage,
        frozen,
    )
```

A `Strategy` leaf holds the following data:

type | field | description
----- | ------------ | ---------
string | strategy_id | Identifier for strategy (e.g. "main")
dict<address_s, decimal> | free_collateral | Mapping of collateral address to free collateral amount
dict<address_s, decimal> | frozen_collateral | Mapping of collateral address to frozen collateral amount
int | max_leverage | Maximum leverage strategy can take
bool | frozen | Whether the strategy is frozen or not (relevant for tokenization)

These contents are always stored in the tree in ABI-encoded form: `((uint8,(bytes32,(address[],uint128[]),(address[],uint128[]),uint64,bool)))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the DDX grains conversion (10 ** 18) for certain
variables, is shown on the right.

The following sample `Strategy` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000002000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000040046d61696e00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000000000016000000000000000000000000000000000000000000000000000000000000000140000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000001000000000000000000000000b69e673309512a9d726f87304c6984054f87a93b0000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000000000000000002a58743747cf74b400000000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000`.

field | value
-----|------
strategy_id  |  "main"
free_collateral  |  {"0xb69e673309512a9d726f87304c6984054f87a93b": 199971.08}
frozen_collateral  |  {}
max_leverage  |  20
frozen  |  False


### Position

A `Position` contains information pertaining to an open position.

#### Key encoding / decoding

> Key encoding / decoding (Python)
```python
from eth_abi.utils.padding import zpad32_right, encode_single, decode_single
from web3.auto import w3

def generate_strategy_id_hash(strategy_id: str) -> bytes:
    # Get the last 4 bytes of the hash of the strategy id
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
    # ItemType.POSITION == 2
    return (
            ItemType.POSITION.to_bytes(1, byteorder="little")
            + pack_bytes(symbol)
            + chain_discriminant.to_bytes(1, byteorder="little")
            + bytes.fromhex(trader_address[2:])
            + generate_strategy_id_hash(strategy_id)
        )

def decode_key(position_key: bytes):
    # symbol, chain_discriminant, trader_address, strategy_id_hash
    return (
            unpack_bytes(position_key[1:7]),
            int.from_bytes(position_key[7:8], "little"),
            f"0x{position_key[8:28].hex()}",
            f"0x{position_key[28:].hex()}",
        )
```

The location of a `Position` leaf is determined by its key, which is encoded as
follows:

Bytes | Value
-----| ------------
0 | Position discriminant
[1, 6] | Symbol (5-bit encoded/packed)
7 | Blockchain discriminant
[8, 27] | Trader's Ethereum address
[28, 31] | Abbreviated hash of strategy ID

The following sample `Position` materials generates the following encoded key: `0x0285225824040000603699848c84529987e14ba32c8a66def67e9ece2576ebd1`.

field | value
-----|------
trader_address  |  "0x603699848c84529987E14Ba32C8a66DEF67E9eCE"
strategy_id  |  "main"
symbol  |  "ETHPERP"
chain_discriminant  |  0

#### Value definition

> Value encoding / decoding (Python)
```python
from typing import Dict
from decimal import Decimal, ROUND_DOWN
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(Decimal(".000000000000000001"), rounding=ROUND_DOWN)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, side: int, balance: Decimal, avg_entry_price: Decimal):
    # Position item discriminant
    item_type = 2

    # Scale balance and average entry price to DDX grains
    return encode_single(
        "(uint8,(uint8,uint128,uint128))",
        [
            item_type,
            [
                side,
                to_base_unit_amount(balance, 18),
                to_base_unit_amount(avg_entry_price, 18),
            ],
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (item_type, (side, balance, avg_entry_price)) = decode_single(
        "(uint8,(uint8,uint128,uint128))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale balance and average entry price from DDX grains
    return (
        side, to_unit_amount(balance, 18), to_unit_amount(avg_entry_price, 18)
    )
```

A `Position` leaf holds the following data:

type | field | description
----- | ------------ | ---------
int | side | Side of position (`Long=1`, `Short=2`)
decimal | balance | Size of the position
decimal | average_entry_price | Average entry price of the position

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint8,uint128,uint128))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the DDX grains conversion (10 ** 18) for certain
variables, is shown on the right.

The following sample `Position` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000068155a43676e0000000000000000000000000000000000000000000000000000d4eff354906660000`.

field | value
-----|------
side  |  1
balance  |  120
avg_entry_price  |  245.5


### Book order

A `BookOrder` contains information pertaining to a maker order in the order book.

#### Key encoding / decoding

> Key encoding / decoding (Python)
```python
from eth_abi.utils.padding import zpad32_right, encode_single, decode_single
from web3.auto import w3

def generate_strategy_id_hash(strategy_id: str) -> bytes:
    # Get the last 4 bytes of the hash of the strategy id
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

Bytes | Value
-----| ------------
0 | Book order discriminant
[1, 6] | Symbol (5-bit encoded/packed)
[7, 31] | First 25 bytes of order's unique hash

The following sample `BookOrder` materials generates the following encoded key: `0x038522582404003d940b7e18acdf6c6f4740f7226245a796d53b6f2ffb9a8ca4`.

field | value
-----|------
symbol  |  "ETHPERP"
order_hash  |  "0x3d940b7e18acdf6c6f4740f7226245a796d53b6f2ffb9a8ca4ABABABABABABAB"

#### Value definition

> Value encoding / decoding (Python)
```python
from typing import Dict
from decimal import Decimal, ROUND_DOWN
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(Decimal(".000000000000000001"), rounding=ROUND_DOWN)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, side: int, amount: Decimal, price: Decimal, trader_address: str, strategy_id_hash: bytes):
    # BookOrder item discriminant
    item_type = 3

    # Scale amount and price to DDX grains
    return encode_single(
        "(uint8,(uint8,uint128,uint128,address,bytes32))",
        [
            self.item_type,
            [
                side,
                to_base_unit_amount(amount, 18),
                to_base_unit_amount(price, 18),
                trader_address,
                strategy_id_hash,
            ],
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (
        item_type,
        (side, amount, price, trader_address, strategy_id_hash),
    ) = decode_single(
        "(uint8,(uint8,uint128,uint128,address,bytes32))",
        w3.toBytes(hexstr=abi_encoded_value),
    )
    
    # Scale amount and price from DDX grains
    return (
        side,
        to_unit_amount(amount, 18),
        to_unit_amount(price, 18),
        trader_address,
        strategy_id_hash[:4],
    )
```

A `BookOrder` leaf holds the following data:

type | field | description
----- | ------------ | ---------
int | side | Side of order (`Bid=0`, `Ask=1`)
decimal | amount | Amount/size of order
decimal | price | Price the order has been placed at
address_s | trader_address | The order creator's Ethereum address
bytes | strategy_id_hash | First 4 bytes of strategy ID hash this order belongs to

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint8,uint128,uint128,address,bytes32))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the DDX grains conversion (10 ** 18) for certain
variables, is shown on the right.

The following sample `BookOrder` materials generates the following ABI-encoded value: `0x00000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001158e460913d0000000000000000000000000000000000000000000000000000d8d726b7177a80000000000000000000000000000603699848c84529987e14ba32c8a66def67e9ece2576ebd100000000000000000000000000000000000000000000000000000000`.

field | value
-----|------
side  |  0
amount  |  20
price  |  250
trader_address  |  "0x603699848c84529987E14Ba32C8a66DEF67E9eCE"
strategy_id_hash  |  "0x2576ebd1"



### Price

A `Price` contains information pertaining to a market's price checkpoint.

#### Key encoding / decoding

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
    # ItemType.PRICE == 4
    return zpad32_right(
            ItemType.PRICE.to_bytes(1, byteorder="little")
            + pack_bytes(symbol)
            # TODO (apalepu): Uncomment once this is added on Rust side
            # + index_price_hash[:26]
        )

def decode_key(price_key: bytes):
    # symbol
    return unpack_bytes(price_key[1:7])
```

The location of a `Price` leaf is determined by its key, which is encoded as
follows:

Bytes | Value
-----| ------------
0 | Price discriminant
[1, 6] | Symbol (5-bit encoded/packed)
[7, 31] | 25 bytes of index price's unique hash

The following sample `Price` materials generates the following encoded key: `0x0485225824040000000000000000000000000000000000000000000000000000`.

field | value
-----|------
symbol  |  "ETHPERP"
index_price_hash  |  "0x3d940b7e18acdf6c6f4740f7226245a796d53b6f2ffb9a8ca4ABABABABABABAB"

#### Value definition

> Value encoding / decoding (Python)
```python
from typing import Dict
from decimal import Decimal, ROUND_DOWN
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(Decimal(".000000000000000001"), rounding=ROUND_DOWN)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, index_price: Decimal, index_price_hash: bytes, ema: Decimal):
    # Price item discriminant
    item_type = 4

    # Scale index price and ema to DDX grains
    price_encoding = encode_single(
        "(uint8,(uint128,bytes32,uint128))",
        [
            self.item_type,
            [
                to_base_unit_amount(index_price, 18),
                index_price_hash,
                to_base_unit_amount(abs(ema), 18),
            ],
        ],
    )
    if self.ema < 0:
        price_encoding_byte_array = bytearray(price_encoding)
        price_encoding_byte_array[-17] = 1
        price_encoding = bytes(price_encoding_byte_array)

    return price_encoding

def abi_decoded_value(abi_encoded_value: str):
    price_encoding_byte_array = bytearray(bytes.fromhex(abi_encoded_value[2:]))
    multiplier = -1 if price_encoding_byte_array[-17] == 1 else 1
    price_encoding_byte_array[-17] = 0
    abi_encoded_value = bytes(price_encoding_byte_array).hex()

    (item_type, (index_price, index_price_hash, ema)) = decode_single(
        "(uint8,(uint128,bytes32,uint128))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale index price and ema from DDX grains
    return (
        to_unit_amount(index_price, 18),
        index_price_hash,
        to_unit_amount(ema * multiplier, 18),
    )
```

A `Price` leaf holds the following data:

type | field | description
----- | ------------ | ---------
decimal | index_price | Composite index price perpetual is tracking
bytes | index_price_hash | Unique hash of index price
decimal | ema | EMA component of price, tracking the difference between the DerivaDEX order book price and the underlying

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint128,bytes32,uint128))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the DDX grains conversion (10 ** 18) for certain
variables, is shown on the right. It's demonstrated in the code sample, but it's worth highlighting that
the `ema` component, which can be a negative value, is ABI-encoded to a 32-byte value where the first 16 bytes will either be
`0x00000000000000000000000000000000` (positive) or `0x00000000000000000000000000000001` (negative), and the next 16 bytes
is the absolute value of the `ema`.

The following sample `Price` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000d8d726b7177a800003ad520dd6051f521d43b7b834450b663b7782df758823a8e6a9845cdc86139690000000000000000000000000000000000000000000000000000000000000000`.

field | value
-----|------
index_price  |  250
index_price_hash  |  '0x3ad520dd6051f521d43b7b834450b663b7782df758823a8e6a9845cdc8613969'
ema  |  0


### Insurance fund

An `InsuranceFund` contains information pertaining to the insurance fund's capitalization.

#### Key encoding / decoding

> Key encoding / decoding (Python)
```python
from eth_abi.utils.padding import zpad32_right

def encode_key():
    # ItemType.INSURANCE_FUND == 5
    return zpad32_right(
        ItemType.INSURANCE_FUND.to_bytes(1, byteorder="little")
        + "OrganicInsuranceFund".encode("utf8")
    )
```

The location of an `InsuranceFund` leaf is determined by its key, which is encoded as
follows:

Bytes | Value
-----| ------------
0 | Insurance fund discriminant
[1, 20] | "OrganicInsuranceFund" bytes-encoded
[21, 31] | Zero-padding

The `InsuranceFund` leaf is located at the encoded key: `0x054f7267616e6963496e737572616e636546756e640000000000000000000000`.

#### Value definition

> Value encoding / decoding (Python)
```python
from typing import Dict
from decimal import Decimal, ROUND_DOWN
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(Decimal(".000000000000000001"), rounding=ROUND_DOWN)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, capitalization: Dict[str, Decimal]):
    # InsuranceFund item discriminant
    item_type = 5

    # Scale collateral amounts to DDX grains
    return encode_single(
        "((uint8,(address[],uint128[])))",
        [
            [
                self.item_type,
                [
                    list(capitalization.keys()),
                    to_base_unit_amount_list(
                        list(capitalization.values()), 18
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

    # Scale collateral amounts from DDX grains
    return cls(
        {
            k: to_unit_amount(v, 18)
            for k, v in zip(
                list(capitalization_tokens), list(capitalization_amounts)
            )
        },
    )
```

An `InsuranceFund` leaf holds the following data:

type | field | description
----- | ------------ | ---------
dict<address_s, decimal> | capitalization | Mapping of collateral address to organic insurance fund capitalization

These contents are always stored in the tree in ABI-encoded form: `((uint8,(address[],uint128[])))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the DDX grains conversion (10 ** 18) for certain
variables, is shown on the right.

The following sample `InsuranceFund` materials generates the following ABI-encoded value: `0x0000000000000000000000000000000000000000000000000000000000000020000000000000000000000000000000000000000000000000000000000000000500000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000`.

field | value
-----|------
capitalization  |  {}

### Stats

An `Stats` contains data such as volume info for trade mining for any given trader.

#### Key encoding / decoding

> Key encoding / decoding (Python)
```python
from eth_abi.utils.padding import zpad32_right

def encode_key(trader_address: str):
    # ItemType.STATS == 6
    return zpad32_right(
            ItemType.STATS.to_bytes(1, byteorder="little")
            + chain_discriminant.to_bytes(1, byteorder="little")
            + bytes.fromhex(trader_address[2:])
        )

def decode_key(book_order_key: bytes):
    # chain_discriminant, trader_address
    return int.from_bytes(stats_key[1:2], "little"), f"0x{stats_key[2:22].hex()}"
```

The location of a `Stats` leaf is determined by its key, which is encoded as
follows:

Bytes | Value
-----| ------------
0 | Stats discriminant
1 | Blockchain discriminant
[2, 21] | Trader's Ethereum address
[22, 31] | Zero-padding

The following sample `Stats` materials generates the following encoded key: `0x0600603699848c84529987e14ba32c8a66def67e9ece00000000000000000000`.

field | value
-----|------
trader_address  |  "0x0x603699848c84529987E14Ba32C8a66DEF67E9eCE"

#### Value definition

> Value encoding / decoding (Python)
```python
from typing import Dict
from decimal import Decimal, ROUND_DOWN
from eth_abi import encode_single, decode_single
from web3.auto import w3

def round_to_unit(val):
    return val.quantize(Decimal(".000000000000000001"), rounding=ROUND_DOWN)

def to_base_unit_amount(val, decimals):
    return int(round_to_unit(val) * 10 ** decimals)

def to_unit_amount(val, decimals):
    return Decimal(str(val)) / 10 ** decimals

def abi_encoded_value(self, maker_volume: Decimal, taker_volume: Decimal):
    # Stats item discriminant
    item_type = 6

    # Scale volume amounts to DDX grains
    return encode_single(
        "(uint8,(uint128,uint128))",
        [
            item_type,
            [
                to_base_unit_amount(maker_volume, 18),
                to_base_unit_amount(taker_volume, 18),
            ],
        ],
    )

def abi_decoded_value(abi_encoded_value: str):
    (item_type, (maker_volume, taker_volume)) = decode_single(
        "(uint8,(uint128,uint128))", w3.toBytes(hexstr=abi_encoded_value),
    )

    # Scale volumes from DDX grains
    return (to_unit_amount(maker_volume, 18), to_unit_amount(taker_volume, 18))
```

A `Stats` leaf holds the following data:

type | field | description
----- | ------------ | ---------
decimal | maker_volume | Maker volume of trader during this trade mining epoch
decimal | taker_volume | Taker volume of trader during this trade mining epoch

These contents are always stored in the tree in ABI-encoded form: `(uint8,(uint128,uint128))`. Meaning,
you will want to decode the contents into a more suitable form for your purposes as necessary (for example loading
data from a snapshot of the state), and will need to encode it back again if you are saving it back into the tree.
A sample of Python code that derives this ABI-encoding, including the DDX grains conversion (10 ** 18) for certain
variables, is shown on the right.

The following sample `Stats` materials generates the following ABI-encoded value: `0x000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000340aad21b3b70000000000000000000000000000000000000000000000000000340aad21b3b700000`.

field | value
-----|------
maker_volume | 60
capitalization | 60


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
w3 = Web3(Web3.HTTPProvider("https://kovan.infura.io/v3/<your_api_key>"))

# Open up the ABI shown above saved at a file location, in this case `trader_abi.json` is the filename
with open('trader_abi.json') as f:
    trader_abi = json.load(f)
    
    # Create an trader_abi contract wrapper
    trader_contract = w3.eth.contract(address=Web3.toChecksumAddress('0x80ead6c2d69acc72dad76fb3151820a9b5d6a9e9'), abi=trader_abi)
    
    # Deposit 1000 USDC
    trader_contract.functions.deposit('0xc4f290a59d66a2e06677ed27422a1106049d9e72', encode_single("bytes32", 'main'.encode("utf8")), 1000000000).transact()
```

DerivaDEX is a decentralized exchange. As such, trading is non-custodial. Users are responsible for their own funds, which are deposited to the DerivaDEX smart contracts on Ethereum for trading. 

To deposit funds on DerivaDEX, first ensure that you have created an Ethereum account. The deposit interaction is between a user and the DerivaDEX smart contracts. To be more explicit, you will not be utilizing the WebSocket API to facilitate a deposit. The DerivaDEX Solidity smart contracts adhere to the [Diamond Standard](https://medium.com/derivadex/the-diamond-standard-a-new-paradigm-for-upgradeability-569121a08954). The `deposit` smart contract function you will need to use is located in the `Trader` facet, at the address of the main `DerivaDEX` proxy contract (`0x80ead6c2d69acc72dad76fb3151820a9b5d6a9e9`).

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
from web3.auto import w3
from eth_account.messages import encode_defunct
from eth_abi import encode_single
import time

def get_url(self) -> str:
    # Initialize a Web3 account from a private key
    web3_account = w3.eth.account.from_key("<private_key>")
    
    # Retrieve current UNIX time in nanoseconds to derive a unique, monotonically-increasing nonce
    nonce = str(time.time_ns())

    # abi.encode(['bytes32'], [nonce])
    encoded_nonce = encode_single("bytes32", nonce.encode("utf8")).hex()
    
    # Prefix the encoded nonce with ETH prefix to prepare for signing
    prefixed_message = encode_defunct(hexstr=encoded_nonce)

    # Hash prefixed message and sign the result
    signed_message = web3_account.sign_message(prefixed_message)

    # Construct WS connection url with format
    # f"wss://beta.derivadex.io/trader/v1?token=<encoded_nonce><signature>"
    # encoded_nonce = 32-byte hexstr without 0x prefix (i.e. 64 chars)
    # signature = 65-byte hexstr without 0x prefix (i.e. 130 chars)
    return f"wss://beta.derivadex.io/trader/v1?token={encoded_nonce}{signed_message.signature.hex()[2:]}"
```

Addresses used to connect to the websocket API must _already_ have funds deposited. If you haven't, [do that first](#Making-a-deposit).

The steps to connect are:

1. Generate a string (`nonce`) that is greater than the last value used for this address. Requests with a nonce less than or equal to the previous value will be rejected. For this reason, we **STRONGLY** recommend using [Unix time](https://en.wikipedia.org/wiki/Unix_time) in nanoseconds, but users may opt to maintain their own sequence counter if they really know what they are doing and are comfortable with the risk. 
2. ABI encode the `nonce` to a 32-byte value ('bytes32').
3. `keccak256` hash the encoded nonce without the ETH prefix.
4. Prefix the hash from above with the ETH prefix.
5. Generate a `signature` of the hash of the prefixed message from above using eth_sign or equivalent signer.
6. Connect to the websocket with the url: `wss://beta.derivadex.io/trader/v1?token=[encoded_nonce][signature]`

An example Python implementation is displayed on the right, but feel free to utilize whichever language, tooling, and abstractions you see fit.


# Commands
The websocket API offers commands for placing and canceling orders, as well as withdrawals. Since commands modify system state, these requests must include an [EIP-712 signature](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md). Examples are included in the [sample code](samples.md).

## Place order

### Request
> Request format (JSON)
```json
{
	"t": "Request",
	"c": {
		"t": "Order",
		"c": {
			"traderAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
			"symbol": "ETHPERP",
			"strategy": "main",
			"side": "Bid",
			"orderType": "Limit",
			"nonce": "0x3136313839303336353634383833373230303000000000000000000000000000",
			"amount": 10,
			"price": 487.50,
			"stopPrice": 0,
			"signature": "0xd5a1ca6d40a030368710ab86d391e5d16164ea16d2c809894eefddd1658bb08c6898177aa492d4d45272ee41cb40f252327a23e8d1fc2af6904e8860d3f72b3b1b"
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
bytes32_s | nonce | An incrementing numeric identifier for this request that is unique per user for all time
decimal | amount | The order amount/size requested
decimal | price | The order price (If `orderType` is `Market`, this must be set to `0`)
decimal | stopPrice | Currently, always set to `0` as stops are not implemented.
bytes_s | signature | EIP-712 signature of the order placement intent

To be more explicit, all of these fields must be passed in, even if not all of the fields apply due to certain functionalities not currently implemented (i.e. stops) or the fact that prices aren't applicable in the case of market orders. Please follow the guidelines specified in the table above around these conditions.


### Response
> Receipt (success) format (JSON)
```json
{
	"t": "Sequenced",
	"c": {
		"sender": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
		"nonce": "0x3136313839303336353634383833373230303000000000000000000000000000",
		"requestIndex": 13280,
		"enclaveSignature": "0xd42d6ec7851ff6e0ebac80cd087c20b9e3c2df8ddbeeed59914bec25dec1091d7ed0551bc815af1c8fc1e7438f58121b2ce0aa23d1fba0cd2413a6b16a7c2cb11c"
	}
}
```

A place order `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Sequenced` or `Error`.
 
A successful `command` returns a `Sequenced` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

type | field | description
------ | ---- | -----------
bytes32_s | requestId | The requestId supplied in the initial request - can be used to correlate requests with receipts
int | requestIndex | A ticket number which guarantees fair sequencing
bytes_s | enclaveSignature | An Operator's signature which proves secure handling of the request 

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

type | field | description
------ | ---- | -----------
string | msg | Error message


## Cancel order

### Request
> Request format (JSON)
```json
{
	"t": "Request",
	"c": {
		"t": "CancelOrder",
		"c": {
			"symbol": "ETHPERP",
			"orderHash": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
			"nonce": "0x3136313839303336353634383833373230303000000000000000000000000000",
			"signature": "0xd5a1ca6d40a030368710ab86d391e5d16164ea16d2c809894eefddd1658bb08c6898177aa492d4d45272ee41cb40f252327a23e8d1fc2af6904e8860d3f72b3b1b"
		}
	}
}
```

You can cancel existing orders by specifying specific attributes in the `CancelOrder` command's request payload.

type | field | description
-----|---- | -----------------------
string | symbol | Currently always 'ETHPERP'. New symbols coming soon! 
bytes32_s | orderHash| The hash of the order being canceled.
bytes32_s | nonce | An incrementing numeric identifier for this request that is unique per user for all time
bytes_s | signature | EIP-712 signature

As described in the `Signatures & hashing` section, the `orderHash` is something that you construct client-side prior to submitting the order to the exchange. In this regard, you have the `orderHash` for each order you submit irrespective of acknowledgement from the exchange. You can determine which ones have actually been received by the exchange by filtering all the ones you have sent by using the `nonce` field in the successful/received responses from the place order `command`.

### Response
> Receipt (success) format (JSON)
```json
{
	"t": "Sequenced",
	"c": {
		"sender": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
		"nonce": "0x3136313839303336353634383833373230303000000000000000000000000000",
		"requestIndex": 13280,
		"enclaveSignature": "0xd42d6ec7851ff6e0ebac80cd087c20b9e3c2df8ddbeeed59914bec25dec1091d7ed0551bc815af1c8fc1e7438f58121b2ce0aa23d1fba0cd2413a6b16a7c2cb11c"
	}
}
```

A cancel order `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Sequenced` or `Error`.
 
A successful `command` returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

type | field | description
------ | ---- | -----------
bytes32_s | requestId | The requestId supplied in the initial request - can be used to correlate requests with receipts
int | requestIndex | A ticket number which guarantees fair sequencing
bytes_s | enclaveSignature | An Operator's signature which proves secure handling of the request 

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

type | field | description
------ | ---- | -----------
string | msg | Error message


## Withdraw

### Request
> Request format (JSON)
```json
{
	"t": "Request",
	"c": {
		"t": "Withdraw",
		"c": {
			"traderAddress": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
			"strategyId": "main",
			"currency": "0x41082c820342539de44c1b404fead3b4b39e15d6",
			"amount": 440.32,
			"nonce": "0x3136313839303336353634383833373230303000000000000000000000000000",
			"signature": "0xd5a1ca6d40a030368710ab86d391e5d16164ea16d2c809894eefddd1658bb08c6898177aa492d4d45272ee41cb40f252327a23e8d1fc2af6904e8860d3f72b3b1b"
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
	"t": "Sequenced",
	"c": {
		"sender": "0x603699848c84529987E14Ba32C8a66DEF67E9eCE",
		"nonce": "0x3136313839303336353634383833373230303000000000000000000000000000",
		"requestIndex": 13280,
		"enclaveSignature": "0xd42d6ec7851ff6e0ebac80cd087c20b9e3c2df8ddbeeed59914bec25dec1091d7ed0551bc815af1c8fc1e7438f58121b2ce0aa23d1fba0cd2413a6b16a7c2cb11c"
	}
}
```

A withdraw `command` returns a response receipt, which confirms that an Operator has received the request and has sequenced it for processing. The receipt `type` will be either `Received` or `Error`.
 
A successful `command` returns a `Received` receipt from the Operator. DerivaDEX Operators execute code within a trusted execution environment. The enclaveSignature affirms that this environment has the security guarantees associated with Intel SGX TEEs.

type | field | description
------ | ---- | -----------
bytes32_s | requestId | The requestId supplied in the initial request - can be used to correlate requests with receipts
int | requestIndex | A ticket number which guarantees fair sequencing
bytes_s | enclaveSignature | An Operator's signature which proves secure handling of the request 

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

type | field | description
------ | ---- | -----------
string | msg | Error message

# Subscriptions

You can subscribe to the whole DerivaDEX state and all of the data via a subscription request.

## State and transaction log

DerivaDEX maintains its state in a Sparse Merkle Tree (SMT), which consists of leaves of various types.
The SMT is updated when any state-changing transaction takes place. Any client that subscribes to
this state and transaction log can efficiently track everything that happens on the exchange in
full. This is an exceptionally powerful subscription endpoint, allowing consumers to:
1) validate the honesty and integrity of the exchange's operations
2) ensure they are verifiably up-to-date with the latest exchange data

### Subscription request
> Request format (JSON)
```json
{
	"t": "SubscribeMarket",
	"c": {
		"events": ["TxLogUpdate"]
	}
}
```

type | field | description
-----|----- | ---------------------
string[] | events | Events being subscribed to. There is only one event that can be subscribed to, the `TxLogUpdate` event. 

### Subscription response
> Receipt (success) format (JSON)
```json
{
    "t": "Subscribed",
    "c": {
        "message": "Subscribed to [TxLogUpdate] for 0x603699848c84529987E14Ba32C8a66DEF67E9eCE"
    }
}
```

Each `subscription` returns a receipt, which confirms that an Operator has received the `subscription` request. The receipt `type` will be either `Subscribed` or `Error`.

A successful `subscription` returns a `Subscribed` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | message | Success message 

> Receipt (error) format (JSON)
```json
{
    "t": "Error",
    "c": {
        "message": "Invalid subscription"
    }
}
```

An erroneous `subscription` returns an `Error` receipt from the Operator.

type | field | description
------ | ---- | -----------
string | message | Error message 


### Event response
> Partial response (JSON)
```json
{
	"t": "TxLogUpdate",
	"e": "Partial",
	"c": [
		[{
			"lowEpochId": "3285",
			"highEpochId": "3285",
			"smtKey": "0x03852258240400c873ea38e15d879465347afac4cdfb3a6e0317e6d89e29ffa9",
			"smtHash": "0x1efadbf30bbd972a82e7e6eb69d02426905ecaed708c906a579e37fe85309ade",
			"smtValue": "0x0000000000000000000000000000000000000000000000000000000000000003000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000098eee79d10ce0000000000000000000000000000000000000000000000000079bffc1fc3ca0d0000000000000000000000000000e36ea790bc9d7ab70c55260c66d52b1eca985f842576ebd100000000000000000000000000000000000000000000000000000000"
		}, {
			"lowEpochId": "3285",
			"highEpochId": "3285",
			"smtKey": "0x038522582404009fdc11a1a8b36a931888d77829719c950ee22fa5dbb61a9010",
			"smtHash": "0xa1efd81e2e9e48bb4a87c3e12956e73e533dbced114930f6f9c8fad9c9703891",
			"smtValue": "0x00000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000ca5690c079320000000000000000000000000000000000000000000000000079fefd71b5fa530000000000000000000000000000e36ea790bc9d7ab70c55260c66d52b1eca985f842576ebd100000000000000000000000000000000000000000000000000000000"
		}, {
			"lowEpochId": "3285",
			"highEpochId": "3285",
			"smtKey": "0x03852258240400d29e5da477e70f8ae124adf64e3aacad16e047afbb72133e02",
			"smtHash": "0x03860f96f101ea693aed84948af1624a5fd75c33492aa3165feff26ad62f7655",
			"smtValue": "0x00000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000693191d6e576000000000000000000000000000000000000000000000000007a5d5be5aed2fb0000000000000000000000000000e36ea790bc9d7ab70c55260c66d52b1eca985f842576ebd100000000000000000000000000000000000000000000000000000000"
		}],
		[{
			"epochId": "3286",
			"txOrdinal": "0",
			"requestIndex": "180842",
			"stateRootHash": "0xc480af2c9623e3571de640112a4b94a29406199359508a6ff1c0c4a8b07af7d1",
			"eventKind": 9,
			"createdAt": "2021-06-30T20:32:38.018Z",
			"signature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
			"event": {
				"ETHPERP": {
					"ema": "0.325889706776203498",
					"indexPrice": "2263.343",
					"indexPriceHash": "0x5f994903e82e60704f1d9fbb04315ad248fc7bb4bf739d32c4d53157beb079e0"
				}
			}
		}]
	]
}
```

Upon subscription, you will first receive a `Partial` back, containing a snapshot of the DerivaDEX state as of the latest checkpoint and all of the transactions from the latest checkpoint up until now.
DerivaDEX has on-chain checkpoints roughly every 10 minutes. Say you were to connect to the API and subscribe to this endpoint 7 minutes after Checkpoint 100 had completed. You would first receive
the entire state as of Checkpoint 100, and then all of the state-transitioning transactions that have elapsed in the last 7 minutes. Thus, you will be up-to-date without having to stream through all of
the transactions from the beginning of time (Checkpoint 0). This is an incredibly powerful response, as you can construct literally the **entire** state
of the DerivaDEX exchange from this.

The response is a 2-item array, with the first item referencing the state snapshot and the second referencing the transactions, as shown in the sample on the right. The types and fields that
make up this response are described in the table below:

type | field | description 
------ | ---- | -------
int_s | lowEpochId | Epoch ID of state entry
int_s | highEpochId | Epoch ID of state entry
bytes32_s | smtHash | Hash of the SMT's key and ABI-encoded value
bytes32_s | smtKey | Unique location of leaf in the SMT. The first byte of this key is the leaf item discriminant, and will inform you how to decode the remainder of the key and the `smtValue`.
bytes_s | smtValue | ABI-encoding of SMT leaf's value. You can appropriately decode this value using the first byte of the `smtKey`.
int_s | epochId | Epoch ID of transaction
int_s | txOrdinal | Monotonically increasing numerical value representing when a transaction took place in a given epoch. This will necessarily increase by 1 in order of being processed, and will only be reset to 0 when a new epoch has begun.
int_s | requestIndex | Sequenced identifier for the request initially made. This will be the same as the `requestIndex` you receive in a successful `command`.
bytes32_s | stateRootHash | SMT's state root hash prior to the transaction being applied
int | eventKind | Enum representation for event type (`0=PartialFill`, `1=CompleteFill`, `2=Post`, `3=Cancel`, `4=Liquidation`, `5=StrategyUpdate`, `6=TraderUpdate`, `7=Withdraw`, `8=WithdrawDDX`, `9=PriceCheckpoint`, `10=Funding`, `11=TradeMining`, `12=NoTransition`)
timestamp_s | createdAt | Timestamp transaction log entry was created
bytes_s | signature | Enclave's signature of transaction data
object | event | Event data (structure and contents are different depending on the `eventKind`)

> Update response (JSON)
```json
{
	"t": "TxLogUpdate",
	"e": "Update",
	"c": [{
		"epochId": "3286",
		"txOrdinal": "1",
		"requestIndex": "180849",
		"stateRootHash": "0xc211ee51304784f6d5cf6841768570199dc6cd75025b383a615b67ae1d5fb234",
		"eventKind": 9,
		"createdAt": "2021-06-30T20:33:00.478Z",
		"signature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000",
		"event": {
			"ETHPERP": {
				"ema": "0.513297744127047512",
				"indexPrice": "2263.064285714285714285",
				"indexPriceHash": "0x695c3030efe5609cee9cf66c7df0e451a80c388c3725abc6f5888d542bc3d480"
			}
		}
	}]
}
```

From this point onwards, you will receive `Update` messages, with individual transactions that you can apply to your local state you have constructed using the `Partial` to always stay up-to-date.
You may notice that the transaction log entries streamed in these update messages are identical to the individual
transaction entries shown in the second part of the `Partial` response above. 

type | field | description
-----|----- | ---------------------
int_s | epochId | Epoch ID of transaction
int_s | txOrdinal | Monotonically increasing numerical value representing when a transaction took place in a given epoch. This will necessarily increase by 1 in order of being processed, and will only be reset to 0 when a new epoch has begun.
int_s | requestIndex | Sequenced identifier for the request initially made. This will be the same as the `requestIndex` you receive in a successful `command`.
bytes32_s | stateRootHash | SMT's state root hash prior to the transaction being applied
int | eventKind | Enum representation for event type (`0=PartialFill`, `1=CompleteFill`, `2=Post`, `3=Cancel`, `4=Liquidation`, `5=StrategyUpdate`, `6=TraderUpdate`, `7=Withdraw`, `8=WithdrawDDX`, `9=PriceCheckpoint`, `10=Funding`, `11=TradeMining`, `12=NoTransition`)
timestamp_s | createdAt | Timestamp transaction log entry was created
bytes_s | signature | Enclave's signature of transaction data
object | event | Event data (structure and contents are different depending on the `eventKind`)

As mentioned above, there are 12 various types of transactions on DerivaDEX, each of which can modify the SMT's state
in different ways. The full set of transactions along with their corresponding numeric discrimants can be seen in the table below.
                   
Event | Discriminant
-----| ------------
PartialFill | 0
CompleteFill | 1
Post | 2
Cancel | 3
Liquidation | 4
StrategyUpdate | 5
TraderUpdate | 6
Withdraw | 7
WithdrawDDX | 8
PriceCheckpoint | 9
Funding | 10
TradeMining | 11
NoTransition | 12

Each of these transaction types are described at length below.

#### Partial fill

> Sample PartialFill (JSON)
```json
{
    "epochId": 1,
    "event": {
        "c": [
            [{
                "amount": "20",
                "makerOrderHash": "0xccdc891e0178fff88e9158ae341247afad50d08b000b00eed6",
                "makerOrderRemainingAmount": "0",
                "makerOutcome": {
                    "ddxFeeElection": false,
                    "fee": "0",
                    "newCollateral": "200000",
                    "newPositionAvgEntryPrice": "241",
                    "newPositionBalance": "60",
                    "positionSide": "Short",
                    "realizedPnl": "0",
                    "strategy": "main",
                    "trader": "0xa8dda8d7f5310e4a9e24f8eba77e091ac264f872"
                },
                "price": "247",
                "reason": "Trade",
                "symbol": "ETHPERP",
                "takerOrderHash": "0x3b1462eb4cd85476f059c4cd7d3aefdf9ec09f0ee85af3eeff",
                "takerOutcome": {
                    "ddxFeeElection": false,
                    "fee": "9.88",
                    "newCollateral": "199971.08",
                    "newPositionAvgEntryPrice": "241",
                    "newPositionBalance": "60",
                    "positionSide": "Long",
                    "realizedPnl": "0",
                    "strategy": "main",
                    "trader": "0x603699848c84529987e14ba32c8a66def67e9ece"
                },
                "takerSide": "Bid"
            }], {
                "amount": "80",
                "orderHash": "0x3b1462eb4cd85476f059c4cd7d3aefdf9ec09f0ee85af3eeff",
                "price": "250",
                "side": "Bid",
                "strategyId": "main",
                "symbol": "ETHPERP",
                "traderAddress": "0x603699848c84529987e14ba32c8a66def67e9ece"
            }
        ],
        "t": "PartialFill"
    },
    "eventKind": 0,
    "requestIndex": 25,
    "stateRootHash": "0x45f744d76fa11ff3b58decd3ec5573a23694bff212399801c4ed14cd680cbc73",
    "txOrdinal": 8
}
```

A `PartialFill` transaction is a scenario where the taker order has been partially filled across 1 or more
maker orders and thus has a remaining order that enters the order book. The event portion of the transaction response consists of a 2-item array. The
first item is a list of `Fill` events and the second item is the remaining `Post` event.

#### Complete fill

> Sample CompleteFill (JSON)
```json
{
    "epochId": 1,
    "event": {
        "c": [{
            "amount": "20",
            "makerOrderHash": "0x4b75a446ca9220d9dfd13bd697cba32657ad4caf1c51f59e8c",
            "makerOrderRemainingAmount": "0",
            "makerOutcome": {
                "ddxFeeElection": false,
                "fee": "0",
                "newCollateral": "200000",
                "newPositionAvgEntryPrice": "235",
                "newPositionBalance": "20",
                "positionSide": "Short",
                "realizedPnl": "0",
                "strategy": "main",
                "trader": "0xa8dda8d7f5310e4a9e24f8eba77e091ac264f872"
            },
            "price": "235",
            "reason": "Trade",
            "symbol": "ETHPERP",
            "takerOrderHash": "0x4347298b461db640c829e51890d10695249e193fcfb84c9786",
            "takerOutcome": {
                "ddxFeeElection": false,
                "fee": "9.4",
                "newCollateral": "199990.6",
                "newPositionAvgEntryPrice": "235",
                "newPositionBalance": "20",
                "positionSide": "Long",
                "realizedPnl": "0",
                "strategy": "main",
                "trader": "0x603699848c84529987e14ba32c8a66def67e9ece"
            },
            "takerSide": "Bid"
        }],
        "t": "CompleteFill"
    },
    "eventKind": 1,
    "requestIndex": 23,
    "stateRootHash": "0xe79bba967fb7ffd2a46ac76ca1d920109376eb57b4b691ffdc0375690d94e21c",
    "txOrdinal": 6
}
```

A `CompleteFill` is a scenario where the taker order has been completely filled across 1 or more maker orders.
The event portion of the transaction response consists of a list of `Fill` events.


#### Post

> Sample Post (JSON)
```json
{
    "epochId": 1,
    "event": {
        "c": {
            "amount": "20",
            "orderHash": "0x4b75a446ca9220d9dfd13bd697cba32657ad4caf1c51f59e8c",
            "price": "235",
            "side": "Ask",
            "strategyId": "main",
            "symbol": "ETHPERP",
            "traderAddress": "0xa8dda8d7f5310e4a9e24f8eba77e091ac264f872"
        },
        "t": "Post"
    },
    "eventKind": 2,
    "requestIndex": 20,
    "stateRootHash": "0x661b2467616c9268b59e2abe88de871f8857c3e255ae02cdcf421d84dad8def0",
    "txOrdinal": 3
}
```

A `Post` is an order that enters the order book. The event portion of the transaction has attributes defined as follows:

type | field | description
-----|----- | ---------------------
decimal_s | amount | Size of order posted to the order book
bytes32_s | orderHash | Hexstr representation of the EIP-712 hash of the order being placed
decimal_s | price | Price the order has been placed at
string | side | Side of order (`Bid` or `Ask`)
string | strategyId | Strategy ID this order belongs to (e.g. "main")
string | symbol | Symbol for the market this order has been placed (e.g. "ETHPERP")
address_s | traderAddress | Trader's Ethereum address


#### Cancel

> Sample Cancel (JSON)
```json
{
    "epochId": 1,
    "event": {
        "c": {
            "amount": "20",
            "orderHash": "0x4b75a446ca9220d9dfd13bd697cba32657ad4caf1c51f59e8c",
            "symbol": "ETHPERP"
        },
        "t": "Cancel"
    },
    "eventKind": 3,
    "requestIndex": 21,
    "stateRootHash": "0x661b2467616c9268b59e2abe88de871f8857c3e255ae02cdcf421d84dad8def0",
    "txOrdinal": 4
}
```

A `Cancel` is when an existing order is canceled and removed from the order book. The event portion of the transaction has attributes defined as follows:

type | field | description
-----|----- | ---------------------
decimal_s | amount | Size of order canceled from the order book
bytes32_s | orderHash | Hexstr representation of the EIP-712 hash of the order being canceled
string | symbol | Symbol for the market this order has been canceled (e.g. "ETHPERP")


#### Liquidation

TBD

#### Strategy update

> Sample StrategyUpdate (JSON)
```json
{
    "epochId": 1,
    "event": {
        "c": {
            "amount": "200000",
            "collateralAddress": "0xb69e673309512a9d726f87304c6984054f87a93b",
            "strategyId": "main",
            "trader": "0x603699848c84529987e14ba32c8a66def67e9ece",
            "txHash": "0xe6ae70d48a2800d2871237b1b90b15b8e3921f106ba31c1e4703df1c8ca49683",
            "updateType": "Deposit"
        },
        "t": "StrategyUpdate"
    },
    "eventKind": 5,
    "requestIndex": 13,
    "stateRootHash": "0x7d9c627046b71fabb3342602e7324df15b3dd46261584e9f6ee96ca12860aea2",
    "txOrdinal": 0
}
```

A `StrategyUpdate` is an update to a trader's strategy (such as depositing or withdrawing collateral). The event portion of the transaction has attributes defined as follows:

type | field | description
-----|----- | ---------------------
decimal_s | amount | Amount deposited or withdrawn
address_s | collateralAddress | Deposited / withdrawn collateral token's Ethereum address
string | strategyId | Strategy ID deposited to or withdrawn from (e.g. "main")
address_s | trader | Trader's Ethereum address
bytes32_s | txHash | Ethereum transaction hash for the on-chain deposit / withdrawal
string | updateType | Action corresponding to either "Deposit" or "Withdraw"

#### Trader update

> Sample TraderUpdate (JSON)
```json
{
    "epochId": 1,
    "event": {
        "c": {
            "amount": "1000",
            "trader": "0x603699848c84529987e14ba32c8a66def67e9ece",
            "txHash": "0xe6ae70d48a2800d2871237b1b90b15b8e3921f106ba31c1e4703df1c8ca49683",
            "updateType": "Deposit"
        },
        "t": "TraderUpdate"
    },
    "eventKind": 6,
    "requestIndex": 13,
    "stateRootHash": "0x7d9c627046b71fabb3342602e7324df15b3dd46261584e9f6ee96ca12860aea2",
    "txOrdinal": 0
}
```

A `TraderUpdate` is an update to a trader's DDX account (such as depositing or withdrawing DDX). The event portion of the transaction has attributes defined as follows:

type | field | description
-----|----- | ---------------------
decimal_s | amount | Amount of DDX deposited or withdrawn
address_s | trader | Trader's Ethereum address
bytes32_s | txHash | Ethereum transaction hash for the on-chain deposit / withdrawal
string | updateType | Action corresponding to either "Deposit" or "Withdraw"


#### Withdraw

TBD

#### Withdraw DDX

> Sample WithdrawDDX (JSON)
```json
{
	"epochId": 21,
	"event": {
		"c": {
			"amount": "100",
			"signerAddress": "0xa8dda8d7f5310e4a9e24f8eba77e091ac264f872",
			"traderAddress": "0x603699848c84529987e14ba32c8a66def67e9ece"
		},
		"t": "WithdrawDDX"
	},
	"eventKind": 8,
	"requestIndex": 431,
	"stateRootHash": "0xe59fae6c648ecba3fa433c8d0778a73dba3c7632e9f1a5e66b6f0c1060bdeb7d",
	"txOrdinal": 0
}
```

A `WithdrawDDX` is when a withdrawal of DDX is signaled). The event portion of the transaction has attributes defined as follows:

type | field | description
-----|----- | ---------------------
address_s | signerAddress | Signer's Ethereum address withdrawal is taking place from
address_s | traderAddress | Trader Ethereum address DDX is being withdrawn to
decimal_s | amount | Amount of DDX withdrawal has been signaled for

#### Price checkpoint

> Sample PriceCheckpoint (JSON)
```json
{
    "epochId": 1,
    "event": {
        "c": {
            "ETHPERP": {
                "ema": "0",
                "indexPrice": "250",
                "indexPriceHash": "0x91e61212e54565a46348af417a8aa5c820157d2c9d91931ce3ba717c9c2e57bd"
            }
        },
        "t": "PriceCheckpoint"
    },
    "eventKind": 9,
    "requestIndex": 19,
    "stateRootHash": "0x74304fc871abe5302df949c221bd8a2a0fd097cf15819c446da05420bfcbd2eb",
    "txOrdinal": 2
}
```

A `PriceCheckpoint` is when a market registers an update to the composite index price a perpetual is tracking along with the ema component. The event portion of the transaction essentially emits the latest `Price` leaf contents. As a reminder, the `Price` leaf's attributes are defined as follows:

type | field | description
-----|----- | ---------------------
decimal_s | ema | Captures a smoothed average of the spread between the underlying index price and the DerivaDEX order book for the market.
decimal_s | indexPrice | Composite index price (a weighted average across several price feed sources)
bytes32_s | indexPriceHash | Index price hash


#### Funding

> Sample Funding (JSON)
```json
{
	"epochId": 20,
	"event": {
		"c": {
			"fundingEpochId": 2,
			"fundingRates": {
				"ETHPERP": "0.0024"
			},
			"timeValue": 464
		},
		"t": "Funding"
	},
	"eventKind": 10,
	"requestIndex": 429,
	"stateRootHash": "0xe59fae6c648ecba3fa433c8d0778a73dba3c7632e9f1a5e66b6f0c1060bdeb7d",
	"txOrdinal": 1
}
```

A `Funding` is when a there is a funding rate distribution. The event portion of the transaction has attributes defined as follows:

type | field | description
-----|----- | ---------------------
int | fundingEpochId | The epoch id for the funding event
int | timeValue | Time value (??)
dict<string, decimal_s> | fundingRates | Mapping of symbol to funding rate

#### Trade mining

> Sample TradeMining (JSON)
```json
{
	"epochId": 20,
	"event": {
		"c": {
			"ddxDistributed": "3196.347031963470319632",
			"timeValue": 464,
			"totalVolume": {
				"makerVolume": "1000",
				"takerVolume": "40"
			},
			"tradeMiningEpochId": 2
		},
		"t": "TradeMining"
	},
	"eventKind": 11,
	"requestIndex": 428,
	"stateRootHash": "0x61a07997f1b4af0cffe2d356338a2dd35573b49078765a39e614c91c74c1812c",
	"txOrdinal": 0
}
```

A `TradeMining` is when a there is a funding rate distribution. The event portion of the transaction has attributes defined as follows:

type | field | description
-----|----- | ---------------------
int | tradeMiningEpochId | The epoch id for the trade mining event
int | timeValue | Time value (??)
decimal_s | ddxDistributed | The total DDX distributed during this trade mining distribution
dict<string, decimal_s> | totalVolume | Mapping containing the total maker and taker volumes for this trade mining interval


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
