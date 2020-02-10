## EVM opcodes vs ewasm methods

These tables compares EVM opcodes with ewasm methods and wasm instructions. The EVM defines 134 opcodes (as of the Byzantium hard fork). Of these 134 opcodes, some represent generic machine instructions (ADD, MUL, ISZERO, XOR, etc.) and the rest are Ethereum-specific operations (SLOAD, SSTORE, GASPRICE, BLOCKHASH, etc.).

When EVM bytecode is transpiled to ewasm bytecode, the Ethereum-specific opcodes translate to corresponding ewasm interface methods (Ethereum Environment Interface (EEI) methods). These interface methods are provided as [host functions](https://webassembly.github.io/threads/exec/runtime.html#syntax-hostfunc) to the wasm VM. All other EVM opcodes are not Ethereum-specific and do not access data from the Ethereum environment, so they reduce to pure wasm instructions.

To reference EVM opcodes, try the [yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf), the [readable yellow paper](https://github.com/chronaeon/beigepaper/blob/master/beigepaper.pdf), or the [pyethereum implementation](https://github.com/ethereum/pyethereum/blob/develop/ethereum/opcodes.py).

To reference ewasm interface methods, see the [EEI spec](https://github.com/ewasm/design/blob/master/eth_interface.md).


### EVM opcodes (Byzantium) with corresponding ewasm methods (Ethereum Environment Interface (EEI) Revision 4)

EVM opcode            |     ewasm interface method
----------------------|-----------------------------
0x00: STOP            |      finish(0, 0)
0x30: ADDRESS         |      getAddress
0x31: BALANCE         |      getBalance
0x32: ORIGIN          |      getTxOrigin
0x33: CALLER          |      getCaller
0x34: CALLVALUE       |      getCallValue
0x35: CALLDATALOAD    |      callDataCopy(resultOffset, dataOffset, 32)
0x36: CALLDATASIZE    |      getCallDataSize
0x37: CALLDATACOPY    |      callDataCopy(resultOffset, dataOffset, size)
0x38: CODESIZE        |      getCodeSize
0x39: CODECOPY        |      codeCopy
0x3a: GASPRICE        |      getTxGasPrice
0x3b: EXTCODESIZE     |      getExternalCodeSize
0x3c: EXTCODECOPY     |      externalCodeCopy
0x3d: RETURNDATASIZE  |      getReturnDataSize
0x3e: RETURNDATACOPY  |      returnDataCopy
0x40: BLOCKHASH       |      getBlockHash
0x41: COINBASE        |      getBlockCoinbase
0x42: TIMESTAMP       |      getBlockTimestamp
0x43: NUMBER          |      getBlockNumber
0x44: DIFFICULTY      |      getBlockDifficulty
0x45: GASLIMIT        |      getBlockGasLimit
0x54: SLOAD           |      storageLoad
0x55: SSTORE          |      storageStore
0x58: PC              |      -<sup>[†](#f1)</sup>
0x5a: GAS             |      getGasLeft
0xa0: LOG0            |      log(0,..)
...                   |
0xa4: LOG4            |      log(4,..)
0xf0: CREATE          |      create
0xf1: CALL            |      call
0xf2: CALLCODE        |      callCode
0xf3: RETURN          |      finish
0xf4: DELEGATECALL    |      callDelegate
0xfa: STATICCALL      |      callStatic
0xfd: REVERT          |      revert
0xff: SELFDESTRUCT    |      selfDestruct


†. <small id="f1">In EVM, the Program Counter (PC) opcode returns the currently executing bytecode position. When EVM is transpiled to ewasm with evm2wasm, this bytecode position is calculated by the transpiler and [inserted as a wasm constant](https://github.com/ewasm/evm2wasm/blob/80136977553c9b22e385f919fff808ef8a54be95/index.js#L247-L248). For ewasm code that is not transpiled from EVM, there is no interface method corresponding to the `PC` opcode.</small>


## EVM opcodes vs WebAssembly instructions

Whereas EVM is an untyped VM with a 256-bit word size, WebAssembly is a typed VM with 32-bit and 64-bit word sizes. The two wasm base types are integers and floats, which when combined with the two word sizes, create four types: i32, i64, f32, f64.

Ethereum WebAssembly, or ewasm, is a subset of wasm. Only the integer types (i32 and i64) are supported, floating point types and floating point instructions are not supported. In total, 60 wasm instructions are supported in ewasm:
* 10 control flow instructions
* 11 basic instructions
* 19 integer arithmetic instructions
* 10 integer comparison instructions
* 3 integer conversion instructions
* 7 memory instructions (load, store, extend)

ewasm excludes the wasm floating point instructions:
* 14 floating point arithmetic instructions
* 6 floating point comparison instructions
* 7 floating point conversion instructions

To reference wasm instructions, see the [wasm spec](https://github.com/WebAssembly/design/blob/master/Semantics.md) or [reference manual](https://github.com/sunfishcode/wasm-reference-manual/blob/master/WebAssembly.md#instructions).


### EVM opcodes that reduce to wasm instructions

EVM opcode (256-bit)  |    wasm instruction i32/i64 (32-bit or 64-bit)
----------------------|----------------------
0x01: ADD             |       i32.add
0x02: MUL             |       i32.mul
0x03: SUB             |       i32.sub
0x04: DIV             |       i32.div_u
0x05: SDIV            |       i32.div_s
0x06: MOD             |       i32.rem_u
0x07: SMOD            |       i32.rem_s
0x08: ADDMOD          |       -<sup>[‡](#f2)</sup>
0x09: MULMOD          |       -<sup>[‡](#f2)</sup>
0x0a: EXP             |       -<sup>[‡](#f2)</sup>
0x0b: SIGNEXTEND      |       -<sup>[‡](#f2)</sup>
0x10: LT              |       i32.lt_u
0x11: GT              |       i32.gt_u
0x12: SLT             |       i32.lt_s
0x13: SGT             |       i32.gt_s
0x14: EQ              |       i32.eq
0x15: ISZERO          |       i32.eqz
0x16: AND             |       i32.and
0x17: OR              |       i32.or
0x18: XOR             |       i32.xor
0x19: NOT             |       i32.sub (i32.const 0xffffffff)
0x1a: BYTE            |       i32.load8
0x20: SHA3            |       -<sup>[‡](#f2)</sup>
0x50: POP             |       drop
0x51: MLOAD           |       i32.load
0x52: MSTORE          |       i32.store
0x53: MSTORE8         |       i32.store
0x56: JUMP            |       br
0x57: JUMPI           |       br_if, br_table
0x59: MSIZE           |       current_memory
0x5b: JUMPDEST        |       block, loop
0xfe: INVALID         |       unreachable
0x60: PUSH1           |       i32.const
...                   |
0x7f: PUSH32          |       i32.const
0x80: DUP1            |       get_global, get_local
...                   |
0x8f: DUP16           |       get_global, get_local
0x90: SWAP1           |       get_global, get_local
...                   |
0x9f: SWAP16          |       get_global, get_local


‡. <small id="f2">Some EVM opcodes, such as `ADDMOD` (modular addition), `EXP` (exponentiation), or `SHA3` (Keccak-256 hash), do not have simple reductions to single wasm instructions (or to only a few wasm instructions). The functionality of these opcodes is replicated by a relatively large wasm module (for instance, `SHA3` reduces to about [900 lines of wast code](https://github.com/ewasm/evm2wasm/blob/80136977553c9b22e385f919fff808ef8a54be95/wasm/keccak.wast)).</small>
