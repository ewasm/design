# Ethereum Environment Interface (EEI) Specification (*Revision 3*)

The Ethereum Environment Interface exposes the core Ethereum API to the ewasm environment. The Ethereum [module](https://github.com/WebAssembly/design/blob/master/Modules.md) will be implemented in the Ethereum client's native language. All parameters and returns are restricted to 32 or 64 bit integers. Floats are disallowed.

# Data types

We define the following Ethereum data types:
- `bytes`: an array of bytes with unrestricted length
- `address`: a 160 bit number, represented as a 20 bytes long little endian unsigned integer in memory
- `u128`: a 128 bit number, represented as a 16 bytes long little endian unsigned integer in memory
- `u256`: a 256 bit number, represented as a 32 bytes long little endian unsigned integer in memory

We also define the following WebAssembly data types:
- `i32`: same as `i32` in WebAssembly
- `i32ptr`: same as `i32` in WebAssembly, but treated as a pointer to a WebAssembly memory offset
- `i64`: same as `i64` in WebAssembly

# API

## useGas

Subtracts an amount to the gas counter

**Parameters**

-   `amount` **i64** the amount to subtract to the gas counter

**Returns**

*nothing*

## getAddress

Gets address of currently executing account and loads it into memory at
the given offset.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load the address into (`address`)

**Returns**

*nothing*

## getBalance

Gets balance of the given account and loads it into memory at the given
offset.

**Parameters**

-   `addressOffset` **i32ptr** the memory offset to load the address from (`address`)
-   `resultOffset` **i32ptr** the memory offset to load the balance into (`u128`)

**Returns**

*nothing*

## getBlockHash

Gets the hash of one of the 256 most recent complete blocks.

**Parameters**

-   `number` **i64** which block to load
-   `resultOffset` **i32ptr** the memory offset to load the hash into (`u256`)

**Returns**

`result` **i32** Returns 0 on success and 1 on failure

*Note:* in case of failure, the output memory pointed by `resultOffset` is unchanged.

## call

Sends a message with arbitrary data to a given address path

**Parameters**

-   `gas` **i64** the gas limit
-   `addressOffset` **i32ptr** the memory offset to load the address from (`address`)
-   `valueOffset` **i32ptr** the memory offset to load the value from (`u128`)
-   `dataOffset` **i32ptr** the memory offset to load data from (`bytes`)
-   `dataLength` **i32** the length of data

**Returns**

`result` **i32** Returns 0 on success, 1 on failure and 2 on `revert`

## callDataCopy

Copies the input data in current environment to memory. This pertains to
the input data passed with the message call instruction or transaction.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load data into (`bytes`)
-   `dataOffset` **i32** the offset in the input data
-   `length` **i32** the length of data to copy

**Returns**

*nothing*

## getCallDataSize

Get size of input data in current environment. This pertains to the input
data passed with the message call instruction or transaction.

**Parameters**

*none*

**Returns**

`callDataSize` **i32**

## callCode

 Message-call into this account with an alternative account's code.

**Parameters**

-   `gas` **i64** the gas limit
-   `addressOffset` **i32ptr** the memory offset to load the address from (`address`)
-   `valueOffset` **i32ptr** the memory offset to load the value from (`u128`)
-   `dataOffset` **i32ptr** the memory offset to load data from (`bytes`)
-   `dataLength` **i32** the length of data

**Returns**

`result` **i32** Returns 0 on success, 1 on failure and 2 on `revert`

## callDelegate

Message-call into this account with an alternative account’s code, but
persisting the current values for sender and value.

**Parameters**

-   `gas` **i64** the gas limit
-   `addressOffset` **i32ptr** the memory offset to load the address from (`address`)
-   `dataOffset` **i32ptr** the memory offset to load data from (`bytes`)
-   `dataLength` **i32** the length of data

**Returns**

`result` **i32** Returns 0 on success, 1 on failure and 2 on `revert`

## callStatic

Sends a message with arbitrary data to a given address path, but disallow state
modifications. This includes `log`, `create`, `selfdestruct` and `call` with a non-zero
value.

**Parameters**

-   `gas` **i64** the gas limit
-   `addressOffset` **i32ptr** the memory offset to load the address from (`address`)
-   `dataOffset` **i32ptr** the memory offset to load data from (`bytes`)
-   `dataLength` **i32** the length of data

**Returns**

`result` **i32** Returns 0 on success, 1 on failure and 2 on `revert`

## storageStore

Store 256-bit a value in memory to persistent storage

**Parameters**

-   `pathOffset` **i32ptr** the memory offset to load the path from (`u256`)
-   `valueOffset` **i32ptr** the memory offset to load the value from (`u256`)

**Returns**

*nothing*

## storageLoad

Loads a 256-bit a value to memory from persistent storage

**Parameters**

-   `pathOffset` **i32ptr** the memory offset to load the path from (`u256`)
-   `resultOffset` **i32ptr** the memory offset to store the result at (`u256`)

**Returns**

*nothing*

## getCaller

Gets caller address and loads it into memory at the given offset. This is
the address of the account that is directly responsible for this execution.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load the address into (`address`)

**Returns**

*nothing*

## getCallValue

Gets the deposited value by the instruction/transaction responsible for
this execution and loads it into memory at the given location.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load the value into (`u128`)

**Returns**

*nothing*

## codeCopy

Copies the code running in current environment to memory.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load the result into (`bytes`)
-   `codeOffset` **i32** the offset within the code
-   `length` **i32** the length of code to copy

**Returns**

*nothing*

## getCodeSize

Gets the size of code running in current environment.

**Parameters**

*none*

**Returns**

`codeSize` **i32**

## getBlockCoinbase

Gets the block’s beneficiary address and loads into memory.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load the coinbase address into (`address`)

**Returns**

*nothing*

## create

Creates a new contract with a given value.

**Parameters**

-   `valueOffset` **i32ptr** the memory offset to load the value from (`u128`)
-   `dataOffset` **i32ptr** the memory offset to load the code for the new contract from (`bytes`)
-   `length` **i32** the data length
-   `resultOffset` **i32ptr** the memory offset to write the new contract address to (`address`)

*Note*: `create` will clear the return buffer in case of success or may fill it with data coming from `revert`.

**Returns**

`result` **i32** Returns 0 on success, 1 on failure and 2 on `revert`

## getBlockDifficulty

Get the block’s difficulty.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load the difficulty into (`u256`)

**Returns**

*nothing*

## externalCodeCopy

Copies the code of an account to memory.

**Parameters**

-   `addressOffset` **i32ptr** the memory offset to load the address from (`address`)
-   `resultOffset` **i32ptr** the memory offset to load the result into (`bytes`)
-   `codeOffset` **i32** the offset within the code
-   `length` **i32** the length of code to copy

**Returns**

*nothing*

## getExternalCodeSize

Get size of an account’s code.

**Parameters**

-   `addressOffset` **i32ptr** the memory offset to load the address from (`address`)

**Returns**

`extCodeSize` **i32**

## getGasLeft

Returns the current gasCounter

**Parameters**

*none*

**Returns**

`gasLeft` **i64**

## getBlockGasLimit

Get the block’s gas limit.

**Parameters**

*none*

**Returns**

`blockGasLimit` **i64**

## getTxGasPrice

Gets price of gas in current environment.

**Parameters**

-   `valueOffset` **i32ptr** the memory offset to write the value to (`u128`)

**Returns**

*nothing*

## log

Creates a new log in the current environment

**Parameters**

-   `dataOffset` **i32ptr** the memory offset to load data from (`bytes`)
-   `length` **i32** the data length
-   `numberOfTopics` **i32** the number of topics following (0 to 4)
-   `topic1` **i32ptr** the memory offset to load topic1 from (`u256`)
-   `topic2` **i32ptr** the memory offset to load topic2 from (`u256`)
-   `topic3` **i32ptr** the memory offset to load topic3 from (`u256`)
-   `topic4` **i32ptr** the memory offset to load topic4 from (`u256`)

**Returns**

*nothing*

## getBlockNumber

Get the block’s number.

**Parameters**

*none*

**Returns**

`blockNumber` **i64**

## getTxOrigin

Gets the execution's origination address and loads it into memory at the
given offset. This is the sender of original transaction; it is never an
account with non-empty associated code.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load the origin address from (`address`)

**Returns**

*nothing*

## finish

Set the returning output data for the execution.

*Note*: multiple invocations will overwrite the previous data.

**Parameters**

-   `dataOffset` **i32ptr** the memory offset of the output data (`bytes`)
-   `length` **i32** the length of the output data

**Returns**

*nothing*

## revert

Set the returning output data for the execution.

*Note*: multiple invocations will overwrite the previous data.

**Parameters**

-   `dataOffset` **i32ptr** the memory offset of the output data (`bytes`)
-   `length` **i32** the length of the output data

**Returns**

*nothing*

## getReturnDataSize

Get size of current return data buffer to memory. This contains the return data
from the last executed `call`, `callCode`, `callDelegate`, `callStatic` or `create`.

*Note*: `create` only fills the return data buffer in case of a failure.

**Parameters**

*none*

**Returns**

`dataSize` **i32**

## returnDataCopy

Copies the current return data buffer to memory. This contains the return data
from last executed `call`, `callCode`, `callDelegate`, `callStatic` or `create`.

*Note*: `create` only fills the return data buffer in case of a failure.

**Parameters**

-   `resultOffset` **i32ptr** the memory offset to load data into (`bytes`)
-   `dataOffset` **i32** the offset in the return data
-   `length` **i32** the length of data to copy

**Returns**

*nothing*

## selfDestruct

Mark account for later deletion and give the remaining balance to the specified
beneficiary address. This takes effect once the contract execution terminates.

*Note*: multiple invocations will overwrite the benficiary address.

*Note*: the contract **shall** halt execution after this call.

**Parameters**

-   `addressOffset` **i32ptr** the memory offset to load the address from (`address`)

**Returns**

*nothing*

## getBlockTimestamp

Get the block’s timestamp.

**Parameters**

*none*

**Returns**

`blockTimestamp` **i64**
