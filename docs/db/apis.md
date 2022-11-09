#### Author: Takaki Hamada

## APIs detail.

Sommelier DB APIs are constructed from,

1. APIs derived from SQLite3.
2. Sommelier DB Specific SQL function: test_cipher

These are described at following sections.

## APIs derived from SQLite3

The SQLite3's API manual is [here](https://www.sqlite.org/c3ref/intro.html).
When you want to process basic SQL queries, you should call the fucntions included this API. such as `sqlite3_exec`.

At next section, it is explaining the APIs used in [Sommelier Drive Server Database Subroutine](https://github.com/Sommelier-db/sommelier-drive-server/blob/develop/src/orm.c).

## Commonly used API of SQLite3: sqlite3_exec

If you execute SQL queries in C source code, you can use [`sqlite3_exec` function](https://www.sqlite.org/c3ref/exec.html).

This function recieves not only char pointer stored a SQL query, but also a callback function, which is processing a SQL result line by line, and a void pointer, which can be accessed from callback function.
So, we can define arbitary processing routine for SQL execution result row.

## Sommelier DB Specific SQL function: test_cipher

When you search specific token with PKSE for the ciphertexts of SQL result, you call `test_cipher` SQL function in the SQL query.

This SQL function recieves a ciphertext by PKSE and a trapdoor words, and it returns the integer 1 (This means searching result is True) or 0 (Result is False).
For example as following SQL code block, you can call this function with SELECT statement, and you can get PKSE search results.

```SQL
-- `Table` is defined as `Table (id INT, name TEXT, cipher TEXT)`

SELECT id, name from Table where test_cipher(cipher, trapdoor) == 1;
```
