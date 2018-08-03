# System Contracts (Revision 0)

System contracts are interfaces defined as contracts, which are essential or
recommended for an ewasm VM.

An ewasm VM implementation may opt to implement these interfaces natively
or to rely on implementations written in ewasm.

Each of these contracts have a pre defined address and can be executed through
regular contract invocations.

## Sentinel Contract

Address: `0x000000000000000000000000000000000000000a`

Every newly deployed ewasm contract must be processed by the *Sentinel Contract*
prior to including the code in the state. The sentinel will perform three very
important processing steps:
- Validate ewasm semantics
- Inject metering code
- Wrap the result in the deployer preamble (*see Appendix A*)

Input:
- **variable length**: *ewasm contract code*

Output:
- **variable length**: *ewasm contract code*

## EVM Transcompiler

Address: `0x000000000000000000000000000000000000000b`

Transcompiles EVM1 bytecode into ewasm bytecode.

The use of this is optional. A compatible client may implement EVM1 natively or
may choose to use this transcompiler.

Input:
- **variable length**: *EVM1 contract code*

Output:
- **variable length**: *ewasm contract code*

## EVM Precompiled Contracts

Precompiled contracts are defined for EVM1 (see the Yellow Paper). Several
extensions have been proposed as *[Ethereum Improvement Proposals](http://github.com/ethereum/EIPs)*.

We assume the contracts defined in the Yellow Paper still apply for ewasm.

### ecrecover

Address: `0x0000000000000000000000000000000000000001`

Calculates the corresponding Ethereum address for the public key which created the given signature.

Input:
- **32 bytes**: *message hash*
- **32 bytes**: *recovery id*
- **32 bytes**: *R component*
- **32 bytes**: *S component*

Output:
- **32 bytes**: *Ethereum address* (left padded with zeroes)

### sha2-256

Address: `0x0000000000000000000000000000000000000002`

Returns the SHA2-256 hash of the input.

Input:
- **variable length**: *input data*

Output:
- **32 bytes**: *sha2-256 hash*

### ripemd160

Address: `0x0000000000000000000000000000000000000003`

Returns the RIPEMD-160 hash of the input.

Input:
- **variable length**: *input data*

Output:
- **32 bytes**: *ripemd160 hash* (left padded with zeroes)

### identity

Address: `0x0000000000000000000000000000000000000004`

Copies the input data to the output.

Input:
- **variable length**: *input data*

Output:
- **variable length**: *output data*

### keccak256

Address: `0x0000000000000000000000000000000000000009`

Returns the KECCAK-256 hash of the input. It is being used by the EVM Transcompiler.

Input:
- **variable length**: *input data*

Output:
- **32 bytes**: *keccak-256 hash*

### Appendix A: ewasm deployer preamble

```
;;
;; Standard ewasm deployer code.
;;
;; We keep the to-be-deployed contract as a memory segment and simply return it.
;;

(module
  (memory 1
    (segment 0 "\10\00\00\00")     ;; Here comes the size of the code in LSB
    (segment 4 "Hello World CODE") ;; Here comes the code as a escaped hex string
  )
  (export "memory" memory)
  (export "main" $main)
  (import $ethereum_return "ethereum" "return" (param i32 i32))
  (func $main
    (call_import $ethereum_return (i32.const 4) (i32.load (i32.const 0)))
  )
)
```
