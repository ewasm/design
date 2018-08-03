# Determining ewasm gas costs

The goal of this document is to describe the process of determining gas costs
for ewasm instructions.

Each WASM opcode is assigned an appropriate Intel IA-32 (x86) opcode (or a
series of opcodes). These opcodes have a fixed cycle count (called *latency*
by Intel). We are selecting one specific CPU model from the Haswell architecture
(family: `06`, model `3C`). This equals to CPUs produced in 2014.

**Assumption 1**: This specific Haswell CPU represents a good average of the
Ethereum nodes. We assume that a 2.2 Ghz model is the average.

According to Intel, the 2.2 Ghz clock rate roughly equals to `2 200 000 000` cycles per second.

**Assumption 2**: 1 second of CPU execution equals to 10 million gas
(i.e. 1 gas equals to 0.1 us).

This equals to **`0.0045` gas per cycle**. (`10 000 000 / 2 200 000 000`)

To put this in perspective, the average block gas limit as of August 2016 is around
4.7 million. With this assumption we allow contract execution to take up to 0.5 second
of the 15 seconds block time, which also has to include other processing, including PoW
and network roundtrip times.

**Assumption 3**: The gas costs are adjusted on a *regular* basis, at least every 3 years.

We assume that CPUs are continuously improving and the hardware for Ethereum
nodes are upgraded every 3 years (as that matches usual depreciation rates).

The upgrade procedure is not part of this document, but it is plausible to expect a
dedicated **gas cost oracle** contract will exist in the future, which can be used
by the *metering injection contract*.

## Gas vs. *Particles*

The current gas handling fields do not offer the precision needed assuming that
running WASM opcodes takes significantly less processing power compared to EVM1
opcodes.

*Rationale*: EVM1 opcodes operate on 256-bits of data, while WASM opcodes are limited
to at most 64-bits, which results in executing four instructions in the best
case to match EVM1. Most arithmetic instructions in EVM1 cost 3 gas, which would
amount to 0.75 gas for most 64-bit WASM instructions.

Internally, ewasm gas measurements should be recorded in a 64 bit variable
with 4 decimal digits precision. We call this *particles*. It is a minor
implementation detail actually using integers and converting the below gas costs
appropriately.

When converting the *particles* count to Ethereum gas, it has to be divided by
`10000` and has to be rounded up. If the result is less than `0`, then it should equal to `1`.

## Gas costs of individual instructions

The formula for determining the gas cost is: `<cycle count> * <gas per cycle>`

### Registers
|Opcode     |Cycle | IA-32 eqv. |Gas     |
|-----------|------|------------|--------|
|get_local  | 3    | MOV        | 0.0135
|set_local  | 3    | MOV        | 0.0135
|tee_local  | 3    | MOV        | 0.0135
|get_global | 3    | MOV        | 0.0135
|set_global | 3    | MOV        | 0.0135

### Memory
|Opcode        |Cycle | IA-32 eqv. |Gas     |
|--------------|------|------------|--------|
|i32.load8_s   | 3    | MOV        | 0.0135
|i32.load8_u   | 3    | MOV        | 0.0135
|i32.load16_s  | 3    | MOV        | 0.0135
|i32.load16_u  | 3    | MOV        | 0.0135
|i32.load      | 3    | MOV        | 0.0135
|i64.load8_s   | 3    | MOV        | 0.0135
|i64.load8_u   | 3    | MOV        | 0.0135
|i64.load16_s  | 3    | MOV        | 0.0135
|i64.load16_u  | 3    | MOV        | 0.0135
|i64.load32_s  | 3    | MOV        | 0.0135
|i64.load32_u  | 3    | MOV        | 0.0135
|i64.load      | 3    | MOV        | 0.0135
|grow_memory   | ?    | (breaking out from the VM)
|current_memory| ?    | (breaking out from the VM)

### Flow Control
|Opcode     |Cycle | IA-32 eqv. |
|-----------|------|------------|
|nop        | ?    |
|block      | 0    | (this is only a grouping)
|loop       | 0    | (same as block)
|if         | 0    | (only a grouping, then/else counts)
|then       | 2    | JMP (near)
|else       | 2    | JMP (near)
|br         | 2    | JMP (near)
|br_if      | 3    | CMP, JMP (near)
|br_table   | 2    | JMP (near)
|return     | 2    | RET or JMP

### Calls
|Opcode       |Cycle | IA-32 eqv. |
|-------------|------|------------|
|call         | 2    | CALL (near)
|call_indirect| ?    | CALL (far) (breaking out from the VM)
|call_import  | ?    | CALL (far) (breaking out from the VM)

### Constants
|Opcode     |Cycle | IA-32 eqv. |
|-----------|------|------------|
|i32.const  | 0    |
|i64.const  | 0    |


