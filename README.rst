.. image:: https://travis-ci.org/WebAssembly/sexpr-wasm-prototype.svg?branch=master
    :target: https://travis-ci.org/WebAssembly/sexpr-wasm-prototype
    :alt: Build Status

sexpr-wasm
==========

Translates from WebAssembly `s-expressions
<https://github.com/WebAssembly/spec>`_ to `v8-native-prototype
<https://github.com/WebAssembly/v8-native-prototype>`_ bytecode.

Cloning
-------

Clone as normal, but don't forget to update/init submodules as well::

  $ git clone https://github.com/WebAssembly/sexpr-wasm-prototype
  $ git submodule update --init

This will fetch the v8-native-prototype and spec repos, which are needed for
some tests.

Building
--------

Building just sexpr-wasm::

  $ make
  mkdir out
  cc -Wall -Werror -g -c -o out/sexpr-wasm.o src/sexpr-wasm.c
  cc -Wall -Werror -g -c -o out/wasm-parse.o src/wasm-parse.c
  cc -Wall -Werror -g -c -o out/wasm-gen.o src/wasm-gen.c
  cc -o out/sexpr-wasm out/sexpr-wasm.o out/wasm-parse.o out/wasm-gen.o

If you make changes to src/wasm-keywords.gperf, you'll need to install gperf as
well. On Debian-based systems::

  $ sudo apt-get install gperf
  ...
  $ touch src/wasm-keywords.gperf
  $ make
  gperf --compare-strncmp --readonly-tables --struct-type src/wasm-keywords.gperf --output-file src/wasm-keywords.h
  ...

Building v8-native-prototype
----------------------------

The v8-native-prototype lets you run the generated bytecode. Some of the tests
rely on this repo. To build it::

  $ scripts/build-d8.sh
  ...

When it is finished, there will be a d8 executable in the
``third_party/v8-native-prototype/v8/v8/out/Release`` directory.

You can also download a prebuilt version (the same one used to test on Travis)
by running the download-d8.sh script::

  $ scripts/download-d8.sh
  ...

This downloads the d8 executable into the ``out`` directory. The test runner
will look here if there is no d8 executable in the
``third_party/v8-native-prototype/v8/v8/out/Release`` directory.

Running
-------

First write some WebAssembly s-expressions::

  $ cat > test.wasm << HERE
  (module
    (export "test" 0)
    (func (result i32)
      (i32.add (i32.const 1) (i32.const 2))))
  HERE

Then run sexpr-wasm to build v8-native-prototype bytecode::

  $ out/sexpr-wasm test.wasm -o test.bin

This can be loaded into d8 using JavaScript like this::

  $ third_party/v8-native-prototype/v8/v8/out/Release/d8
  V8 version 4.7.0 (candidate)
  d8> buffer = readbuffer('test.bin');
  [object ArrayBuffer]
  d8> module = WASM.instantiateModule(buffer, {});
  {memory: [object ArrayBuffer], test: function test() { [native code] }}
  d8> module.test()
  3

If you just want to run a quick test, you can use the run-d8.py script instead::

  $ test/run-d8.py test.wasm
  test() = 3

To run spec-style tests (with assert_eq, invoke, etc.) use the `--spec` flag::

  $ cat > test2.wasm << HERE
  (module
    (export "neg" 0)
    (func (param i32) (result i32)
      (i32.sub (i32.const 0) (get_local 0))))
  (assert_eq (invoke "neg" (i32.const 100)) (i32.const -100))
  HERE
  $ test/run-d8.py --spec test2.wasm
  instantiating module
  $assert_eq_0 OK
  1/1 tests passed.

Tests
-----

To run tests::

  $ make test
  [+251|-0|%100] (0.55s)

In this case, there were 251 passed tests and no failed tests, which took .55
seconds to run.

You can also run the Python test runner script directly::

  $ test/run-tests.py
  [+251|-0|%100] (0.40s)

  $ test/run-tests.py -v
  + bad-string-escape.txt (0.002s)
  + bad-string-hex-escape.txt (0.004s)
  + bad-string-eof.txt (0.005s)
  + bad-toplevel.txt (0.003s)
  ...

