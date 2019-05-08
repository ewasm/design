# Ethereum flavored WebAssembly (ewasm)

Specification **Revision 4**

This is the master Ewasm documentation and design repository and contains canonical information on the current state of the Ewasm project including a high-level overview of the project, its goals, and its rationale, as well as more technical specifications, and lists of additional resources. This is the starting point for those who wish to learn more about Ewasm, to contribute to Ewasm, to build Ewasm-compatible smart contracts on Ethereum, or to implement Ewasm in a blockchain other than Ethereum.

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

## What is Ewasm?

The term "Ewasm" stands for "Ethereum-flavored" WebAssembly. Ewasm is a restricted subset of Wasm used for smart contracts in Ethereum, which also adds things like metering and a set of Wasm host functions to allow Wasm contracts to interact with Ethereum.

### Can you explain that a bit further?

Full Wasm has [four types](https://webassembly.github.io/spec/core/syntax/types.html): `i32`, `i64`, `f32`, and `f64`. Ewasm is the subset of two types: `i32` and `i64` (floats are valid types in Wasm, but invalid types in Ewasm because they can lead to [non-deterministic behavior](https://github.com/WebAssembly/design/blob/master/Nondeterminism.md)). Because Ewasm is just Wasm, Ewasm can be executed on any generic Wasm engine. Furthermore, because Ewasm is just Wasm, there is no such thing as an "Ewasm engine" or an "Ewasm VM"; these are confusing names for using a standard Wasm VM in an Ethereum client.

If Ewasm is just wasm, then what does the "E" stand for? The "E" (Ethereum-flavored) stands for the Ethereum-specific *host functions* that the Ethereum client provides to the Wasm engine. A host function in Wasm would be called a Foreign Function Interface ([FFI](https://en.wikipedia.org/wiki/Foreign_function_interface)) or a "language binding" in more generic terminology. The definition of host function in the Wasm spec [begins as](https://webassembly.github.io/spec/core/exec/runtime.html#function-instances):

> A *host function* is a function expressed outside WebAssembly but passed to a module as an import. The definition and behavior of host functions are outside the scope of this specification.

In Wasm terminology, the program that provides host functions to the Wasm engine is called the [Embedder](https://webassembly.github.io/spec/core/intro/overview.html#embedder):

> **Embedder** - A WebAssembly implementation will typically be embedded into a host environment. This environment defines how loading of modules is initiated, how imports are provided (including host-side definitions), and how exports can be accessed. However, the details of any particular embedding are beyond the scope of this specification, and will instead be provided by complementary, environment-specific API definitions.

The environment-specific API definitions for Ewasm host functions is what we call the [Ethereum Environment Interface](https://github.com/ewasm/design/blob/master/eth_interface.md), or EEI (which is a [work-in-progress](https://github.com/ewasm/design/pulls), and evolving). The embedder that implements EEI methods is an Ethereum client, which  provides them as host function imports to a generic Wasm engine.  We'll also use the term "Ewasm client" (short for an Ethereum client with an embedded Wasm engine), as in "btw, does your ewasmJS client have all the EEI methods implemented yet?"

Ewasm:
* specifies the [VM semantics](./vm_semantics.md)
* specifies the [semantics for an *Ewasm contract*](./contract_interface.md)
* specifies an [Ethereum environment interface](./eth_interface.md) (a.k.a., EEI methods) to facilitate interaction with the Ethereum environment from an *Ewasm contract*
* specifies [system contracts](./system_contracts.md)
* specifies [metering](./metering.md) for instructions
* aims to restrict [non-deterministic behavior](https://github.com/WebAssembly/design/blob/master/Nondeterminism.md)
* specifies a backwards-compatible upgrade path to EVM

In short:

Ewasm = Wasm - non-determinism (floating point) + metering + EEI methods

## Goals of the Ewasm project

* To provide a specification of *ewasm contract* semantics and the *Ethereum interface*
* To provide an *EVM transcompiler*, preferably as an ewasm contract
* To provide a *metering injector*, preferably as an ewasm contract
* To provide a VM implementation for executing ewasm contracts
* To implement an ewasm backend in the Solidity compiler
* To provide a library and instructions for writing contracts in Rust
* To provide a library and instructions for writing contracts in C

## Glossary

* *Ewasm contract*: a contract adhering to the ewasm specification
* *Ethereum environment interface (EEI)*: a set of methods available to ewasm contracts
* *metering*: the act of measuring execution cost in a deterministic way
* *metering injector*: a transformation tool inserting metering code to an ewasm contract
* *EVM transcompiler*: an EVM bytecode (the current Ethereum VM) to ewasm transcompiler. [See this chapter](./evm_transcompiler.md).

## Resources

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

## Projects

* [ewasm-tests](https://github.com/ewasm/ewasm-tests)
* [wasm-metering](https://github.com/ewasm/wasm-metering)
* [ewasm-kernel](https://github.com/ewasm/ewasm-kernel)
* [evm2wasm](https://github.com/ewasm/evm2wasm)
* [ewasm-libc](https://github.com/ewasm/ewasm-libc)
* [assemblyscript-ewasm-api](https://github.com/ewasm/assemblyscript-ewasm-api)
* [ewasm-rust-api](https://github.com/ewasm/ewasm-rust-api)

## Design Process & Contributing
High-level design discussions should continue to be held in this repository, via issues and pull requests. Feel free to file [issues](https://github.com/ethereum/ewasm-design/issues).

The best place to start contributing to the Ewasm project is our [public Gitter channel](https://gitter.im/ewasm/Lobby). The Ewasm team hangs out in this channel and is on hand to answer questions.
