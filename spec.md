ewasm specification
===================

Introduction
------------

This document aims to specify an ewasm VM in a way useful to contract writers and VM implementers.
To this end, multiple things are specified:

-   The extra state that a VM needs to have around to successfully respond to calls into the EEI.
-   The EEI (Ethereum Environment Interface), currently specified loosley [here](eth_interface.md).

### Notation

We are using [K Framework] notation to specify the EEI, which makes this specification executable.

```k
requires "domains.k"

module EEI
    imports DOMAINS
```

Execution State
---------------

First, we must specify the extra state that must be present for ewasm execution.
We do that by specifying a K *configuration*.

Each XML-like *cell* contains a field which is relevant to Ethereum client execution.
The default/initial values of the cells are provided along with the declaration of the configuration.

The `multiplicity="*"` allows us to have multiple accounts simultaneously, and `type="Map"` allows us to access accounts by using the `<acctID>` as a key.
For example, `eei.accounts[0x00001].nonce` would access the nonce of account `0x00001`.
Similarly, cells that contain a `List` data-type can be indexed using standard array-access notation, eg `L[N]` gets the `N`th element of list `L` (starting at `0`).

```k
    configuration
      <eei>
        <eeiOP>       .EEIOp      </eeiOP>
        <eeiResponse> .K          </eeiResponse>
        <statusCode>  .StatusCode </statusCode>
```

The `<callState>` sub-configuration can be saved/restored when needed between calls.

```k
        <callState>
          <callDepth>  0     </callDepth>
          <returnData> .List </returnData>

          // I_*
          <id>        0        </id>        // I_a
          <program>   .Program </program>   // I_b
          <caller>    0        </caller>    // I_s
          <callData>  .List    </callData>  // I_d
          <callValue> 0        </callValue> // I_v

          // \mu_*
          <gas>         0 </gas>            // \mu_g
          <memoryUsed>  0 </memoryUsed>     // \mu_i
          <previousGas> 0 </previousGas>
        </callState>
```

The `<accounts>` sub-configuration stores information about each account on the blockchain.

```k
        <accounts>
          <account multiplicity="*" type="Map">
            <acctID>  0        </acctID>
            <balance> 0        </balance>
            <code>    .Program </code>
            <storage> .Map     </storage>
            <nonce>   0        </nonce>
          </account>
        </accounts>
```

Transaction and block information:

```k
        <gasPrice> 0 </gasPrice> // I_p
        <origin>   0 </origin>   // I_o

        <previousHash>     0          </previousHash>     // H_p
        <ommersHash>       0          </ommersHash>       // H_o
        <coinbase>         0          </coinbase>         // H_c
        <stateRoot>        0          </stateRoot>        // H_r
        <transactionsRoot> 0          </transactionsRoot> // H_t
        <receiptsRoot>     0          </receiptsRoot>     // H_e
     // <logsBloom>        .WordStack </logsBloom>        // H_b
        <difficulty>       0          </difficulty>       // H_d
        <blockNumber>      0          </blockNumber>      // H_i
        <gasLimit>         0          </gasLimit>         // H_l
        <gasUsed>          0          </gasUsed>          // H_g
        <timestamp>        0          </timestamp>        // H_s
     // <extraData>        .WordStack </extraData>        // H_x
        <mixHash>          0          </mixHash>          // H_m
        <blockNonce>       0          </blockNonce>       // H_n
     // <ommerBlockHeaders> [ .JSONList ] </ommerBlockHeaders>
        <blockhash>         .List         </blockhash>
      </eei>
```

In the texual rules below, we'll refer to cells by accesing subcells with the `.` operator.
For example, we would access the `statusCode` cell with `eei.statusCode`.

Data
----

### Abstract Programs

Different VMs have different representations of programs.
Here, we make a sort `Program` which has a single constant `.Program` to use as the default.

```k
    syntax Program ::= ".Program"
```

### Status Codes

The [EVMC] status codes are used to indicate to the client how VM execution ended.
Currently, they are broken into three subsorts, for exceptional, ending, or error statuses.
The extra status code `.StatusCode` is used as a default status code when none has been set.

```k
    syntax StatusCode ::= ExceptionalStatusCode
                        | EndStatusCode
                        | ErrorStatusCode
                        | ".StatusCode"
```

#### Exceptional Codes

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
-   `EVMC_PRECOMPILE_FAILURE` indicates an errors in the precompiled contracts (eg. invalid points handed to elliptic curve functions).

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

