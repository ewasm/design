## Why do we want ewasm?

* Fast & Efficient: To truly distinguish Ethereum as the World Computer we need to have a very performant VM. The current architecture of the VM is one of the greatest blockers to raw performance. WebAssembly aims to execute at near native speed by taking advantage of common hardware capabilities available on a wide range of platforms. This will open the door to a wide array of uses that require performance/throughput. 
* Security: With the add performance gains from ewasm we will be able to implement parts of Ethereum such as the precompiled contract in the VM itself which will minimize our trusted computing base.
Standardized Instruction Set: WebAssembly is currently being designed as an open standard by a W3C Community Group and is actively being developed by engineers from Mozilla, Google, Microsoft, and Apple.
* Toolchain Compatibility: A LLVM front-end for WASM is part of the MVP. This will Allow developers to write contracts and reuse applications written in common languages such as C/C++, go and rust.
* Portability: WASM is targeted to be deployed in all the major web browsers which will result in it being one of the most widely deployed VM architecture. Contracts compiled to ewasm will share compatibility with any standard WASM environment. Which will make running a program either directly on Ethereum, on a cloud hosting environment, or on one's local machine - a frictionless process.
* Optional And Flexible Metering: Metering the VM adds overhead but is essential for running untrusted code. If code is trusted then metering maybe optional. Ewasm defines metering as an optional layer to accommodate for these use cases.
* Furthermore some of Wasm's top [design goals](https://github.com/WebAssembly/design/blob/master/HighLevelGoals.md) are largely applicable to Ethereum  

> Define a portable, size- and load-time-efficient binary format to serve as a compilation target which can be compiled to execute at native speed by taking advantage of common hardware capabilities available on a wide range of platforms, including mobile and IoT.


### Details
#### Rationale For Registered Based ISA.

* Register-based virtual machines are more like actual hardware.
* Easier to JIT
* Although most early computers used stack or accumulator-style architectures, virtually every new architecture designed after 1980 uses a load-store register architecture. The major reasons for the emergence of general-purpose register (GPR) computers are twofold. First, registers—like other forms of storage internal to the processor—are faster than memory. Second, registers are more efficient for a compiler to use than other forms of internal storage. For example, on a register computer the expression (A * B) – (B * C) – (A * D) may be evaluated by doing the multiplications in any order, which may be more efficient because of the location of the operands or because of pipelining concerns. Nevertheless, on a stack computer the hardware must evaluate the expression in only one order, since operands are hidden on the stack, and it may have to load an operand multiple times. More importantly, registers can be used to hold variables. When variables are allocated to registers, the memory traffic reduces, the program speeds up (since registers are faster than memory), and the code density improves (since a register can be named with fewer bits than can a memory location). [Reference](http://www.cpp.edu/~kding/materials/Computer%20Architecture%20A%20Quantitative%20Approach%20(5th%20edition).pdf)

* (Java is stack based.) "Java byte-codes have additional disadvantages. Directly mapping byte-codes onto the underlying architecture is much more difficult than generating machine instructions from an abstract syntax-tree. Code generators that are based on a high-level representation do not have to deal with unfavorable peculiarities of Java byte-codes but can tailor their output towards advanced and specific processor features, such as special purpose instructions, size of register sets, and cache architectures. This is especially true for today's most common RISC processors which are less suited for byte-code's heavily used stack operations." [Reference](ftp://ftp.cis.upenn.edu/pub/cis700/public_html/papers/Kistler96.pdf)
* [The design of the Inferno virtual machine](http://herpolhode.com/rob/hotchips.html)
* [Virtual Machine Showdown: Stack Versus Registers](http://static.usenix.org/events/vee05/full_papers/p153-yunhe.pdf)

## Futher Reading
* [wams's design docs](https://github.com/WebAssembly/design)
* [chrome's binary encoding](https://docs.google.com/document/d/1-G11CnMA0My20KI9D7dBR6ZCPOBCRD0oCH6SHCPFGx0/edit?pref=2&pli=1)
* A Tree-Based Alternative to Java Byte-Code - ftp://ftp.cis.upenn.edu/pub/cis700/public_html/papers/Kistler96.pdf
* [JavaTrees](http://central.kaserver5.org/Kasoft/Typeset/JavaTree/Pt06.html#Head363)
* Adaptive Compression of Syntax Trees and Iterative Dynamic Code Optimization: Two Basic Technologies for Mobile-Object Systems -ftp://ftp.cis.upenn.edu/pub/cis700/public_html/papers/Franz97b.pdf
* [Computer Architecture A Quantitative Approach (5th edition)](http://www.cpp.edu/~kding/materials/Computer%20Architecture%20A%20Quantitative%20Approach%20(5th%20edition).pdf)
