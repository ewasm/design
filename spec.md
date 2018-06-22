EEI specification
=================

Introduction
------------

This document aims to specify an ewasm VM in a way useful to contract writers and VM implementers.

### Terminology

**EEI**: The *Ethereum Environment Interface* refers to the layer between the Ethereum Client code and the execution engine.

**EEI method**: A method exposed by the Ethereum Client to the execution engine for interacting with the blockchain state.

**EEI namespace**: Named subset of the available EEI methods for specifying which blockchain state interactions are available to an execution engine.

**Ethereum Client**: Code which can interact with the blockchain (read/validate and broadcasting transactions).

**EVMC**: The *Ethereum Virtual Machine Connector* (correct name?) refers to one of (**TODO: Which one?**):
          -   The C++ header file specifying the EEI.
          -   The execution engine side of the EEI.

**execution engine**: The underlying "hardware" of the VM, implementing the basic computational functions.

**VM**: The VM is the combination of an Ethereum Client and the execution engine.

### Notation

We are using [K Framework] notation to specify the EEI, which makes this specification executable.

```k
requires "domains.k"

module EEI
    imports DOMAINS
```

Execution State
---------------

Below both a K rule and a prose description of each state transition is given.
The state is specified using a K *configuration*.
Each XML-like *cell* contains a field which is relevant to Ethereum client execution (eg. below the first cell is the `<k>` cell).
The default/initial values of the cells are provided along with the declaration of the configuration.

In the texual rules below, we'll refer to cells by accesing subcells with the `.` operator.
For example, we would access contents of the `statusCode` cell with `eei.statusCode`.
For cells that contain elements of K builtin sorts `Map`, `List`, and `Set`, we'll use standard K operators for referring to their contents.
For example, we can access the third element of the `returnData` cell's list using `eei.callState.returnData[2]`.

For some cells, we have comments following the cell declarations with the name the [Yellow Paper] gives to that element of the state.

```k
    configuration
      <k> $PGM:EthereumSimulation </k>
      <eei>
        <statusCode> .StatusCode </statusCode>
```

The `<callState>` sub-configuration can be saved/restored when needed between calls.

```k
        <callState>
          <callDepth>  0     </callDepth>
          <returnData> .List </returnData>

          <acct>      0        </acct>      // I_a
          <program>   .Program </program>   // I_b
          <caller>    0        </caller>    // I_s
          <callData>  .List    </callData>  // I_d
          <callValue> 0        </callValue> // I_v

          <gas>        0 </gas>        // \mu_g
          <memoryUsed> 0 </memoryUsed> // \mu_i
        </callState>
```

The execution `<substate>` keeps track of the self-destruct set, the log, and accumulated gas refund.

```k
        <substate>
          <selfDestruct> .Set  </selfDestruct> // A_s
          <log>          .List </log>          // A_l
          <refund>       0     </refund>       // A_r
        </substate>
```

The `<accounts>` sub-configuration stores information about each account on the blockchain.
The `multiplicity="*"` allows us to have multiple accounts simultaneously, and `type="Map"` allows us to access accounts by using the `<id>` as a key.
For example, `eei.accounts[0x0001].nonce` would access the nonce of account `0x0001`.

```k
        <accounts>
          <account multiplicity="*" type="Map">
            <id>      0        </id>
            <balance> 0        </balance>
            <code>    .Program </code>
            <storage> .Map     </storage>
            <nonce>   0        </nonce>
          </account>
        </accounts>
```

Transaction state `<tx>`:

```k
        <tx>
          <gasPrice> 0 </gasPrice> // I_p
          <origin>   0 </origin>   // I_o
        </tx>
```

And finally, block stack `<block>`:

```k
        <block>
          <hashes>           .List      </hashes>
          <coinbase>         0          </coinbase>         // H_c
       // <logsBloom>        .WordStack </logsBloom>        // H_b
          <difficulty>       0          </difficulty>       // H_d
          <number>           0          </number>           // H_i
          <gasLimit>         0          </gasLimit>         // H_l
          <timestamp>        0          </timestamp>        // H_s
        </block>
      </eei>
```