#### Ending Codes

These additional status codes indicate that execution has ended in some non-exceptional way.

-   `EVMC_SUCCESS` indicates successful end of execution.
-   `EVMC_REVERT` indicates that the contract called `REVERT`.

```k
    syntax EndStatusCode ::= ExceptionalStatusCode
                           | "EVMC_SUCCESS"
                           | "EVMC_REVERT"
```

#### Other Codes

The following codes indicate other non-execution errors with the VM.

-   `EVMC_REJECTED` indicates malformed or wrong-version EVM bytecode.
-   `EVMC_INTERNAL_ERROR` indicates some other error that is unrecoverable but not due to the bytecode.

```k
    syntax ErrorStatusCode ::= "EVMC_REJECTED"
                             | "EVMC_INTERNAL_ERROR"
```

EEI Methods
-----------

The EEI exports several methods which can be invoked by any of the VMs when appropriate.
Here the syntax and semantics of these methods is defined.
The special EEIOp `.EEIOp` is the "no-op" or "skip" method.

```k
    syntax EEIOp ::= ".EEIOp"
```

In the semantics below, we'll give both a texual description of the state updates for each method, and the K rule.
Each section header gives the name of the given EEI method, along with the arguments needed.
For example, `EEI.useGas : Int` declares that `EEI.useGas` in an EEI method which takes a single integer as input.

### EEI Getters

Many of the methods exported by the EEI simply query for somestate of the execution environment.
These methods are prefixed with `get`, and have largely similar and simple rules.

#### `EEI.getAddress`

Return the address of the currently executing account.

1.  Load and return the value `eei.id`.

```k
    syntax EEIOp ::= "EEI.getAddress"
 // ---------------------------------
    rule <eeiOP>       EEI.getAddress => .EEIOp </eeiOP>
         <eeiResponse> _              => ADDR   </eeiResponse>
         <id>          ADDR                     </id>
```

#### `EEI.getBalance : Int`

Return the balance of the given account (`ACCT`).

1.  Load and return the value `eei.accounts[ACCT].balance`.

```k
    syntax EEIOp ::= "EEI.getBalance" Int
 // -------------------------------------
    rule <eeiOP>       EEI.getBalance ACCT => .EEIOp </eeiOP>
         <eeiResponse> _                   => BAL    </eeiResponse>
         <account>
           <acctID>  ACCT </acctID>
           <balance> BAL  </balance>
           ...
         </account>
```

#### `EEI.getBlockHash : Int`

Return the blockhash of one of the `N`th most recent complete blocks (as long as `N <Int 256`).

1.  If `N <Int 256`:

    i.  then: Load and return `eei.blockhash[N]`.

    ii. else: Return `0`.

```k
    syntax EEIOp ::= "EEI.getBlockHash" Int
 // ---------------------------------------
    rule <eeiOP>       EEI.getBlockHash N => .EEIOp       </eeiOP>
         <eeiResponse> _                  => BLKHASHES[N] </eeiResponse>
         <blockhash>   BLKHASHES                          </blockhash>
      requires N <Int 256

    rule <eeiOP>       EEI.getBlockHash N => .EEIOp </eeiOP>
         <eeiResponse> _                  => 0      </eeiResponse>
      requires N >=Int 256
```

#### `EEI.callDataCopy`

-   `callDataSize` can be implemented client-side in terms of this opcode.

Returns the calldata associated with this call.

1.  Load and return `eei.callData`.

```k
    syntax EEIOp ::= "EEI.callDataCopy"
 // -----------------------------------
    rule <eeiOP>       EEI.callDataCopy => .EEIOp </eeiOP>
         <eeiResponse> _                => CDATA  </eeiResponse>
         <callData>    CDATA                      </callData>
```

#### `EEI.getCaller`

Get the account id of the caller into the current execution.

1.  Load and return `eei.caller`.

```k
    syntax EEIOp ::= "EEI.getCaller"
 // --------------------------------
    rule <eeiOP>       EEI.getCaller => .EEIOp </eeiOP>
         <eeiResponse> _             => CACCT  </eeiResponse>
         <caller>      CACCT                   </caller>
```

#### `EEI.getCallValue`

Get the value transferred for the current call.

1.  Load and return `eei.callValue`.

