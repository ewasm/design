# Ethereum flavored WebAssembly (ewasm) Design (*Revision 3*)

This repository contains documents describing the design and high-level overview of ewasm. Expect the contents of this repository to be in flux: everything is still under discussion.

## What is WebAssembly?

> WebAssembly (or Wasm as a contraction) is a new, portable, size- and load-time-efficient format. WebAssembly aims to execute at native speed by taking advantage of common hardware capabilities available on a wide range of platforms. WebAssembly is currently being designed as an open standard by a W3C Community Group.

Please review the [WebAssembly](http://webassembly.org/) [design](http://webassembly.org/docs/high-level-goals/) and [instruction set]() first. (You can also make a pull request or raise an issue at the [Wasm Github repo](https://github.com/WebAssembly/design).)

A few key points:
* WebAssembly defines an instruction set, intermediate source format (WAST) and a binary encoded format (WASM).
* WebAssembly has a few higher level features, such as the ability to import and execute outside methods defined via an interface.
* [LLVM](https://llvm.org/) includes a WebAssembly backend to generate WASM output.
* Major browser JavaScript engines will notably have native support for
  WebAssembly, including but not limited to: Google's
  [V8](https://github.com/v8/v8) engine (Node.js and Chromium-based browsers),
  Microsoft's [Chakra](https://github.com/Microsoft/ChakraCore) engine
  (Microsoft Edge), Mozilla's
  [Spidermonkey](https://github.com/mozilla/gecko-dev/tree/master/js) engine
  (Firefox and Thunderbird).
* Other non-browser implementations exist too:
  [wasm-jit-prototype](https://github.com/WebAssembly/wasm-jit-prototype) (a
  standalone VM using an LLVM backend),
  [wabt](https://github.com/WebAssembly/wabt) (a stack-based interpreter),
  [ml-proto](https://github.com/WebAssembly/spec/tree/master/ml-proto) (the
  OCaml reference interpreter), etc.

## What is Ethereum flavored WebAssembly (ewasm)?

ewasm is a restricted subset of WASM to be used for contracts in Ethereum.

ewasm:
* specifies the [VM semantics](./vm_semantics.md)
* specifies the [semantics for an *ewasm contract*](./contract_interface.md)
* specifies an [Ethereum environment interface](./eth_interface.md) to facilitate interaction with the Ethereum environment from an *ewasm contract*
* specifies [system contracts](./system_contracts.md)
* specifies [metering](./metering.md) for instructions
* and aims to restrict [non-deterministic behavior](https://github.com/WebAssembly/design/blob/master/Nondeterminism.md)
* specifies a backwards compatible upgrade path to EVM1

### Goals of the ewasm project

* To provide a specification of *ewasm contract* semantics and the *Ethereum interface*
* To provide an *EVM transpiler*, preferably as an ewasm contract
* To provide a *metering injector*, preferably as an ewasm contract
* To provide a VM implementation for executing ewasm contracts
* To implement an ewasm backend in the Solidity compiler
* To provide a library and instructions for writing contracts in Rust
* To provide a library and instructions for writing contracts in C

### Glossary

* *Ewasm contract*: a contract adhering to the ewasm specification
* *Ethereum environment interface (EEI)*: a set of methods available to ewasm contracts
* *metering*: the act of measuring execution cost in a deterministic way
* *metering injector*: a transformation tool inserting metering code to an ewasm contract
* *EVM transpiler*: an EVM bytecode (the current Ethereum VM) to ewasm transcompiler

### Resources

* [FAQ](./faq.md)
* [Rationale](./rationale.md)
* [VM semantics](./vm_semantics.md)
* [Ethereum environment interface](./eth_interface.md)
* [Ewasm Contract Interface](./contract_interface.md)
* [System contracts](./system_contracts.md)
* [Backwards compatibility instructions](./backwards_compatibility.md)
* [Original Proposal](https://github.com/ethereum/EIPs/issues/48) (EIP#48)
* [WebAssembly Specification](https://github.com/WebAssembly/spec/blob/md-proto/md-proto/WebAssembly.md)
* [WebAssembly design documents](https://github.com/WebAssembly/design)

### Projects

* [ewasm-tests](https://github.com/ewasm/ewasm-tests)
* [wasm-metering](https://github.com/ewasm/wasm-metering)
* [ewasm-kernel](https://github.com/ewasm/ewasm-kernel)
* [evm2wasm](https://github.com/ewasm/evm2wasm)
* [ewasm-libc](https://github.com/ewasm/ewasm-libc)
* [assemblyscript-ewasm-api](https://github.com/ewasm/assemblyscript-ewasm-api)
* [ewasm-rust-api](https://github.com/ewasm/ewasm-rust-api)

### Design Process & Contributing
For now, high-level design discussions should continue to be held in the design repository, via issues and pull requests. Feel free to file [issues](https://github.com/ethereum/ewasm-design/issues).

## Chat
[Matrix](https://riot.im/app/#/room/#the_vertex:matrix.org)  
IRC freenode@ewasm  
[Gitter](https://gitter.im/ewasm/Lobby)  