Ethereum Simulations
--------------------

An `EthereumSimulation` is a list of commands to be executed via the EEI.
Each `EthereumCommmand` can invoke the execution engine on an input or interact with the client.

```k
    syntax EthereumSimulation ::= List{EthereumCommand, ""}
```

Sort `EEIMethod` is used to make calls into the client via the EEI.
Sort `Program` is used to invoke the execution engine via the EEI.

```k
    syntax EthereumCommand ::= EEIMethod
                             | Program
```

Each execution engine has it's own program representation, so we make a wrapper for them.
Here, we make a sort `Program` which has a single constant `.Program` to use as the default.

```k
    syntax Program ::= ".Program"
```

Status Codes
------------

The [EVMC] status codes are used by the execution engine to indicate to the client how execution ended.
Currently, they are broken into three subsorts, for exceptional, ending, or error statuses.
The extra status code `.StatusCode` is used as a default status code when none has been set.

```k
    syntax StatusCode ::= ExceptionalStatusCode
                        | EndStatusCode
                        | ErrorStatusCode
                        | ".StatusCode"
```

### Exceptional Codes

The following codes all indicate that the VM ended execution with an exception, but give details about how.

-   `EVMC_FAILURE` is a catch-all for generic execution failure.
-   `EVMC_INVALID_INSTRUCTION` indicates reaching the designated `INVALID` opcode.
-   `EVMC_UNDEFINED_INSTRUCTION` indicates that an undefined opcode has been reached.
-   `EVMC_OUT_OF_GAS` indicates that execution exhausted the gas supply.
-   `EVMC_BAD_JUMP_DESTINATION` indicates a `JUMP*` to a non-`JUMPDEST` location.
-   `EVMC_STACK_OVERFLOW` indicates pushing more than 1024 elements onto the wordstack.
-   `EVMC_STACK_UNDERFLOW` indicates popping elements off an empty wordstack.
-   `EVMC_CALL_DEPTH_EXCEEDED` indicates that we have executed too deeply a nested sequence of `CALL*` or `CREATE` opcodes.
-   `EVMC_INVALID_MEMORY_ACCESS` indicates that a bad memory access occured.
    This can happen when accessing local memory with `CODECOPY*` or `CALLDATACOPY`, or when accessing return data with `RETURNDATACOPY`.
-   `EVMC_STATIC_MODE_VIOLATION` indicates that a `STATICCALL` tried to change state.
-   `EVMC_PRECOMPILE_FAILURE` indicates an error in the precompiled contracts (eg. invalid points handed to elliptic curve functions).

```k
    syntax ExceptionalStatusCode ::= "EVMC_FAILURE"
                                   | "EVMC_INVALID_INSTRUCTION"
                                   | "EVMC_UNDEFINED_INSTRUCTION"
                                   | "EVMC_OUT_OF_GAS"
                                   | "EVMC_BAD_JUMP_DESTINATION"
                                   | "EVMC_STACK_OVERFLOW"
                                   | "EVMC_STACK_UNDERFLOW"
                                   | "EVMC_CALL_DEPTH_EXCEEDED"
                                   | "EVMC_INVALID_MEMORY_ACCESS"
                                   | "EVMC_STATIC_MODE_VIOLATION"
                                   | "EVMC_PRECOMPILE_FAILURE"
```

### Ending Codes

These additional status codes indicate that execution has ended in some non-exceptional way.

-   `EVMC_SUCCESS` indicates successful end of execution.
-   `EVMC_REVERT` indicates that the contract called `REVERT`.

```k
    syntax EndStatusCode ::= ExceptionalStatusCode
                           | "EVMC_SUCCESS"
                           | "EVMC_REVERT"
```

### Error Codes

The following codes indicate other non-execution errors with the execution engine.

