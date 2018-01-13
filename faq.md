# FAQ

WASM's FAQ can be found [here](https://github.com/WebAssembly/design/blob/master/FAQ.md) 

## Is eWASM primarily the replacement for EVM?  
Currently it is being researched as a replacement instuction set for EVM1. Other instuction sets have been considered but so far WASM seems the most suitable. 

## What are alternatives to WASM?  
Some that have been considered are [here](./comparison.md)

## What are the benefits?   
* Performance 
* A well Standized ISA, that will be widely deployed
* AST byte; which can decouple metering and make it more preforment.
* Shared tooling / Broader Tooling Compatibility

## What is metering?  
Metering VMs is the same concept as electrical power companies have when charging you for the amount of eletricity that you used. With VM's we attempt to get a measurement of computation time of some code and instead of eletricity used, you are charged for the CPU's time used. We call this metering.

## Will Solidity/Serpent be compatible with eWASM, or will another HLL have to be created?  
Not off the bat, a transpiler will have to be created to compile exsiting EVM code into eWASM. As far as other High level langauges you should be able to use an language that can be compiled by LLVM ( c/c++/rust/go)

## How does eWASM handle non-determinism when a variety of programming languages are allowed to be used?
Part of the project goal is to eliminate nasal-demons. It's in the MVP. There are still a couple of edge case like sign values on NaNs but they can be canonicalized by AST transforms.  

## Will eWASM be compatible with WASM?  
Yes, the Ethereum System Interface can also be written in WASM.

## Can eWASM be built even if WASM is not currently complete or will we need to wait for its completion/MVP?   
Yes, but we would loss the "Shared tooling" benifit. So It might not make sense.