```k
    syntax EEIOp ::= "EEI.getCallValue"
 // -----------------------------------
    rule <eeiOP>       EEI.getCallValue => .EEIOp </eeiOP>
         <eeiResponse> _                => CVALUE </eeiResponse>
         <callValue>   CVALUE                     </callValue>
```

#### `EEI.codeCopy : Int`

-   `getCodeSize` and `getExternalCodeSize` can be implemented in terms of this method.
-   This implements what is traditionally `externalCodeCopy`, but traditional `codeCopy` can be implemented in terms of this as well.

Get the code of the given account `ACCT`.

1.  Load and return `eei.accounts[ACCT].code`.

```k
    syntax EEIOp ::= "EEI.codeCopy" Int
 // -----------------------------------
    rule <eeiOP>       EEI.codeCopy ACCT => .EEIOp   </eeiOP>
         <eeiResponse> _                 => ACCTCODE </eeiResponse>
         <accounts>
           <acctID> ACCT </acctID>
           <code> ACCTCODE </code>
           ...
         </accounts>
```

#### `EEI.getBlockCoinbase`

Get the coinbase of the current block.

1.  Load and return `eei.coinbase`.

```k
    syntax EEIOp ::= "EEI.getBlockCoinbase"
 // ---------------------------------------
    rule <eeiOP>       EEI.getBlockCoinbase => .EEIOp </eeiOP>
         <eeiResponse> _                    => CBASE  </eeiResponse>
         <coinbase>    CBASE                          </coinbase>
```

#### `EEI.getBlockDifficulty`

Get the difficulty of the current block.

1.  Load and return `eei.difficulty`.

```k
    syntax EEIOp ::= "EEI.getBlockDifficulty"
 // -----------------------------------------
    rule <eeiOP>       EEI.getBlockDifficulty => .EEIOp </eeiOP>
         <eeiResponse> _                      => DIFF   </eeiResponse>
         <difficulty>  DIFF                             </difficulty>
```

#### `EEI.getGasLeft`

Get the gas left available for this execution.

1.  Load and return `eei.gas`.

```k
    syntax EEIOp ::= "EEI.getGasLeft"
 // ---------------------------------
    rule <eeiOP>       EEI.getGasLeft => .EEIOp </eeiOP>
         <eeiResponse> _              => GAVAIL </eeiResponse>
         <gas>         GAVAIL                   </gas>
```

#### `EEI.getBlockGasLimit`

Get the gas limit for the current block.

1.  Load and return `eei.gasLimit`.

```k
    syntax EEIOp ::= "EEI.getBlockGasLimit"
 // ---------------------------------------
    rule <eeiOP>       EEI.getBlockGasLimit => .EEIOp </eeiOP>
         <eeiResponse> _                    => GLIMIT </eeiResponse>
         <gasLimit>    GLIMIT                         </gasLimit>
```

#### `EEI.getTxGasPrice`

Get the gas price of the current transation.

1.  Load and return `eei.gasPrice`.

```k
    syntax EEIOp ::= "EEI.getTxGasPrice"
 // ------------------------------------
    rule <eeiOP>       EEI.getTxGasPrice => .EEIOp </eeiOP>
         <eeiResponse> _                 => GPRICE </eeiResponse>
         <gasPrice>    GPRICE                      </gasPrice>
```

#### `EEI.getBlockNumber`

Get the current block number.

1.  Load and return `eei.blockNumber`.

```k
    syntax EEIOp ::= "EEI.getBlockNumber"
 // -------------------------------------
    rule <eeiOP>       EEI.getBlockNumber => .EEIOp    </eeiOP>
         <eeiResponse> _                  => BLKNUMBER </eeiResponse>
         <blockNumber> BLKNUMBER                       </blockNumber>
```

#### `EEI.getTxOrigin`

Get the address which sent this transaction.

1.  Load and return `eei.origin`.

```k
    syntax EEIOp ::= "EEI.getTxOrigin"
 // ----------------------------------
    rule <eeiOP>       EEI.getTxOrigin => .EEIOp </eeiOP>
         <eeiResponse> _               => ORG    </eeiResponse>
         <origin>      ORG                       </origin>
```

#### `EEI.returnDataCopy`

-   `getReturnDataSize` can be implemented in terms of this method.

Get the return data of the last call.

1.  Load and return `eei.returnData`.