### 32-bit Integer operators
|Opcode     |Cycle | IA-32 eqv. | Gas    |
|-----------|------|------------|--------|
|i32.add    |1     | ADD        | 0.0045
|i32.sub    |1     | SUB        | 0.0045
|i32.mul    |3     | MUL        | 0.0135
|i32.div_s  |80    | DIV        | 0.36
|i32.div_u  |80    | DIV        | 0.36
|i32.rem_s  |80    | DIV        | 0.36
|i32.rem_u  |80    | DIV        | 0.36
|i32.and    |1     | AND        | 0.0045
|i32.or     |1     | OR         | 0.0045
|i32.xor    |1     | XOR        | 0.0045
|i32.shl    |1.5   | SHL        | 0.0067
|i32.shr_u  |1.5   | SHR        | 0.0067
|i32.shr_s  |1.5   | SHR        | 0.0067
|i32.rotl   |2     | ROL        | 0.0090
|i32.rotr   |2     | ROR        | 0.0090
|i32.eq     |1     | CMP        | 0.0045
|i32.eqz    |1     | CMP        | 0.0045
|i32.ne     |1     | CMP        | 0.0045
|i32.lt_s   |1     | CMP        | 0.0045
|i32.lt_u   |1     | CMP        | 0.0045
|i32.le_s   |1     | CMP        | 0.0045
|i32.le_u   |1     | CMP        | 0.0045
|i32.gt_s   |1     | CMP        | 0.0045
|i32.gt_u   |1     | CMP        | 0.0045
|i32.ge_s   |1     | CMP        | 0.0045
|i32.ge_u   |1     | CMP        | 0.0045
|i32.clz    |105   | (clz)      | 0.4725
|i32.ctz    |105   | (ctz)      | 0.4725
|i32.popcnt |?     |

### 64-bit Integer operators
|Opcode     |Cycle | IA-32 eqv. | Gas    |
|-----------|------|------------|--------|
|i64.add    |1     | ADD        | 0.0045
|i64.sub    |1     | SUB        | 0.0045
|i64.mul    |3     | MUL        | 0.0135
|i64.div_s  |80    | DIV        | 0.36
|i64.div_u  |80    | DIV        | 0.36
|i64.rem_s  |80    | DIV        | 0.36
|i64.rem_u  |80    | DIV        | 0.36
|i64.and    |1     | AND        | 0.0045
|i64.or     |1     | OR         | 0.0045
|i64.xor    |1     | XOR        | 0.0045
|i64.shl    |1.5   | SHL        | 0.0067
|i64.shr_u  |1.5   | SHR        | 0.0067
|i64.shr_s  |1.5   | SHR        | 0.0067
|i64.rotl   |2     | ROL        | 0.0090
|i64.rotr   |2     | ROR        | 0.0090
|i64.eq     |1     | CMP        | 0.0045
|i64.eqz    |1     | CMP        | 0.0045
|i64.ne     |1     | CMP        | 0.0045
|i64.lt_s   |1     | CMP        | 0.0045
|i64.lt_u   |1     | CMP        | 0.0045
|i64.le_s   |1     | CMP        | 0.0045
|i64.le_u   |1     | CMP        | 0.0045
|i64.gt_s   |1     | CMP        | 0.0045
|i64.gt_u   |1     | CMP        | 0.0045
|i64.ge_s   |1     | CMP        | 0.0045
|i64.ge_u   |1     | CMP        | 0.0045
|i64.clz    |?     |
|i64.ctz    |?     |
|i64.popcnt |?     |

### Datatype conversions, truncations, reinterpretations, promotions, and demotions
|Opcode          |Cycle | IA-32 eqv. | Gas    |
|----------------|------|------------|--------|
|i32.wrap/i64    |3     | MOV        | 0.0135
|i64.extend_s/i32|?     | (signextend)
|i64.extend_u/i32|3     | MOV        | 0.0135

### Type-parametric operators.
|Opcode     |Cycle | IA-32 eqv. | Gas    |
|-----------|------|------------|--------|
|drop       |3     | MOV        | 0.0135
|select     |3     | CMP, JMP   | 0.0135


### Other
|Opcode     |Cycle | IA-32 eqv. | Gas    |
|-----------|------|------------|--------|
|unreachable|0     | (breaking out from the VM - or *INT3*)

## Notes about complex instructions

Some of the above instructions are complex and cannot be mapped 1-by-1 to a machine
instruction. For them we assume the below algorithms and calculate gas based on the
component costs.

### i32.clz

Number of leading 0-bits within 32 bits.

