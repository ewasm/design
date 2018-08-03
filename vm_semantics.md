# VM Semantics

We assume an existing Ethereum client (with EVM1) as the basis for extension.

There are only 3 new rules:

1. Whenever a contract is loaded from the state it must be checked for the ewasm signature.

If the signature is present, it must be executed as an ewasm contract, otherwise it must be executed as an EVM1 contract.

2. If there's no native EVM1 support in the client, it can use the *EVM Transcompiler* to translate the code.

3. When deploying an ewasm contract, the bytecode must be verified and annotated by the *Sentinel Contract*.
