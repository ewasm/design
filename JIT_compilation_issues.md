### The background

* two ways to execute wasm code: interpreted, or compiled (aka native execution).
  - note that "two ways" is an over-simplification. There's a spectrum in between, where you have slow interpreters at one end, interpeter speedup techniques along the middle, and fully compiled JIT (Just-in-Time) or AOT (Ahead-of-Time) execution at the opposite end.
  - because of this spectrum between interpretation and compilation, some people ask why bother trying to do compiled execution since there is "literally no difference" between interpreted execution and compiled execution. Others point out that this is technically true, there is no difference "except for two orders of magnitude in execution speed."
 
 * an example benchmark: the ECPAIRING precompile (i.e. the zk-snark precompile). We took the Rust implementation of ECPAIRING (which is used in parity, and in ethereumJS by compiling the Rust code to asm.js), and compiled it to wasm. Then we deployed it to our ewasm prototype testnet client, which uses Binaryen (a wasm interpreter) as the wasm engine. With interpreted wasm execution, three CALLs to the ECPAIRING contract took 21 seconds (we can't tell you exactly how much gas it will cost because we haven't yet finished the metering injection "sentinel contract"). Then we executed the same thing in node.js (which uses v8 as a wasm engine), and it took ~100 milliseconds. (somebody out there might wanna try this same benchmark by deploying it on Kovan and running it in Parity, and see how much gas it costs and how long it takes to run in Parity's wasm interpreter).

### The problem with interpreters: gas costs

* In theory, it is possible to just use wasm interpreters as the baseline (or maybe "reference") wasm engine. But the issue is, how will we calibrate ewasm gas costs? If gas costs are calibrated to interpreter execution speeds, then it would be cost prohibitive (in terms of the block gas limit) to run a contract such as the ECPAIRING contract. This would mean that there would still be demand from users to add new precompiles to the EVM/ewasm protocol, (recall that the advantage and motivation of "precompiled contracts" (aka "builtins"), is that they have custom gas costs calibrated to native execution speed).

### The problem with JITs: compiler bombs

* the easiest way to run wasm code at native speeds is to just plug-in to already existing JIT wasm engines: the browsers (Chrome/node.js/v8 and Firefox/spiderMonkey), or non-browsers (WAVM - WebAssembly Virtual Machine - which is based on llvm, i.e. translating wasm bytecode to llvm bitcode and then to machine code). Each client (geth, parity, trinity/pyethereum, ethereumJS) could choose to use whatever JIT engine they want.

* adopting any of these JIT engines would be easy -- just pull one off-the-shelf and it will work out of the box. But they're super complex machines, with massive codebases, so unless you are an expert in how JIT compilers work, they're essentially black boxes.

* the problem is, what if wasm JIT engines are vulnerable to DoS attacks? Running most contracts is fine because the JIT compilation happens very fast (e.g. a few milliseconds), then execution happens fast (say 100 milliseconds). But some contracts could be exploits which take the JIT engine a very long time to compile: "compiler bombs" or "JIT bombs".

* what we couldn't answer before: how do the standard wasm JIT engines work and are they vulnerable to JIT bombs? If they do only one linear pass over the wasm bytecode, then compilation time should be linearly proportional to code size, and JIT bomb attacks would be a non-issue. (or in fancier words, "does wasm JIT compilation have a linear-time upper bound?"). But if standard wasm JIT engines have compilation times that are quadratic (or worse) for certain inputs, then they could be vulnerable to JIT bombs.

* what we've just learned: the standard wasm JIT engines are not linear-time-bounded. We learned this by fuzz testing v8 (and WAVM) to find slow inputs. We found several bombs, which (for example) are 20kb pieces of wasm code that take two seconds to compile in v8. You can see them here [INSERT_LINKS_HERE]

* different wasm engines are vulnerable to different JIT bombs. Even different versions of the same engine, or the same engine run with different option flags, are affected differently by different bombs. Some bombs work across multiple versions (and maybe across multiple engines). We are still exploring and analyzing the features of bombs and how they exploit JIT engines; the studies so far are very preliminary (and we're not JIT compiler experts, or at least some of us aren't).

### Potential solutions

* Restated, the problem is that when we pull off-shelf-engine wasm engines and use them to JIT execute wasm contracts, the execution process (execution stage?) is metered. But the JIT compilation stage is not metered, and there does not appear to be any easy way to add metering to the compilation stage.

* One solution idea is to do metered AOT instead of (not metered) JIT. To imagine a system where contracts are AOT compiled at deployment time, picture every ethereum client maintaining a cache directory of binaries: 0x666cryptokitties.exe, 0xdeadMultiSigContract.exe, and so on.

* Some people's opinion is that a system based on metered AOT would be a big PITA. It would require implementing a new wasm AOT compiler, or adapting an existing wasm engine, adding metering, and then requiring clients to maintain a directory of compiled binaries. From the point of view of a compiler expert, this may not sound like such a big deal. But from the point of view of an average Ethereum client developer, it sounds like a lot of development effort.

* one way to explain to an average Ethereum client developer how metered AOT might work, is to say: take something like WAVM, and adapt it to do AOT compilation rather than JIT (we'll just call this WAVM). Then take WAVM and compile it to wasm (call it WAVM.wasm), and inject metering into that wasm. Then the Ethereum client will run this "WAVM-AOT-metered-compiler" whenever a user deploys a new wasm contract.

* a twist on this idea is to create a wasm "JIT bomb sniffer". Take this WAVM.wasm, or take say v8.wasm (i.e. v8 compiled down to wasm) and inject metering into it. Call them WAVM-sniffer.wasm and v8-sniffer.wasm. Then when a user sends a contract deployment tx, pass it through the bomb sniffer and check if the sniffer's gas usage exceeds a threshold. If the bomb sniffer's gas threshold is exceeded, then deployment fails. This would hopefully ensure that any deployed contracts do not contain JIT bombs, and could be safely executed using off-the-shelf wasm engines.

* the downside of the bomb sniffer idea is that the sniffer would only be protection for particular versions of particular wasm engines. If a client upgrades the wasm engine to a newer version, there might be bombs that have already been deployed which could be used to DoS attack any clients using the new wasm engine. Also, to ensure that deployed contracts could be safely JITted by a variety of wasm engines, multiple sniffers would be needed, with each sniffer tailored to a particular wasm engine. Another concern is that there might be some way to "mask the bomb smell" and deploy JIT bombs by somehow sneaking them past the sniffer. Also, Ethereum client developers wouldn't be able to safely discuss wasm engine DoS vulnerabilities inside or around airports (or at least, they would have to whisper if they do).

### Next steps

* to experiment with the bomb sniffer idea, we plan to compile WAVM (and/or v8) to wasm, to see how big the wasm binary is and how long it takes to execute a bomb sniffer in a wasm interpreter. And we will also check whether WAVM.wasm (and/or v8.wasm) are JIT bombs when executed in themselves (i.e. executing WAVM.wasm/v8.wasm in WAVM.exe/v8.exe).