(Based on *Warren, Section 5-3: Counting Leading 0's*:)

```
function clz(x)
    if x = 0 return 32
    n ← 0
    if (x & 0xFFFF0000) = 0: n ← n + 16, x ← x << 16
    if (x & 0xFF000000) = 0: n ← n +  8, x ← x <<  8
    if (x & 0xF0000000) = 0: n ← n +  4, x ← x <<  4
    if (x & 0xC0000000) = 0: n ← n +  2, x ← x <<  2
    if (x & 0x80000000) = 0: n ← n +  1
    return n
```

For an implementation in WebAssembly, see Appendix A.

Counting the individual instructions results in:
- 6 `if`
- 6 `then`
- 2 `return`
- 5 `and`
- 5 `eqz`,
- 11 `get_local`,
- 11 `set_local`,
- 5 `add`,
- 5 `shl`

These amount to a total of 104.5 cycles in the worst case.

Note: `clz` exists natively on most common CPUs, including Intel and ARM CPUs.
We could consider it native for the purposes of Ethereum and assign a lower gas cost.

### i32.ctz

Number of trailing 0-bits within 32 bits.

(Based on *Warren, Section 5-4: Counting Trailing 0's.*:)
```
function ctz (x)
    if x = 0 return 32
    n ← 0
    if (x & 0x0000FFFF) = 0: n ← n + 16, x ← x >> 16
    if (x & 0x000000FF) = 0: n ← n +  8, x ← x >>  8
    if (x & 0x0000000F) = 0: n ← n +  4, x ← x >>  4
    if (x & 0x00000003) = 0: n ← n +  2, x ← x >>  2
    if (x & 0x00000001) = 0: n ← n +  1
    return n
```

This algorithm has the very same steps as `i32.clz` and therefore their cost equals.

Note: `ctz` does not exists natively on ARM CPUs. ARM should be considered an
important platform for Ethereum light clients and therefore it is sensible to assign
a gas cost based on a complex implementation.

### i32.popcnt

Number of 1-bits within 32 bits.

**TBD**

Note: `popcnt` is natively supported by recent CPUs, mostly through SIMD extensions only.

### i64.clz

Number of leading 0-bits within 64 bits.

**TBD**

### i64.ctz

Number of trailing 0-bits within 64 bits.

**TBD**

### i64.popcnt

Number of 1-bits within 64 bits.

**TBD**

### i64.extend_s/i32

Sign extend i32 to i64.

Given WebAssembly supports both arithmetic (`shr_s`) and logical right shifts
(`shr_u`) it is fairly simple to implement:

```
function signextend (x)
    return (x << 32) >>> 32
```

This cost equals the sum of `i64.shl` and `i64.shr_s`, ? cycles.

**TBD**

### select

Ternary operator.

**TBD**

### breaking out of the VM

Any instruction pausing the VM and transferring data between the ewasm contract
and the host is *breaking out of the VM*.

These instructions include:
- `current_memory`
- `grow_memory`
- `call_indirect`
- `call_import`

**TBD**

## Appendix A: i32.clz in WebAssembly

```
(func $clz
  (param $x i32)
  (result i32)

  (local $n i32)

  (if (i32.eqz (get_local $x))
    (then
      (return (i32.const 32))
    )
  )

  (set_local $n (i32.const 0))

  (if (i32.eqz (i32.and (get_local $x) (i32.const 0xffff0000))
    (then
      (set_local $n (i32.add (get_local $n) (i32.const 16)))
      (set_local $x (i32.shl (get_local $x) (i32.const 16)))
    )
  )

  (if (i32.eqz (i32.and (get_local $x) (i32.const 0xff000000))
    (then
      (set_local $n (i32.add (get_local $n) (i32.const 8)))
      (set_local $x (i32.shl (get_local $x) (i32.const 8)))
    )
  )

  (if (i32.eqz (i32.and (get_local $x) (i32.const 0xf0000000))
    (then
      (set_local $n (i32.add (get_local $n) (i32.const 4)))
      (set_local $x (i32.shl (get_local $x) (i32.const 4)))
    )
  )

  (if (i32.eqz (i32.and (get_local $x) (i32.const 0xc0000000))
    (then
      (set_local $n (i32.add (get_local $n) (i32.const 2)))
      (set_local $x (i32.shl (get_local $x) (i32.const 2))
    )
  )

  (if (i32.eqz (i32.and (get_local $x) (i32.const 0x80000000))
    (then
      (set_local $n (i32.add (get_local $n) (i32.const 1)))
    )
  )

  (return (get_local $n))
)
```

## References

- Appendix C of *Architecture Optimization Manual* (http://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-optimization-manual.html)
- Haswell CPU family (http://ark.intel.com/products/codename/42174/Haswell)
- Instruction tables by Agner Fog: http://www.agner.org/optimize/instruction_tables.pdf
- EVM 1.0 gas costs (https://docs.google.com/spreadsheets/d/1m89CVujrQe5LAFJ8-YAUCcNK950dUzMQPMJBxRtGCqs/edit#gid=0)
- Warren, Henry S. (2003). Hacker's Delight (1st ed.). Boston, Mass.: Addison-Wesley. ISBN 0-201-91465-4
