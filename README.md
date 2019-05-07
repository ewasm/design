# Ethereum flavored WebAssembly (ewasm)

Specification **Revision 4**

This is the master Ewasm documentation and design repository, and contains canonical information on the current state of the Ewasm project including both a high-level overview of the project, its goals, and its rationale, as well as more technical specifications, and lists of additional resources. This is the starting point for those who wish to learn more about Ewasm, to contribute to Ewasm, to build Ewasm-compatible smart contracts on Ethereum, or to implement Ewasm in a blockchain other than Ethereum.

**This repository is also available through [ReadTheDocs](https://ewasm.readthedocs.io).**

## What is WebAssembly?

WebAssembly (or Wasm, for short: note that the term "Wasm" is not an acronym and is therefore not written using all-caps) is a new, portable, size- and load-time-efficient bytecode format. WebAssembly aims to execute at near-native speed by taking advantage of common hardware capabilities available on a wide range of platforms. WebAssembly is currently being designed as an open standard by a W3C Community Group.

Please review the [WebAssembly](http://webassembly.org/) [design](http://webassembly.org/docs/high-level-goals/) and [instruction set](https://webassembly.github.io/spec/core/appendix/index-instructions.html#index-instr) first. (You can also make a pull request or raise an issue at the [Wasm Github repo](https://github.com/WebAssembly/design).)

A few key points:
* WebAssembly defines an instruction set, intermediate source format (Wast) and a binary encoded format (Wasm).
* WebAssembly has a few higher level features, such as the ability to import and execute outside methods defined via an interface.
* [LLVM](https://llvm.org/) includes a WebAssembly backend to generate Wasm output.
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

## What is Ethereum-flavored WebAssembly (Ewasm)?

Ewasm is a restricted subset of Wasm to be used for contracts in Ethereum.

Ewasm:
* specifies the [VM semantics](./vm_semantics.md)
* specifies the [semantics for an *Ewasm contract*](./contract_interface.md)
* specifies an [Ethereum environment interface](./eth_interface.md) (a.k.a., EEI methods) to facilitate interaction with the Ethereum environment from an *Ewasm contract*
* specifies [system contracts](./system_contracts.md)
* specifies [metering](./metering.md) for instructions
* and aims to restrict [non-deterministic behavior](https://github.com/WebAssembly/design/blob/master/Nondeterminism.md)
* specifies a backwards compatible upgrade path to EVM1

In short:

Ewasm = Wasm - non-determinism (floating point) + metering + EEI methods

### Goals of the Ewasm project

* To provide a specification of *ewasm contract* semantics and the *Ethereum interface*
* To provide an *EVM transcompiler*, preferably as an ewasm contract
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
* *EVM transcompiler*: an EVM bytecode (the current Ethereum VM) to ewasm transcompiler. [See this chapter](./evm_transcompiler.md).

### Resources

* [FAQ](./faq.md)
* [Rationale](./rationale.md)
* [VM semantics](./vm_semantics.md)
* [Ethereum environment interface](./eth_interface.md)
* [Ewasm Contract Interface](./contract_interface.md)
* [System contracts](./system_contracts.md)
* [Backwards compatibility instructions](./backwards_compatibility.md)
* [Original Proposal](https://github.com/ethereum/EIPs/issues/48) (EIP#48)
* [WebAssembly Specification](https://webassembly.github.io/spec/)
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

## Contributing

The best place to start contributing to the Ewasm project is our [public Gitter channel](https://gitter.im/ewasm/Lobby). The Ewasm team hangs out in this channel and is on hand to answer questions.
