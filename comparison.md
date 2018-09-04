# WASM
### Good
* limited well defined non-determinism
* performant (near native speed)
* portable
* will be widely deployed
* AST bytecode makes it easy to decouple metering from the VM

### Bad
* not stable yet

# LLVM IR
### Good
* very tested
* large community
* was used by google's PNaCl
* widely deployed

### Bad
* not intrinsically portable
* not stable
* large surface (ISA) that VM implementors would have to deal with

Response from Derek Schuff (one of the engineers for PNaCl) from google on WASM vs LLVM

>I'm guessing you are unfamiliar with PNaCl. This is more or less the approach taken by PNaCl; i.e. use LLVM as the starting point for a wire format. It turns out that LLVM IR/bitcode by itself is neither portable nor stable enough to be used for this purpose, and it is designed for compiler optimizations, it has a huge surface area, much more than is needed for this purpose. PNaCl solves these problems by defining a portable target triple (an architecture called "le32" used instead of e.g. i386 or arm), a subset of LLVM IR, and a stable frozen wire format based on LLVM's bitcode. So this approach (while not as simple as "use LLVM-IR directly") does work. However LLVM's IR and bitcode formats were designed (respectively) for use as a compiler IR and for temporary file serialization for link-time optimization. They were not designed for the goals we have, in particular a small compressed distribution format and fast decoding. We think we can do much better for wasm, with the experience we've gained from PNaCl

LLVM IR is meant to make compiler optimizations easy to implement, and to represent the constructs and semantics required by C, C++, and other languages on a large variety of operating systems and architectures. This means that by default the IR is not portable (the same program has different representations for different architectures) or stable (it changes over time as optimization and language requirements change). It has representations for a huge variety of information that is useful for implementing mid-level compiler optimizations but is not useful for code generation (but which represents a large surface area for codegen implementers to deal with). It also has undefined behavior (largely similar to that of C and C++) which makes some classes of optimization feasible or more powerful, but which can lead to unpredictable behavior at runtime. LLVM's binary format (bitcode) was designed for temporary on-disk serialization of the IR for link-time optimization, and not for stability or compressibility (although it does have some features for both of those).

None of these problems are insurmountable. For example PNaCl defines a small portable subset of the IR with reduced undefined behavior, and a stable version of the bitcode encoding. It also employs several techniques to improve startup performance. However, each customization, workaround, and special solution means less benefit from the common infrastructure



# CLI / ECMA-335 
### Good
* Defines a Kernel runtime and CIL subset that should be all we need. (We could use just the CIL intermediate language subset, but that could cut us off from the larger ecosystem.)
* Looks like the kernel can be implemented to be safe and deterministic. I didn't try to analyze much, but safety and security were design goals.
* Stable international standards since 2000.
* Stable .NET and open-source Mono implementations.
* Mature tools and language support, large user base, open source (even from Microsoft).   

### Bad   
* Missing standard SIMD support. (Both Mono and .NET are putting it in, so it should be standard eventually. )
* Missing some of the nicer features of Wasm -- stack machine, not so compact, not AST-based, etc.
* Likely a bigger runtime footprint than Wasm.
* Microsoft.
* Did I mention Microsoft?

# RISCV
### Isn't RISCV a purely hardware spec?

# JVM
### Nonstarter.  Oracle ownership and intellectual property issues.

# EVM1
### Good:
* We own it.
* It can evolve to where we need to go.
* Coordination with other projects is optional.
* We have the engineering talent we need.
* We can incorporate wasm and other tech as appropriate.

### Bad:
* We own it.
* We will need to maintain and build our own community of developers and users. (This might be a pro.)
* EVM design has some clunkiness that gets in the way of performance and clean evolution.

