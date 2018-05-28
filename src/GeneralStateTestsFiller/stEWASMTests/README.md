## ewasm Tests

Current [eWasm test cases][1] are based on the main [Ethereum test cases][2]
which are used by all clients. This allows us to run eWasm tests using
cpp-ethereum/testeth.

In the main directory of the ewasm tests repository, we can see different kind
of tests, for example: ABITests, BasicTests, BlockchainTests, GeneralStateTests,
VMTests, etc. For testing eWasm we are using
GeneralStateTests. The [GeneralStateTests][3] contains subdirectories grouping
together tests cases specific for different scenarios.

## Test files

In [GeneralStateTests][3] you can find the test cases specific for
eWasm: [stEWASMTests][4], this directory contains the "filled" json files, those
files are read by testeth, testeth reads the [pre-state][5],
the [transaction][6] and executes it and at the and compares if the postState
hash is the same as the [expected][7]. These "filled" json files are generated
by testeth and are based on the [filler files][8] (Note that the filler files
are inside the [src][15] directory)

## Fillers

Filler files can be written in json or in yaml format, all tests specific for
eWasm are written in yaml.

The yaml file name should be the same as the test name + `Filler`
([example][9]), this is validated by testeth.

The test contains 4 main elements:

### 1. env - environmental parameters

For almost all the ewasm tests the environmental parameters remains the same,
unless we need to test something specific like getting the block gas limit,
block number, etc.

env example:

```yaml
env:
    currentCoinbase: 2adc25665018aa1fe0e6bc666dac8fc2697ff9ba
    currentDifficulty: '0x020000'
    currentGasLimit: '89128960'
    currentNumber: '1'
    currentTimestamp: '1000'
    previousHash: 5e20a0453cecd065ea59c37ac63e079ee08998b6045136a8ce6635c7912ec0b6
```

### 2. pre - state before the transaction (pre-state)

Here we define accounts and the initial values for each one, for almos all test
cases we are using the account `a94f5374fce5edbc8e2a8697c15331677e6ebf0b` this
is the account which starts the initial transaction, its [secret key][10] is used in
the transaction definition. Then we can add more accounts that can be called by
the transaction.

Each account contains he following fields:

- balance
- code (empty for Externally Owned Accounts)
- nonce
- storage

#### code

Example of wast code: 

```lisp
(module
    (import "ethereum" "getCallDataSize" (func  $getCallDataSize (result i32)))
    (import "ethereum" "storageStore" (func $storageStore (param i32 i32)))
    (memory 1)
    (export "memory" (memory 0))
    (export "main" (func $main))
    (func $main
      (i32.store (i32.const 0) (call $getCallDataSize))
      (call $storageStore (i32.const 100) (i32.const 0))
    )
  )
```

`testeth` detects source starting with `(module` as wasm and will automatically compile it.

```lisp
(module
```

then you can import all the needed [EEI methods][11]

```lisp
(import "ethereum" "getCallDataSize" (func $getCallDataSize (result i32)))
(import "ethereum" "storageStore" (func $storage (param i32 i32)))
```

EEI methods can have parameters and some of them can return values, in the example above, `storageStore` has two `i32` parameters, and `getCallDataSize` returns a `i32` value.

Then you have to define the memory to be used (all tests right now uses 1):

```lisp
(memory 1)
```

then you have to export the memory and the function main:

```lisp
(export "memory" (memory 0))
(export "main" (func $main))
```

finally define the main function:

```lisp
(func $main
  (i32.store (i32.const 0) (call $getCallDataSize))
  (call $storageStore (i32.const 100) (i32.const 0))
)
```

#### debugging

Hera also provides options to [debug wast code][14]. In order to use this debug
option we need to build hera with the flag `-DHERA_DEBUGGING=ON`.

The most common used method for debugging is `printMemHex`, this methods allows
us to print parts of the WASM memory, this method receives two parameters:
`offset` and `length`, this print the number of bytes specified in `length`
starting at `offset`.

After you have debugged your test case and you are sure it is working as
expected you should remove the debug imports and statements, this is because
when Hera is built without the `-DHERA_DEBUGGING=ON` flag all tests which
includes debug statements will fail.

- How to import a debug method:

```lisp
(import "debug" "printMemHex" (func $printMemHex (param i32 i32)))
```

- How to call a debug method:

```lisp
(call $printMemHex (i32.const 0) (i32.const 32))
```

### 3. expect - expected result

Example:

```yaml
expect:
  - indexes:
    data: !!int -1
    gas: !!int -1
    value: !!int -1
  network:
    - ALL
  result:
    # InitialBalance - contract1.call - valueTransfer - txCost - contract2.calldatasize - contract2.sstore
    # 100000000000   - 700            - 9000          - 21000  - 2                      - 20000            = 99999949298
    a94f5374fce5edbc8e2a8697c15331677e6ebf0b:
      balance: '99999949298'
    deadbeef00000000000000000000000000000000:
      storage: {
      }
    deadbeef00000000000000000000000000000001:
      storage: {
        0: "0x4"
      }
      balance: "100000000001"
```

