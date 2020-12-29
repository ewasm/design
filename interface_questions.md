## ewasm interface methods: synchronous vs asynchronous

One design question that arose while prototyping ewasm in Javascript is the issue of a synchronous versus an asynchronous [Ethereum Environment Interface aka EEI specification](./eth_interface.md). The initial design specified synchronous EEI methods: `storageLoad`, `storageStore`, `callDelegate`, etc. These environment methods are provided to the wasm VM as "host functions" imported by the wasm instance. When running browser-based wasm VMs (e.g. in Chrome or Firefox), these host functions are implemented in Javascript. By contrast, in a wasm VM such as [binaryen](https://github.com/WebAssembly/binaryen), a C++ wasm interpreter, the host functions are implemented in C++. A C++ implementation has full control over environment method execution, and is fully synchronous by default. When some wasm code calls to a host function, the wasm instance will pause and wait until the host function returns, and then continue with wasm execution.

Javascript, however, has a [never blocking](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop#Never_blocking) event loop. As a consequence, when a browser wasm instance calls to a host function, the host function cannot use any callbacks (or Promises, async/await, etc.) because the event loop continues wasm execution without waiting. Thus, _synchronously_ returning a Javascript host function result from a callback to a wasm instance is [not currently supported](https://github.com/WebAssembly/design/issues/720).

Asynchronous ewasm methods [were proposed](https://github.com/ewasm/design/pull/48) to overcome this limitation of JS wasm environments. In the asynchronous version, contract code has callback entry points. This allows the wasm instance to call the host function, terminate, and then restart the instance at the callback entry point.

Here are example ewasm contracts comparing the two versions:

#### synchronous

```
;; address 5d48c1018904a172886829bbbd9c6f4a2d06c47b has a balance of 0xde0b6b3a7640000 (1 ETH)
(module
  ;; synchronous getBalance method
  ;; params are addressOffset, resultOffset
  (import  "ethereum" "getBalance"  (func $getBalance (param i32 i32)))
  (memory 1 )
  ;; address memory location at offset 0
  (data (i32.const 0)  "\5d\48\c1\01\89\04\a1\72\88\68\29\bb\bd\9c\6f\4a\2d\06\c4\7b")
  (export "memory" (memory 0))
  (export "main" (func $main))
  (func $main
    ;; pass 0 as the addressOffset, 100 as the resultOffset
    (call $getBalance (i32.const 0) (i32.const 100))
    ;; getBalance host function result written to memory location 100
    (if (i64.eq (i64.load (i32.const 100)) (i64.const 0xde0b6b3a7640000))
      (return)
    )
    (unreachable) ;; throw if getBalance result not equal to 1 ETH
  )
)
```


#### async proposal, example using getBalance

```
;; address 5d48c1018904a172886829bbbd9c6f4a2d06c47b has a balance of 0xde0b6b3a7640000 (1 ETH)
(module
  ;; asynchronous getBalance method
  ;; params are addressOffset, resultOffset, and callbackIndex
  (import  "ethereum" "getBalance"  (func $balance (param i32 i32 i32)))
  (memory 1)
  (data (i32.const 0)  "\5d\48\c1\01\89\04\a1\72\88\68\29\bb\bd\9c\6f\4a\2d\06\c4\7b")
  (export "memory" (memory 0))
  (export "main" (func $main))
  (export "1" (func  $callback)) ;; callback entry point is an export with name "1"

  (func $main
    ;; pass 0 as the address memory location
    ;; pass 100 as the result memory location
    ;; pass 1 as the callback param
    (call $balance (i32.const 0) (i32.const 100) (i32.const 1))
  )

  (func $callback
    (block
      (if (i64.eq (i64.load (i32.const 100)) (i64.const 0xde0b6b3a7640000))
        (return)
      )
      (unreachable) ;; throw if test fails
    )
  )
)
```

After evaluating the trade-offs between the sync and async ewasm interfaces, we've decided to adopt the synchronous version. First, the synchronous interface better matches the EVM execution model, which synchronously executes contract calls. Secondly, although implementing the synchronous interface in Javascript requires inconvenient workarounds, in other programming languages (C++, Go, Rust, etc.) it is simpler than the async version. Lastly, adapting existing EVM higher-level languages such as Solidity to the async ewasm version would be much more complicated.

### Implementing the Synchronous EEI methods in Javascript

Implementing the synchronous interface in Javascript requires a workaround for returning data from host functions to the wasm instance, and there are several approaches.

One approach is to use a `SharedArrayBuffer` and [Atomics](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics) inside the host function to block execution of the wasm instance. The problem with this approach is that browser vendors are [disabling SharedArrayBuffer](https://www.mozilla.org/en-US/security/advisories/mfsa2018-01/) to mitigate the [Spectre timing attack](https://blog.mozilla.org/security/2018/01/03/mitigations-landing-new-class-timing-attack/). For this approach to be practical, the sharedArrayBuffer would need to be re-enabled in browsers and remain as an [ECMAScript standard feature](https://github.com/tc39/security/issues/3).

A second approach is to preload all environment data that an ewasm contract will access. The problem with this approach is that the current Ethereum transaction protocol permits contracts to dynamically access environment data, so which data a contract will access is not known prior to execution. For a Javascript host function to provide synchronous dynamic data access, each time the contract tries to access some piece of environment data that is not preloaded, a fetch is initiated and the wasm instance would be terminated. Once the fetch returns and the piece of data is loaded, then the wasm instance is restarted and the contract ran from the beginning. For contracts which access lots of unknown environment data, the wasm instance would need to be restarted from the beginning many times. Although this approach is very inefficient under the current transaction protocol, proposed protocol improvements such as [EIP 648](https://github.com/ethereum/EIPs/issues/648) require transactions to specify access ranges. Under such proposals, any potential data that the contract could access is known ahead of time, and could simply be preloaded.

A third approach is to execute wasm code in an interpreter (i.e. a wasm interpreter written in or compiled to Javascript), rather than executing wasm code in a browser's native wasm instance. Interpreting the wasm code gives full control over its execution, but would be slower than a native wasm instance.
