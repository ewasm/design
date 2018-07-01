EEI specification
=================

Introduction
------------

This document aims to specify an ewasm VM in a way useful to contract writers and VM implementers.
To this end, multiple things are specified:

-   The extra state that a VM needs to have around to successfully respond to calls into the EEI.
-   The EEI (Ethereum Environment Interface), currently specified loosely [here](eth_interface.md).

### Terminology

**EEI**: The *Ethereum Environment Interface* refers to the layer between the Ethereum Client code and the execution engine.
         **TODO**: Is *EEI* the right term to use?

**Ethereum Client**: Code which can interact with the blockchain (read/validate and sending transactions).

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
For example, we can access the third element of the `returnData` cell's list using `eei.returnData[2]`.

For some cells, we have comments following the cell declarations with the name the [Yellow Paper] gives to that element of the state.

```k
    configuration
      <k> $PGM:EthereumSimulation </k>
      <eei>
        <statusCode> .StatusCode </statusCode>
        <returnData> .List       </returnData>
```

The `<callState>` sub-configuration can be saved/restored when needed between calls.
When stored, it's stored in the `<callStack>` cell as a list.

```k
        <callState>
          <callDepth> 0        </callDepth>
          <acct>      0        </acct>      // I_a
          <program>   .Program </program>   // I_b
          <caller>    0        </caller>    // I_s
          <callData>  .List    </callData>  // I_d
          <callValue> 0        </callValue> // I_v
          <gas>       0        </gas>       // \mu_g
        </callState>

        <callStack> .List </callStack>
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

Similar to the `<callState>`, the an `<account>` state can be saved or restored on the `<accountStack>` cell.

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

        <accountsStack> .List </accountsStack>
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

The following are status codes used to report network state failures to the EVM from the client.
These are not present in the [EVM-C API].

-   `EVMC_ACCOUNT_ALREADY_EXISTS` indicates that a newly created account already exists.
-   `EVMC_BALANCE_UNDERFLOW` indicates an attempt to create an account which already exists.

```k
    syntax ExceptionalStatusCode ::= "EVMC_ACCOUNT_ALREADY_EXISTS"
                                   | "EVMC_BALANCE_UNDERFLOW"
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

### EEI Internal Helpers

These methods are used by the other EEI methods as helpers and intermediates to perform larger more complex tasks.
They are for the most part not intended to be exposed to the execution engine or Ethereum client for direct usage.

**TODO**: `{push,pop,drop}Accounts` should be able to take a specific list of accounts to push/pop/drop, making them more efficient.

#### `EEI.pushCallState`

Saves a copy of the current call state in the `<callStack>`.

1.  Load the current `CALLSTATE` from `eei.callState`.

2.  Prepend `CALLSTATE` to the `eei.callStack`.

```k
    syntax EEIMethod ::= "EEI.pushCallState"
 // ----------------------------------------
    rule <k> EEI.pushCallState => . ... </k>
         <callState> CALLSTATE </callState>
         <callStack> (.List => ListItem(CALLSTATE)) ... </callStack>
```

#### `EEI.popCallState`

Restores the most recently saved `<callState>`.

1.  Load the new `CALLSTATE` from `eei.callStack[0]`.

2.  Remove the first element of `eei.callStack`.

3.  Set `eei.callState` to `CALLSTATE`.

```k
    syntax EEIMethod ::= "EEI.popCallState"
 // ----------------------------------------
    rule <k> EEI.popCallState => . ... </k>
         <callState> _ => CALLSTATE </callState>
         <callStack> (ListItem(CALLSTATE) => .List) ... </callStack>
```

#### `EEI.dropCallState`

Forgets the most recently saved `<callState>` as reverting back to it will no longer happen.

1.  Remove the first element of `eei.callStack`.

```k
    syntax EEIMethod ::= "EEI.dropCallState"
 // ----------------------------------------
    rule <k> EEI.dropCallState => . ... </k>
         <callStack> (ListItem(_) => .List) ... </callStack>
```