-   `EVMC_REJECTED` indicates malformed or wrong-version EVM bytecode.
-   `EVMC_INTERNAL_ERROR` indicates some other error that is unrecoverable but not due to the bytecode.

```k
    syntax ErrorStatusCode ::= "EVMC_REJECTED"
                             | "EVMC_INTERNAL_ERROR"
```

EEI Methods
-----------

The EEI exports several methods which can be invoked by the VM to interact with the client.
Here the syntax and semantics of these methods is defined.

Each section header gives the name of the given EEI method, along with the arguments needed.
For example, `EEI.useGas : Int` declares that `EEI.useGas` in an EEI method which takes a single integer as input.
The semantics are provided in three forms:

1.  a short prose description of purpose,

2.  a list of steps that must be taken, and

3.  a set K rules specifying the state update that can happen.

### Block and Transaction Information Getters

Many of the methods exported by the EEI simply query for some state of the current block/transaction.
These methods are prefixed with `get`, and have largely similar and simple rules.

#### `EEI.getBlockCoinbase`

Get the coinbase of the current block.

1.  Load and return `eei.block.coinbase`.

```k
    syntax EEIMethod ::= "EEI.getBlockCoinbase"
 // -------------------------------------------
    rule <k> EEI.getBlockCoinbase => CBASE ... </k>
         <coinbase> CBASE </coinbase>
```

#### `EEI.getBlockDifficulty`

Get the difficulty of the current block.

1.  Load and return `eei.block.difficulty`.

```k
    syntax EEIMethod ::= "EEI.getBlockDifficulty"
 // ---------------------------------------------
    rule <k> EEI.getBlockDifficulty => DIFF ... </k>
         <difficulty> DIFF </difficulty>
```

#### `EEI.getBlockGasLimit`

Get the gas limit for the current block.

1.  Load and return `eei.block.gasLimit`.

```k
    syntax EEIMethod ::= "EEI.getBlockGasLimit"
 // -------------------------------------------
    rule <k> EEI.getBlockGasLimit => GLIMIT ... </k>
         <gasLimit> GLIMIT </gasLimit>
```

#### `EEI.getBlockHash : Int`

Return the blockhash of one of the `N`th most recent complete blocks (as long as `N <Int 256`).
If there are not `N` blocks yet, return `0`.

**TODO:** Double-check this logic, esp for off-by-one errors.

1.  Load `BLOCKNUM` from `eei.block.number`.

2.  If `N <Int 256` and `N <Int BLOCKNUM`:

    i.  then: Load and return `eei.block.hashes[N]`.

    ii. else: Return `0`.

```k
    syntax EEIMethod ::= "EEI.getBlockHash" Int
 // -------------------------------------------
    rule <k> EEI.getBlockHash N => BLKHASHES[N] ... </k>
         <hashes> BLKHASHES </hashes>
      requires N <Int 256

    rule <k> EEI.getBlockHash N => 0 ... </k>
      requires N >=Int 256
```

#### `EEI.getBlockNumber`

Get the current block number.

1.  Load and return `eei.block.number`.

```k
    syntax EEIMethod ::= "EEI.getBlockNumber"
 // -----------------------------------------
    rule <k> EEI.getBlockNumber => BLKNUMBER ... </k>
         <number> BLKNUMBER </number>
```

#### `EEI.getBlockTimestamp`

Get the timestamp of the last block.

1.  Load and return `eei.block.timestamp`.

```k
    syntax EEIMethod ::= "EEI.getBlockTimestamp"
 // --------------------------------------------
    rule <k> EEI.getBlockTimestamp => TSTAMP ... </k>
         <timestamp> TSTAMP </timestamp>
```

#### `EEI.getTxGasPrice`

Get the gas price of the current transation.

1.  Load and return `eei.tx.gasPrice`.