For the expect section values of indexes and network remain the same in all test
cases, the most important field in this section is `result`.

In `result` field we specify the accounts as we do in the `pre` section, but now
we define the values that we expect as result of the transaction and the
execution of the wast code.

```yaml
deadbeef00000000000000000000000000000001:
   storage: {
     0: "0x4"
   }
   balance: "100000000001"
```
### 4. transaction

Transaction contains the following fields:

- data
- gasLimit
- gasPrice - usually set to 1 to make it easy to calculate gas
- nonce
- secretKey - using
  `45a915e4d060149eb4365960e6a7a45f334393093061116b197e3240065ff2d8` which is
  the private key for account
  `a94f5374fce5edbc8e2a8697c15331677e6ebf0b`
- to - called account, if the account is an Internally Owned Account it will
  execute the wasm code.
- value - value sent to the account.


example:

```yaml
transaction:
  data:
  - '0x'
  gasLimit:
  - '0x6acfc0'
  gasPrice: '0x01'
  nonce: '0x04'
  secretKey: "45a915e4d060149eb4365960e6a7a45f334393093061116b197e3240065ff2d8"
  to: 'deadbeef00000000000000000000000000000000'
  value:
    - '0'
```

## Filling and executing the tests

Filling the test is the process to take the source Filler files and generate
json files. The final json file contains the result of the execution as a
post state hash.

In order to "fill the tests" we need cpp-ethereum/testeth built with hera.

Testeth also needs `lllc` (LLL Compiler) in order to fill tests cases (even if
we are not using LLL code in our tests), `lllc` compiler is distributed along
with [solidity][13], you may need to compile solidity/lllc and add it to your `$PATH`.

### building cpp-ethereum with hera

For building cpp ethereum you can use the commit specified in the [hera circleci file][12]

For example, current commit used in hera circle ci is
`b387b5713738da12444af8d3a0f6390f2c87b76e`, in order to build cpp-ethereum with
hera we follow the following steps:

- clone cpp-ethereum

```
git clone https://github.com/ethereum/cpp-ethereum.git
```

- go to the specific commit

```
cd cpp-ethereum/
git reset --hard b387b5713738da12444af8d3a0f6390f2c87b76e
```

- update submodules

```
git submodule update --init
```

- update hera to the latest version (or the specific hera version you want to
  test) and update hera submodules.
  
```
cd hera
git pull origin master
git submodule update --init
```

- build cpp-ethereum with hera, create a `build` directory inside
`cpp-ethereum/` and run cmake and make:

```
mkdir build
cd build
cmake -DHERA=ON -DHERA_DEBUGGING=ON .. && make -j4
```

### Filling the tests

When cpp-ethereum is built, it has testeth in `cpp-ethereum/build/test`


You can fill the test using the following command:

```
ETHEREUM_TEST_PATH=/ewasm/tests ./testeth -t GeneralStateTests/stEWASMTests -- --filltests --vm hera --singlenet "Byzantium" --singletest mytest
```

- `ETHEREUM_TEST_PATH` defines an environment variable indicating the location of
our ewasm tests directory this variable is read by testeth in order to execute
and fill our tests

- `-t GeneralStateTests/stEWASMTests` - Indicates the directory where our test
  files are
  
- `--vm hera` - Indicates we want to run the hera VM instead of the EVM
- `--singlenet "Byzantium"` - Indicates we want to run Byzantium tests, ewasm is
  Byzantium only
- `--singletest mytest` - Where `mytest` is the name of the test you want to
  fill and execute, you only have to provide the test name (without the `Filler` ending)


[1]: https://github.com/ewasm/tests
[2]: https://github.com/ethereum/tests
[3]: https://github.com/ewasm/tests/tree/wasm-tests/GeneralStateTests/
[4]: https://github.com/ewasm/tests/tree/wasm-tests/GeneralStateTests/stEWASMTests
[5]: https://github.com/ewasm/tests/blob/wasm-tests/GeneralStateTests/stEWASMTests/call.json#L31-L53
[6]: https://github.com/ewasm/tests/blob/wasm-tests/GeneralStateTests/stEWASMTests/call.json#L54-L68
[7]: https://github.com/ewasm/tests/blob/wasm-tests/GeneralStateTests/stEWASMTests/call.json#L21
[8]: https://github.com/ewasm/tests/tree/wasm-tests/src/GeneralStateTestsFiller/stEWASMTests
[9]: https://github.com/ewasm/tests/blob/wasm-tests/src/GeneralStateTestsFiller/stEWASMTests/callFiller.yml#L1
[10]: https://github.com/ewasm/tests/blob/wasm-tests/src/GeneralStateTestsFiller/stEWASMTests/callFiller.yml#L88
[11]: https://github.com/ewasm/design/blob/master/eth_interface.md
[12]: https://github.com/ewasm/hera/blob/master/circle.yml#L65
[13]: https://github.com/ethereum/solidity
[14]: https://github.com/ewasm/hera#debugging-module
[15]: https://github.com/ewasm/tests/tree/ewasm-readme/src/
