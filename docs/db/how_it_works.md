## how to add original function like `test_cipher`

In the SQLite3 implementations, it converts input SQL query into bytecode.
And, SQLite3 executes compiled byteode on VDBE (Vertual DataBase Engine), that is vertual machine manipulates table row data on B-Tree.
By this vertual machine, the function calling query is processed as `Function Opecode`.

There are a function `sqlite3RegisterBuiltinFunction` and a macro `TEST_FUNC`, which are used to define the `Function Opecode` (e.g. length, max, min...).
To realize PKSE searching function, we defined the operation of `test_cipher` and called `TEST_FUNC` in `sqlite3RegisterBuiltinFunction`.

Our `test_cipher` implementation is [here](https://github.com/Sommelier-db/sommelier-db/blob/99def25b95f876a9efd82c85e1a7ac57007c6a5f/src/func.c#L2329).