```k
    syntax EEIMethod ::= "EEI.getTxGasPrice"
 // ----------------------------------------
    rule <k> EEI.getTxGasPrice => GPRICE ... </k>
         <gasPrice> GPRICE </gasPrice>
```

#### `EEI.getTxOrigin`

Get the address which sent this transaction.

1.  Load and return `eei.tx.origin`.

```k
    syntax EEIMethod ::= "EEI.getTxOrigin"
 // --------------------------------------
    rule <k> EEI.getTxOrigin => ORG ... </k>
         <origin> ORG </origin>
```

### Call State Methods

These methods return information about the current call operation, which may change throughout a given transaction/block.

#### `EEI.getAddress`

Return the address of the currently executing account.

1.  Load and return the value `eei.callState.acct`.

```k
    syntax EEIMethod ::= "EEI.getAddress"
 // -------------------------------------
    rule <k> EEI.getAddress => ADDR ... </k>
         <acct> ADDR </acct>
```

#### `EEI.getCaller`

Get the account id of the caller into the current execution.

1.  Load and return `eei.callState.caller`.

```k
    syntax EEIMethod ::= "EEI.getCaller"
 // ------------------------------------
    rule <k> EEI.getCaller => CACCT ... </k>
         <caller> CACCT </caller>
```

#### `EEI.getCallData`

-   `callDataSize` can be implemented client-side in terms of this opcode.

Returns the calldata associated with this call.

1.  Load and return `eei.callState.callData`.

```k
    syntax EEIMethod ::= "EEI.getCallData"
 // --------------------------------------
    rule <k> EEI.getCallData => CDATA ... </k>
         <callData> CDATA </callData>
```

#### `EEI.getCallValue`

Get the value transferred for the current call.

1.  Load and return `eei.callState.callValue`.

```k
    syntax EEIMethod ::= "EEI.getCallValue"
 // ---------------------------------------
    rule <k> EEI.getCallValue => CVALUE ... </k>
         <callValue> CVALUE </callValue>
```

#### `EEI.getGasLeft`

Get the gas left available for this execution.

1.  Load and return `eei.callState.gas`.

```k
    syntax EEIMethod ::= "EEI.getGasLeft"
 // -------------------------------------
    rule <k> EEI.getGasLeft => GAVAIL ... </k>
         <gas> GAVAIL </gas>
```

#### `EEI.getReturnData`

-   `getReturnDataSize` can be implemented in terms of this method.

Get the return data of the last call.

1.  Load and return `eei.callState.returnData`.

```k
    syntax EEIMethod ::= "EEI.getReturnData"
 // ----------------------------------------
    rule <k> EEI.getReturnData => RETDATA ... </k>
         <returnData> RETDATA </returnData>
```

### Gas Consumption

#### `EEI.useGas : Int`

Deduct the specified amount of gas (`GDEDUCT`) from the available gas.

1.  Load the value `GAVAIL` from `eei.gas`.

2.  If `GDEDUCT <Int GAVAIL`:

    i.  then: Set `eei.callState.gas` to `GAVAIL -Int GDEDUCT`.

    ii. else: Set `eei.statusCode` to `EVMC_OUT_OF_GAS` and `eei.callState.gas` to `0`.

```k
    syntax EEIMethod ::= "EEI.useGas" Int
 // -------------------------------------
    rule <k> EEI.useGas GDEDUCT => . ... </k>
         <gas> GAVAIL => GAVAIL -Int GDEDUCT </gas>
      requires GAVAIL >Int GDEDUCT

    rule <k> EEI.useGas GDEDUCT => . ... </k>
         <statusCode> _ => EVMC_OUT_OF_GAS </statusCode>
         <gas> GAVAIL </gas>
      requires GAVAIL <=Int GDEDUCT
```

### World State Methods

These operators query the world state (eg. account balances).
We prefix those that query about the currently executing account with `getAccount` (similarly `setAccount` for setting state).
Those that can query about other accounts are prefixed with `getExternalAccount`.

#### `EEI.getAccountBalance`

