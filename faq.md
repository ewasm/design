# FAQ

WASM's FAQ can be found [here](https://github.com/WebAssembly/design/blob/master/FAQ.md) 

## Is ewasm primarily the replacement for EVM?  
Currently it is being researched as a replacement instruction set for EVM1. Other instruction sets have been considered but so far WASM seems the most suitable.

## What are alternatives to WASM?  
Some that have been considered are [here](./comparison.md)

## What are the benefits?   
* Performance 
* A well Standardized ISA, that will be widely deployed
* Stack machine; which can decouple metering, make it more performant and can be transformed to any machine on target architecture
* Shared tooling / Broader Tooling Compatibility

## What is metering?  
Metering VMs is the same concept as electrical power companies have when charging you for the amount of electricity that you used. With VM's we attempt to get a measurement of computation time of some code and instead of electricity used, you are charged for the CPU's time used. We call this metering.

## Will Solidity/Serpent be compatible with ewasm, or will another HLL have to be created?  
Not off the bat, a transpiler will have to be created to compile existing EVM code into ewasm. As far as other High level languages you should be able to use a language that can be compiled by LLVM ( c/c++/rust/go)

## How does ewasm handle non-determinism when a variety of programming languages are allowed to be used?
Part of the project goal is to eliminate nasal-demons. It's in the MVP. There are still a couple of edge case like sign values on NaNs but they can be canonicalized by AST transforms.  

## Will ewasm be compatible with WASM?  
Yes, the Ethereum System Interface can also be written in WASM.

## Can ewasm be built even if WASM is not currently complete or will we need to wait for its completion/MVP?   
Yes, but we would lose the "Shared tooling" benefit. So It might not make sense.

## Does eWASM use synchronous or asynchronous methods?
An answer to this question as well as the reasoning for both can be [read here](./interface_questions.md).