To run a subset of the tests, use a glob-like syntax::

  $ test/run-tests.py const -v
  + dump/const.txt (0.003s)
  + expr/bad-const-i32-garbage.txt (0.002s)
  + expr/bad-const-f32-trailing.txt (0.004s)
  + expr/bad-const-i32-overflow.txt (0.003s)
  + expr/bad-const-i32-trailing.txt (0.002s)
  + expr/bad-const-i32-just-negative-sign.txt (0.003s)
  + expr/const.txt (0.003s)
  + expr/bad-const-i32-underflow.txt (0.004s)
  + expr/bad-const-i64-overflow.txt (0.005s)
  [+9|-0|%100] (0.02s)

  $ test/run-tests.py expr*const*i32 -v
  + expr/bad-const-i32-garbage.txt (0.004s)
  + expr/bad-const-i32-overflow.txt (0.002s)
  + expr/bad-const-i32-trailing.txt (0.002s)
  + expr/bad-const-i32-just-negative-sign.txt (0.002s)
  + expr/bad-const-i32-underflow.txt (0.002s)
  [+5|-0|%100] (0.01s)

When tests are broken, they will give you the expected stdout/stderr as a diff::

  $ <introduce bug in wasm-gen.c>
  $ test/run-tests.py store
  - d8/store.txt
    STDOUT MISMATCH:
    --- expected
    +++ actual
    @@ -1,6 +1,6 @@
    -i32_store8() = -16909061
    -i32_store16() = -859059511
    -i32_store() = -123456
    +i32_store8() = 1050144
    +i32_store16() = 1050144
    +i32_store() = 1
     i64_store() = 1
     f32_store() = 1069547520
     f64_store() = -1064352256

  **** FAILED ******************************************************************
  - d8/store.txt
  [+18|-1|%100] (0.06s)

Writing New Tests
-----------------

Tests must be placed in the test/ directory, and must have the extension
`.txt`. The directory structure is mostly for convenience, so for example you
can type `test/run-tests.py d8` to run all the tests that execute in d8.
There's otherwise no logic attached to a test being in a given directory.

That being said, try to make the test names self explanatory, and try to test
only one thing. Also make sure that tests that are expected to fail start with
`bad-`.

The test format is straightforward::

  # KEY1: VALUE1A VALUE1B...
  # KEY2: VALUE2A VALUE2B...
  (input (to)
    (the executable))
  # STDOUT:
  expected stdout
  # STDERR:
  expected stderr

The test runner will copy the input to a temporary file and pass it as an
argument to the executable (which by default is out/sexpr-wasm).

The currently supported list of keys:

- EXE: the executable to run, defaults to out/sexpr-wasm
- FLAGS: additional flags to pass to the executable
- ERROR: the expected return value from the executable, defaults to 0
- SLOW: if defined, this test is marked as being slow, and is skipped unless
  you pass --slow to run-tests.py

When you first write a test, it's easiest if you omit the expected stdout and
stderr. You can have the test harness fill it in for you automatically. First
let's write our test::

  $ cat > test/my-awesome-test.txt << HERE
  # EXE: test/run-d8.py
  # FLAGS: --spec
  (module
    (export "add2" 0)
    (func (param i32) (result i32)
      (i32.add (get_local 0) (i32.const 2))))
  (assert_eq (invoke "add2" (i32.const 4)) (i32.const 6))
  (assert_eq (invoke "add2" (i32.const -2)) (i32.const 0))
  HERE

If we run it, it will fail::

  $ test/run-tests.py awesome
  - my-awesome-test.txt
    STDOUT MISMATCH:
    --- expected
    +++ actual
    @@ -0,0 +1,4 @@
    +instantiating module
    +$assert_eq_0 OK
    +$assert_eq_1 OK
    +2/2 tests passed.

  **** FAILED ******************************************************************
  - my-awesome-test.txt
  [+0|-1|%100] (0.05s)

We can rebase it automatically with the `-r` flag. Running the test again shows
that the expected stdout has been added::

  $ test/run-tests.py awesome -r
  [+1|-0|%100] (0.05s)
  $ test/run-tests.py awesome
  [+1|-0|%100] (0.05s)
  $ tail -n 5 test/my-awesome-test.txt
  # STDOUT:
  instantiating module
  $assert_eq_0 OK
  $assert_eq_1 OK
  2/2 tests passed.