Return the balance of the current account (`ACCT`).

1.  Load the value `ACCT` from `eei.callState.acct`.

2.  Load and return the value `eei.accounts[ACCT].balance`.

```k
    syntax EEIMethod ::= "EEI.getAccountBalance"
 // --------------------------------------------
    rule <k> EEI.getAccountBalance => BAL ... </k>
         <acct> ACCT </acct>
         <account>
           <id> ACCT </id>
           <balance> BAL </balance>
           ...
         </account>
```

#### `EEI.getAccountCode`

Return the code of the current account (`ACCT`).

1.  Load the value `ACCT` from `eei.callState.acct`.

2.  Load and return `eei.accounts[ACCT].code`.

```k
    syntax EEIMethod ::= "EEI.getAccountCode"
 // -----------------------------------------
    rule <k> EEI.getAccountCode => ACCTCODE ... </k>
         <acct> ACCT </acct>
         <accounts>
           <id> ACCT </id>
           <code> ACCTCODE </code>
           ...
         </accounts>
```

#### `EEI.getExternalAccountCode : Int`

Return the code of the given account `ACCT`.

1.  Load and return `eei.accounts[ACCT].code`.

```k
    syntax EEIMethod ::= "EEI.getExternalAccountCode" Int
 // -----------------------------------------------------
    rule <k> EEI.getExternalAccountCode ACCT => ACCTCODE ... </k>
         <accounts>
           <id> ACCT </id>
           <code> ACCTCODE </code>
           ...
         </accounts>
```

#### `EEI.getAccountStorage : Int`

Return the value at the given `INDEX` in the current executing accout's storage.

1.  Load `ACCT` from `eei.callState.acct`.

2.  If `eei.accounts[ACCT].storage[INDEX]` exists:

    i.  then: return `eei.accounts[ACCT].storage[INDEX]`.

    ii. else: return `0`.

```k
    syntax EEIMethod ::= "EEI.getAccountStorage" Int
 // ------------------------------------------
    rule <k> EEI.getAccountStorage INDEX => VALUE ... </k>
         <acct> ACCT </acct>
         <account>
           <id> ACCT </id>
           <storage> ... INDEX |-> VALUE ... </storage>
           ...
         </account>

    rule <k> EEI.getAccountStorage INDEX => 0 ... </k>
         <acct> ACCT </acct>
         <account>
           <id> ACCT </id>
           <storage> STORAGE </storage>
           ...
         </account>
      requires notBool INDEX in_keys(STORAGE)
```

#### `EEI.setAccountStorage : Int Int`

At the given `INDEX` in the executing accounts storage, stores the given `VALUE`.

1.  Load `ACCT` from `eei.callState.acct`.

2.  Set `eei.accounts[ACCT].storage[INDEX]` to `VALUE`.

```k
    syntax EEIMethod ::= "EEI.setAccountStorage" Int Int
 // ----------------------------------------------------
    rule <k> EEI.setAccountStorage INDEX VALUE => . ... </k>
         <acct> ACCT </acct>
         <account>
           <id> ACCT </id>
           <storage> STORAGE => STORAGE [ INDEX <- VALUE ] </storage>
           ...
         </account>
```

### Logging

#### `EEI.log : List List`

Logging places a user-specified lists of integers (`BS1` and `BS2`) on the blockchain Log for external inspection.

First we define a log-item, which is an account id and two integer lists (in EVM, these come from the wordstack and the local memory).

```k
    syntax LogItem ::= "{" Int "|" List "|" List "}"
 // ------------------------------------------------
```

1.  Load the current `ACCT` from `eei.callState.acct`.

2.  Append `{ ACCT | BS1 | BS2 }` to the `eei.substate.log`.

```k
    syntax EEIMethod ::= "EEI.log" List List
 // ----------------------------------------
    rule <k> EEI.log BS1 BS2 => . ... </k>
         <acct> ACCT </acct>
         <log> ... (.List => ListItem({ ACCT | BS1 | BS2 })) </log>
```

