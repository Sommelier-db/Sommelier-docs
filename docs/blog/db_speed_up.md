#### Author: Takaki Hamada

## Speed Up SQL Processing in Sommelier Drive Server.

In `Sommelier Drive Server`, the data processed by `Drive` are including huge encrypted contents, such as PKSE cipher text.
Therefore, when executing multiple `INSERT` query, the Sommelier DB processing became very slow and `Drive` command processing took more than a minute.
To resolve this problem, I customized the issuing of transactions in SQL query.

## Management Transaction in Sommelier Drive Server.

In Sommelier DB (and SQLite3 it is copied from) `INSERT` query, transaction management execute internaly per query.
So, when adding multiple rows into table at once, programer should manage the DB transaction manually and execute several `INSERT` query in one transaction to speed up DB processing.
At the implementation of `Sommelier Drive Server`, speed up processing by managing the `TRANSACTION` to be `COMMIT` and `BEGIN` again every 10 `INSERT` queries issued.
An example of controlling `TRANSACTION` in C API is implemented at 
[`sommelier-drive-server/src/dbms.c`](https://github.com/Sommelier-db/sommelier-drive-server/blob/develop/src/dbms.c).