#### `EEI.pushAccounts`

Saves a copy of the `<accounts>` state in the `<accountStack>` cell.

1.  Load the current `ACCTDATA` from `eei.accounts`.

2.  Prepend `ACCTDATA` to the `eei.accountStack`.

```k
    syntax EEIMethod ::= "EEI.pushAccount"
 // --------------------------------------
    rule <k> EEI.pushAccount => . ... </k>
         <accounts> ACCTDATA </accounts>
         <accountsStack> (.List => ListItem(ACCTDATA)) ... </accountsStack>
```

#### `EEI.popAccounts`

Restores the most recently saved `<accounts>` state.

1.  Load the new `ACCTDATA` from `eei.accountsStack[0]`.

2.  Remove the first element of `eei.accountsStack`.

3.  Set `eei.accounts` to `ACCTDATA`.

```k
    syntax EEIMethod ::= "EEI.popAccounts"
 // --------------------------------------
    rule <k> EEI.popAccounts => . ... </k>
         <accounts> _ => ACCTDATA </accounts>
         <accountsStack> (ListItem(ACCTDATA) => .List) ... </accountsStack>
```

#### `EEI.dropAccounts`

Forgets the most recently saved `<accounts>` state as reverting back to it will no longer happen.

1.  Remove the first element of `eei.accountsStack`.

```k
    syntax EEIMethod ::= "EEI.dropAccounts"
 // ---------------------------------------
    rule <k> EEI.dropAccounts => . ... </k>
         <accountsStack> (ListItem(_) => .List) ... </accountsStack>
```

#### `EEI.ifStatus : EthereumSimulation EthereumSimulation`

If the statuscode is not exception, execute `ETHSIMULATIONGOOD`, otherwise execute `ETHSIMULATIONBAD`.

1.  Load the `STATUSCODE` from `eei.statusCode`.

2.  If `STATUSCODE` is not an `ExceptionalStatusCode`, then:

    i.  Call `ETHSIMULATIONGOOD`.

    else:

    i.  Call `ETHSIMULATIONBAD`.

```k
    syntax EEIMethod ::= "EEI.ifStatus" EthereumSimulation EthereumSimulation
 // -------------------------------------------------------------------------
    rule <k> EEI.ifStatus ETHSIMULATIONGOOD ETHSIMULATIONBAD => ETHSIMULATIONGOOD ... </k>
         <statusCode> STATUSCODE </statusCode>
      requires notBool isExceptionalStatusCode(STATUSCODE)

    rule <k> EEI.ifStatus ETHSIMULATIONGOOD ETHSIMULATIONBAD => ETHSIMULATIONBAD ... </k>
         <statusCode> STATUSCODE </statusCode>
      requires isExceptionalStatusCode(STATUSCODE)
```

#### `EEI.onGoodStatus : EthereumSimulation`

Executes the given `ETHSIMULATION` if the current status code is not exceptional.

1.  Call `EEI.ifStatus ETHSIMULATION .EthereumSimulation`

```k
    syntax EEIMethod ::= "EEI.onGoodStatus" EthereumSimulation
 // ----------------------------------------------------------
    rule <k> EEI.onGoodStatus ETHSIMULATION => EEI.ifStatus ETHSIMULATION .EthereumSimulation ... </k>
```

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

2.  If `N <Int 256` and `N <Int BLOCKNUM`, then:

    i.  Load and return `eei.block.hashes[N]`.

    else:

    i.  Return `0`.

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

1.  Load and return `eei.returnData`.

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

2.  If `GDEDUCT <Int GAVAIL`, then:

    i.  Set `eei.callState.gas` to `GAVAIL -Int GDEDUCT`.

    else:

    i.  Set `eei.statusCode` to `EVMC_OUT_OF_GAS` and `eei.callState.gas` to `0`.

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