```k
    syntax EEIOp ::= "EEI.returnDataCopy"
 // -------------------------------------
    rule <eeiOP>       EEI.returnDataCopy => .EEIOp  </eeiOP>
         <eeiResponse> _                  => RETDATA </eeiResponse>
         <returnData>  RETDATA                       </returnData>
```

#### `EEI.getBlockTimestamp`

Get the timestamp of the last block.

1.  Load and return `eei.timestamp`.

```k
    syntax EEIOp ::= "EEI.getBlockTimestamp"
 // ----------------------------------------
    rule <eeiOP>       EEI.getBlockTimestamp => .EEIOp </eeiOP>
         <eeiResponse> _                     => TSTAMP </eeiResponse>
         <timestamp>   TSTAMP                          </timestamp>
```

### EEI Interaction Methods

The remaining methods have more complex interactions with the EEI, either setting/getting account information or triggering further computation.

#### `EEI.selfDestruct` **TODO**

#### `EEI.return` **TODO**

#### `EEI.revert` **TODO**

#### `EEI.useGas : Int`

Deduct the specified amount of gas (`GDEDUCT`) from the available gas.

1.  Load the value `GAVAIL` from `eei.gas`.

2.  If `GDEDUCT <=Int GAVAIL`:

    i.  then: Set `eei.gas` to `GAVAIL -Int GDEDUCT`.

    ii. else: Set `eei.statusCode` to `EVMC_OUT_OF_GAS` and `eei.gas` to `0`.

```k
    syntax EEIOp ::= "EEI.useGas" Int
 // ---------------------------------
    rule <eeiOP> EEI.useGas GDEDUCT => .EEIOp              </eeiOP>
         <gas>   GAVAIL             => GAVAIL -Int GDEDUCT </gas>
      requires GAVAIL >=Int GDEDUCT

    rule <eeiOP>      EEI.useGas GDEDUCT => .EEIOp          </eeiOP>
         <gas>        GAVAIL             => 0               </gas>
         <statusCode> _                  => EVMC_OUT_OF_GAS </statusCode>
      requires GAVAIL <Int GDEDUCT
```

#### `EEI.call : Int ByteString ByteString`

-   `EEI.call` **TODO**
-   `EEI.callCode` **TODO**
-   `EEI.callDelegate` **TODO**
-   `EEI.callStatic` **TODO**

**TODO:** Implement one abstract-level `EEI.call`, akin to `#call` in KEVM, which other `CALL*` opcodes can be expressed in terms of.

#### `EEI.storageStore : Int Int`

At the given `INDEX` in the executing accounts storage, stores the given `VALUE`.

1.  Load `ACCT` from `eei.id`.

2.  Set `eei.accounts[ACCT].storage[INDEX]` to `VALUE`.

```k
    syntax EEIOp ::= "EEI.storageStore" Int Int
 // -------------------------------------------
    rule <eeiOP> EEI.storageStore INDEX VALUE => .EEIOp </eeiOP>
         <id>    ACCT </id>
         <account>
           <acctID> ACCT </acctID>
           <storage> STORAGE => STORAGE [ INDEX <- VALUE ] </storage>
           ...
         </account>
```

#### `EEI.storageLoad : Int`

Returns the value at the given `INDEX` in the current executing accounts storage.

1.  Load `ACCT` from `eei.id`.

2.  If `eei.accounts[ACCT].storage[INDEX]` exists:

    i.  then: return `eei.accounts[ACCT].storage[INDEX]`.

    ii. else: return `0`.

```k
    syntax EEIOp ::= "EEI.storageLoad" Int
 // --------------------------------------
    rule <eeiOP>       EEI.storageLoad INDEX => .EEIOp </eeiOP>
         <eeiResponse> _                     => VALUE  </eeiResponse>
         <id> ACCT </id>
         <account>
           <acctID> ACCT </acctID>
           <storage> ... INDEX |-> VALUE ... </storage>
           ...
         </account>

    rule <eeiOP>       EEI.storageLoad INDEX => .EEIOp </eeiOP>
         <eeiResponse> _                     => 0      </eeiResponse>
         <id> ACCT </id>
         <account>
           <acctID> ACCT </acctID>
           <storage> STORAGE </storage>
           ...
         </account>
      requires notBool INDEX in_keys(STORAGE)
```

#### `EEI.create` **TODO**

#### `EEI.log` **TODO**

```k
endmodule
```

Resources
=========

[K Framework]: <https://github.com/kframework/k>
[EVMC]: <https://github.com/ethereum/evmc>