### EEI Call (and Call-like) Methods

The remaining methods have more complex interactions with the EEI, often triggering further computation.

#### `EEI.selfDestruct : Int`

Selfdestructing removes the current executing account and transfers the funds of it to the specified target account `ACCTTO`.
If the target account is the same as the executing account, the balance of the current account is zeroed immediately.
In any case, the status is set to `EVMC_SUCCESS`.

1.  Load `ACCT` from `eei.callState.acct`.

2.  Add `ACCT` to the set `eei.substate.selfDestruct`.

3.  Set `eei.callState.returnData` to `.List` (empty).

4.  Load `BAL` from `eei.accounts[ACCT].balance`.

5.  Set `eei.accounts[ACCT].balance` to `0`.

6.  If `ACCT == ACCTTO`:

    i.  then: skip.

    ii. else: add `BAL` to `eei.accounts[ACCTTO].balance`.

```k
    syntax EEIMethod ::= "EEI.selfDestruct" Int
 // -------------------------------------------
    rule <k> EEI.selfDestruct ACCTTO => . ... </k>
         <statusCode> _ => EVMC_SUCCESS </statusCode>
         <acct> ACCT </acct>
         <returnData> _ => .List </returnData>
         <selfDestruct> ... (.Set => SetItem(ACCT)) ... </selfDestruct>
         <accounts>
           <account>
             <id> ACCT </id>
             <balance> BAL => 0 </balance>
             ...
           </account>
           <account>
             <id> ACCTTO </id>
             <balance> BALTO => BALTO +Int BAL </balance>
             ...
           </account>
           ...
         </accounts>
      requires ACCTTO =/=K ACCT

    rule <k> EEI.selfDestruct ACCT => . ... </k>
         <statusCode> _ => EVMC_SUCCESS </statusCode>
         <acct> ACCT </acct>
         <returnData> _ => .List </returnData>
         <selfDestruct> ... (.Set => SetItem(ACCT)) ... </selfDestruct>
         <accounts>
           <account>
             <id> ACCT </id>
             <balance> BAL => 0 </balance>
             ...
           </account>
           ...
         </accounts>
```

#### `EEI.return : List`

Set the return data to the given list of `RDATA` as well setting the status code to `EVMC_SUCCESS`.

1.  Set `eei.callState.returnData` to `RDATA`.

2.  Set `eei.statusCode` to `EVMC_SUCCESS`.

```k
    syntax EEIMethod ::= "EEI.return" List
 // --------------------------------------
    rule <k> EEI.return RDATA => . ... </k>
         <statusCode> _ => EVMC_SUCCESS </statusCode>
         <returnData> _ => RDATA </returnData>
```

#### `EEI.revert : List`

Set the return data to the given list of `RDATA` as well setting the status code to `EVMC_REVERT`.

1.  Set `eei.callState.returnData` to `RDATA`.

2.  Set `eei.statusCode` to `EVMC_REVERT`.

```k
    syntax EEIMethod ::= "EEI.revert" List
 // --------------------------------------
    rule <k> EEI.revert RDATA => . ... </k>
         <statusCode> _ => EVMC_REVERT </statusCode>
         <returnData> _ => RDATA </returnData>
```

#### `EEI.call : Int ByteString ByteString`

-   `EEI.call` **TODO**
-   `EEI.callCode` **TODO**
-   `EEI.callDelegate` **TODO**
-   `EEI.callStatic` **TODO**

**TODO:** Implement one abstract-level `EEI.call`, akin to `#call` in KEVM, which other `CALL*` opcodes can be expressed in terms of.

#### `EEI.create` **TODO**

```k
endmodule
```

Resources
=========

[K Framework]: <https://github.com/kframework/k>
[EVMC]: <https://github.com/ethereum/evmc>
[Yellow Paper]: <https://github.com/ethereum/yellowpaper>