2.  If `eei.accounts[ACCT].storage[INDEX]` exists, then:

    i.  Return `eei.accounts[ACCT].storage[INDEX]`.

    else:

    i.  Return `0`.

```k
    syntax EEIMethod ::= "EEI.getAccountStorage" Int
 // ------------------------------------------------
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

3.  Set `eei.returnData` to `.List` (empty).

4.  Load `BALFROM` from `eei.accounts[ACCT].balance`.

5.  Set `eei.accounts[ACCT].balance` to `0`.

6.  If `ACCT =/=Int ACCTTO`, then:

    i.  Load `BALTO` from `eei.acounts[ACCTTO].balance`.

    ii. Set `eei.accounts[ACCTTO].balance` to `BALTO +Int BALFROM`.

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
             <balance> BALFROM => 0 </balance>
             ...
           </account>
           <account>
             <id> ACCTTO </id>
             <balance> BALTO => BALTO +Int BALFROM </balance>
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
             <balance> BALFROM => 0 </balance>
             ...
           </account>
           ...
         </accounts>
```

#### `EEI.return : List`

Set the return data to the given list of `RDATA` as well setting the status code to `EVMC_SUCCESS`.

1.  Set `eei.returnData` to `RDATA`.

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

1.  Set `eei.returnData` to `RDATA`.

2.  Set `eei.statusCode` to `EVMC_REVERT`.

```k
    syntax EEIMethod ::= "EEI.revert" List
 // --------------------------------------
    rule <k> EEI.revert RDATA => . ... </k>
         <statusCode> _ => EVMC_REVERT </statusCode>
         <returnData> _ => RDATA </returnData>
```

#### `EEI.transfer : Int Int`

Transfer `VALUE` funds into account `ACCTTO`.

1.  Load `ACCTFROM` from `eei.callState.acct`.

2.  Load `BALFROM` from `eei.accounts[ACCTFROM].balance`.

3.  If `VALUE >Int BALFROM`, then:

    i.  Set `eei.statusCode` to `EVMC_BALANCE_UNDERFLOW`.

    else:

    i.   Set `eei.accounts[ACCTFROM].balance` to `BAL -Int VALUE`.

    ii.  Load `BALTO` from `eei.accounts[ACCTTO].balance`.

    iii. Set `eei.accounts[ACCTTO].balance` to `BALTO +Int VALUE`.

```k
    syntax EEIMethod ::= "EEI.transfer" Int Int
 // -------------------------------------------
    rule <k> EEI.transfer ACCTTO VALUE => ... </k>
         <statusCode> _ => EVMC_BALANCE_UNDERFLOW </statusCode>
         <acct> ACCTFROM </acct>
         <account>
           <id> ACCTFROM </id>
           <balance> BALFROM </balance>
           ...
         </account>
      requires VALUE >Int BALFROM

    rule <k> EEI.transfer ACCTTO VALUE => ... </k>
         <acct> ACCTFROM </acct>
         <account>
           <id> ACCTFROM </id>
           <balance> BALFROM => BALFROM -Int VALUE </balance>
           ...
         </account>
         <account>
           <id> ACCTTO  </id>
           <balance> BALTO => BALTO +Int VALUE </balance>
           ...
         </account>
      requires VALUE <=Int BALFROM
```

#### `EEI.callInit : Int Int Int Int Program List`

Helper for setting up the execution engine to run a specific program as if called by `ACCTFROM` into `ACCTTO`, with apparent value transfer `APPVALUE`, gas allocation `GAVAIL`, program `CODE`, and arguments `ARGS`.

1.  Load `CALLDEPTH` from `eei.callState.callDepth`.

2.  Set `eei.callState.callDepth` to `CALLDEPTH +Int 1`.

3.  Set `eei.callState.caller` to `ACCTFROM`.

4.  Set `eei.callState.acct` to `ACCTTO`.

