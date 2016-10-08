# Backwards Compatibility
The current approach to achieving backwards compatibility with EVM1 is to
support both of the instruction sets with the option to transcompiling EVM1 to 
eWASM. This approach gives clients optionality when dealing with EVM1 code.
A client can either implement only an eWASM VM and transcompile all of the EVM1
code. Or a client can implement a eWASM VM and EVM1 VM and leave the old code as
is.

## Gas Prices
In eWASM we will introduce sub-gas units so that each EVM1 opcode's
transcompiled equivalent eWASM's gas cost is less then the original EM1 opcode's
cost. The fee schedule for eWASM is yet to be specified.

## Identification of code
We assume there is some sort of code handler function that all clients have 
implemented. The code handler identifies the instruction set type by whether it
starts with the *eWASM preamble* or not.

The *eWASM preamble* consists of an invalid EVM1 opcode (`0xEF`) followed by the WASM magic number.

The WASM magic number is the following byte sequence: `0x00, 0x61, 0x73, 0x6d`.

## Solidity
Support of compiling to eWASM can be accompilshed by adding a new backend to
the solidity compile. eWASM support for Solidity is part of the MVP.

## Transcompiler
A post-MVP goal is to have the transcompiler it self become a contract by
compiling it to eWASM. Once this is accomplished, EVM1 contracts created by 
the CREATE op will be transcompiled to eWASM. This will also allow us to assume
that all EVM1 code is now transcompiled eWASM code, which should be reflected
in the state root since the has of the code is stored in the Merkle trie. Note:
this should still allow clients to fallback to EVM1 VMs if running EVM1 code.
