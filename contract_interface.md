# Ewasm Contract Interface (ECI) Specification (*Revision 3*)

The Ewasm Contract Interface (ECI) specifies the structure of a contract module.

### Wire format

Every contract must be stored in the [WebAssembly Binary Encoding](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md) format (in short, WASM bytecode).

### Imports

A contract can only import symbols specified in the [Ethereum Environment Interface or EEI](./eth_interface.md).

In practice, this means that all imports specified by an ewasm module must be from the `ethereum` namespace,
and having a function signature and name directly correspondent to a function specified in the EEI.

As mentioned below, there is a `debug` namespace as well, but that is disallowed in production systems.

#### Debug-mode

Debug-mode is a special VM option, where an additional set of debugging interfaces are available to contracts.  On a live VM, any bytecode trying to import these
symbols should be rejected.

The imports are available under the `debug` namespace:
- `print32(value: i32)` - print value
- `print64(value: i64)` - print value
- `printMem(offset: i32, len: i32)` - print memory segment as printable characters
- `printMemHex(offset: i32, len: i32)` - print memory segment as hex
- `printStorage(pathOffset: i32)` - print storage value as printable characters
- `printStorageHex(pathOffset: i32)` - print storage value as hex

### Exports

A contract must have exactly two exported symbols:
- `memory`: the shared memory space available for the EEI to write into.
- `main`: a function with no parameters and no result value.

### Entry point

The method exported as `main` will be executed by the VM.

On successful execution, the code should return via a normal code path.

If it needs to abort due to a failure, an *unreachable* instruction should be executed.

### Start function

The use of a [start function](https://webassembly.github.io/spec/core/syntax/modules.html#start-function) is disallowed.

The reason for this is that an ewasm VM would need to have access to the memory space of a contract and that must be acquired prior to executing it.
In the [WebAssembly Javascript API](https://webassembly.org/docs/js/) however the start function is executed right during instantiation, which
leaves no time for the client to acquire the memory area.

*Note:* This decision was made on WebAssembly version 0xb (pre version 1) and should be revisited.
