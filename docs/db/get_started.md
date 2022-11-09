## Install Sommelier DB

<!-- Sommelier DB のリポジトリへ誘導するようにした -->

The installing scripts of Sommelier DB is constructed by `configure` and `make`.
Detail installing method is described at [README of Sommelier DB](https://github.com/Sommelier-db/sommelier-db).


## Use Sommelier DB CUI client

When you execute your installed binary path, you can access the CLI SQLite Interface.
In this interface, you can use SQLite's DBMS functions and Sommelier DB's original SQL function `test_cipher`.

The `test_cipher` function's specification is described in [APIs](https://sommelier-db.github.io/Sommelier-docs/db/apis/) page.

## Use Sommelier DB C API

Sommelier DB is based on SQLite3 C API source code.
If you want to control Sommelier DB by source code of C, you can access Sommelier DB C API in the same way as SQLite3 C API.
In particular, you should do the following tasks.

1. Install Sommelier DB in the way descirbed above.
2. Add `#include "#sommelier-db.h"` in your header file. (you will be able to use `sqlite*` interface and SQL function `test_cipher`, that is used searching data by PKSE)
3. Add compile options: `-I</path/to/sommelier-db/include> -L</path/to/sommelier-db/lib> -lsommelier-db`.

[Sommelier Drive Server implementation](https://github.com/Sommelier-db/sommelier-drive-server) is one of examples to use "Sommelier DB C API" and to add "C compile options".