5.  Set `eei.callState.callValue` to `APPVALUE`.

6.  Set `eei.callState.gas` to `GAVAIL`.

7.  Set `eei.callState.program` to `CODE`.

8.  Set `eei.callState.callData` to `ARGS`.

9.  Set `eei.returnData` to the empty `.List`.

```k
    syntax EEIMethod ::= "EEI.callInit" Int Int Int Int Program List
 // ----------------------------------------------------------------
    rule <k> EEI.callInit ACCTFROM ACCTTO APPVALUE GAVAIL CODE ARGS => . ... </k>
         <callState>
           <callDepth>  CALLDEPTH => CALLDEPTH +Int 1 </callDepth>
           <caller>     _         => ACCTFROM         </caller>
           <acct>       _         => ACCTTO           </acct>
           <callValue>  _         => APPVALUE         </callValue>
           <gas>        _         => GAVAIL           </gas>
           <program>    _         => CODE             </program>
           <callData>   _         => ARGS             </callData>
         </callState>
```

#### `EEI.call : Int Int Int List`

**TODO**: Parameterize the `1024` max call depth.

Call into account `ACCTTO`, with gas allocation `GAVAIL`, apparent value `APPVALUE`, and arguments `ARGS`.

1.  Load `CALLDEPTH` from `eei.callState.callDepth`.

2.  If `CALLDEPTH >=Int 1024`, then:

    i.  Set `eei.statusCode` to `EVMC_CALL_DEPTH_EXCEEDED`.

    else:

    i.    Load `CODE` from `eei.accountss[ACCTTO].code`.

    ii.   Load `ACCTFROM` from `eei.callState.acct`.

    iii.  Call `EEI.pushCallState`.

    iv.   Call `EEI.pushAccounts`.

    vi.   Call `EEI.callInit ACCTFROM ACCTTO APPVALUE GAVAIL CODE ARGS`

    vii.  Call `EEI.execute`.

    viii. Call `EEI.popCallState`.

    viii. Call `EEI.ifStatus EEI.dropAccounts EEI.popAccounts`.

    iv.   Call `EEI.execute`.

```k
    syntax EEIMethod ::= "EEI.call" Int Int Int List
 // ------------------------------------------------
    rule <k> EEI.call ACCTTO GAVAIL APPVALUE ARGS => . ... </k>
         <statusCode> _ => EVMC_CALL_DEPTH_EXCEEDED </statusCode>
         <callDepth> CALLDEPTH </callDepth>
      requires CALLDEPTH >=Int 1024

    rule <k> EEI.call ACCTTO GAVAIL APPVALUE ARGS
          => EEI.pushCallState ~> EEI.pushAccounts
          ~> EEI.initCallState ACCTFROM ACCTTO APPVALUE GAVAIL CODE ARGS
          ~> EEI.execute
          ~> EEI.callFinish
          ~> EEI.execute
         ...
         </k>
         <acct> ACCTFROM </acct>
         <callDepth> CALLDEPTH </callDepth>
         <account>
           <id> ACCTTO </id>
           <code> CODE </code>
           ...
         </account>
      requires CALLDEPTH <Int 1024
```

#### `EEI.transferCall : Int Int Int List`

Call into account `ACCTTO`, transfering value `VALUE`, with gas allocation `GAVAIL`, and arguments `ARGS`.

1.  Call `EEI.transfer VALUE ACCT`.

2.  Call `EEI.onGoodStatus (EEI.call ACCTTO VALUE GAVAIL ARGS)`.

```k
    syntax EEIMethod ::= "EEI.transferCall" Int Int Int List
 // --------------------------------------------------------
    rule <k> EEI.transferCall ACCTTO VALUE GAVAIL ARGS
          => EEI.transfer VALUE ACCTTO
          ~> EEI.onGoodStatus (EEI.call ACCTTO VALUE GAVAIL ARGS)
         ...
         </k>
```

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
