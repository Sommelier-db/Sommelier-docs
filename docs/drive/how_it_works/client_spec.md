A client of Sommelier Drive can manage files on the remote server in a similar manner to Unix file systems. While our current implementation provides two types of client CLI, both of them perform each operation in the background as below.

## User registration
1. A client generates new pairs of private and public keys of the PKE and PKSE schemes, `dataPK` and `keywordPK`.
1. The client decides a name of the initial directory `initDir` and encrypts its file path `/'initDir'` with the `dataPK` and `keywordPK`, which results in the PKE and PKSE ciphertexts `dataCT` and `keywordCT`.
1. The client posts the `dataPK`, `keywordPK`, `dataCT`, and `keywordCT` to a remote server.
1. The remote server inserts the provided data into the User table and makes a new `userId`.
1. The remote server also inserts the provided ciphertexts and `userId` into the Path table, where a `permissionHash` is derived from `userId` and a root file path `/` by the remote server.
1. The remote server returns `userId` to the client.
1. The client derives `permissionHash` from the returned `userId` and `/` and requests the records in the Path table where a value of PermissionHash column is equal to `permissionHash`.
1. The remote server returns the requested record in the Path table, which includes the id of the file path `pathId`.
1. The client generates a fresh shared key `sharedKey` and encrypts it with `dataPK`, which results in the shared key encryption `sharedKeyCT`.
1. The client posts the `pathId` and `sharedKeyCT` to the remote server.
1. The remote server inserts the provided `pathId` and `sharedKeyCT` into the Shared key table.
1. The client constructs a bit string called `contentsData` as the following table:
1. The client encrypts the `contentsData` with the `sharedKey`, which results in `contentsCT`.
1. The client computes the hash value `sharedKeyHash` and posts `contentsCT` and `sharedKeyHash` to the remote server.
1. The remote server inserts the provided `contentsCT` and `sharedKeyHash` into the Contents table.

| Field Name | Bit Size | Description |
| :---:| :------: | :-----      |
| `isFile` | 1 | 1 for a file, 0 for a directory.|
| `numReadableUsers` | 64 | A big-endian integer of the number of users with read permission.|
| `readableUserPathIds` | 64 * `numReadableUsers` | A vector of `pathId`s corresponding to this contents.|
| `fileBytes` | variable | A byte string of the contents of the file for a file, an empty byte string for a directory.|

## File/Directory retrieve (touch)
A client with the `userId` will retrieve the contents of the file located in the `filePath` as below.

1. The client derives a `permissionHash` from the `userId` and the parent directory file path `Parent(filePath)` and requests the records in the Path table where a value of PermissionHash column is equal to `permissionHash`.
1. The remote server returns the requested records in the Path table.
1. For each returned record, the client decrypts its `dataCT` and find the `pathId` whose corresponding file path is equal to the `filePath`.
1. The client requests the record in the Shared key table where a value of the PathID column is equal to `pathId`.
1. The remote server returns the requested record in the Shared key table.
1. The client recovers the `sharedKey` from the `sharedKeyCT` in the returned record.
1. The client requests the record in the Contents table where a value of the SharedKeyHash column is equal to `sharedKeyHash` derived from `sharedKey`.
1. The remote server returns the requested record in the Contents table.
1. The client decrypts the `contentsCT` in the returned record with the `sharedKey`, which results in the desired `contentsData`.

## Children file paths retrieve (ls)
A client with the `userId` will retrieve the children file paths under the `filePath` as below.

1. The client derives a `permissionHash` from the `userId` and the `filePath` and requests the records in the Path table where a value of PermissionHash column is equal to `permissionHash`.
1. The remote server returns the requested records in the Path table.
1. For each returned record, the client decrypts its `dataCT`. These recovered file paths are the children file paths under the `filePath`.

## Descendant file paths retrieve (find)
A client with the `userId` will retrieve the descendant file paths under the `filePath` as below.

1. The client generates a trapdoor with the PKSE private key for strings with `filePath` as prefix and requests the records in the Path table where a value of KeywordCT column matches with the trapdoor.
1. The remote server returns the requested records in the Path table.
1. For each returned record, the client decrypts its `dataCT`. These recovered file paths are the descendant file paths under the `filePath`.

## File creation (touch)
A client with the `userId` will locate a new file whose contents bytes are `bytes` in the `filePath`.

1. The client retrieves the `contentsData` located in `Parent(filePath)` in the same way as [the file/directory retrieve process](./#filedirectory-retrieve-touch).
1. The client parse the `contentsData` as `(isFile, numReadableUsers, readableUserPathIds, fileBytes)`.
1. The client generates a fresh shared key `sharedKey` and derives its hash `sharedKeyHash`.
1. For each `pathId` in the `readableUserPathIds`, the client and the remote server performs the following process:
    1. The client requests the records in the Path table where a value of PathID column is equal to the `pathId`.
    1. The remote server returns the requested record in the Path table.
    1. The client retrieves the `usedId` from the returned record and requests the records in the User table where a value of UserID column is equal to the `userId`.
    1. The remote server returns the requested record in the User table.
    1. The client retrieves the `dataPK` and `keywordPK` from the returned record and encrypts the `filePath` with these keys, which results in the `dataCT` and `keywordCT`, respectively.
    1. The client also derives the `permissionHash` from the `userId` and the `Parent(filePath)` and posts the `userId`, `permissionHash`, `dataCT`, and `keywordCT` to the remote server.
    1. The remote server inserts the provided data into the Path table and returns a new `pathId`.
    1. The client encrypts the `sharedKey` with the `dataPK`, which results in the `sharedKeyCT`.
    1. The client posts the `pathId` and `sharedKeyCT` to the remote server.
    1. The remote server inserts the provided data into the Shared key table.
1. The client constructs a new `contentsData` as the following table and encrypts it with the `sharedKey`, which results in the `contentsCT`.
1. The client posts the `sharedKeyHash` and `contentsCT` to the remote server.
1. The remote server inserts the provided data into the Contents table.

| Field Name | Bit Size | Value |
| :---:| :------: | :-----      |
| `isFile` | 1 | 1 |
| `numReadableUsers` | 64 | `numReadableUsers` in Step 2.|
| `readableUserPathIds` | 64 * `numReadableUsers` | `readableUserPathIds` in Step 2.|
| `fileBytes` | variable | `bytes`|

## Directory creation (mkdir)
A client with the `userId` will locate a new directory in the `filePath`.

1. The client retrieves the `contentsData` located in `Parent(filePath)` in the same way as [the file/directory retrieve process](./#filedirectory-retrieve-touch).
1. The client parse the `contentsData` as `(isFile, numReadableUsers, readableUserPathIds, fileBytes)`.
1. The client generates a fresh shared key `sharedKey` and derives its hash `sharedKeyHash`.
1. For each `pathId` in the `readableUserPathIds`, the client and the remote server performs the following process:
    1. The client requests the records in the Path table where a value of PathID column is equal to the `pathId`.
    1. The remote server returns the requested record in the Path table.
    1. The client retrieves the `usedId` from the returned record and requests the records in the User table where a value of UserID column is equal to the `userId`.
    1. The remote server returns the requested record in the User table.
    1. The client retrieves the `dataPK` and `keywordPK` from the returned record and encrypts the `filePath` with these keys, which results in the `dataCT` and `keywordCT`, respectively.
    1. The client also derives the `permissionHash` from the `userId` and the `Parent(filePath)` and posts the `userId`, `permissionHash`, `dataCT`, and `keywordCT` to the remote server.
    1. The remote server inserts the provided data into the Path table and returns a new `pathId`.
    1. The client encrypts the `sharedKey` with the `dataPK`, which results in the `sharedKeyCT`.
    1. The client posts the `pathId` and `sharedKeyCT` to the remote server.
    1. The remote server inserts the provided data into the Shared key table.
1. The client constructs a new `contentsData` as the following table and encrypts it with the `sharedKey`, which results in the `contentsCT`.
1. The client posts the `sharedKeyHash` and `contentsCT` to the remote server.
1. The remote server inserts the provided data into the Contents table.

| Field Name | Bit Size | Value |
| :---:| :------: | :-----      |
| `isFile` | 1 | 0 |
| `numReadableUsers` | 64 | `numReadableUsers` in Step 2.|
| `readableUserPathIds` | 64 * `numReadableUsers` | `readableUserPathIds` in Step 2.|
| `fileBytes` | variable | Empty bytes |

## File Modification
A client with the `userId` will modify a file located in the `filePath` with the bytes `newBytes`.

1. The client retrieves the `contentsData` and `sharedKey` of the file located in `filePath` in the same way as [the file/directory retrieve process](./#filedirectory-retrieve-touch).
1. The client parse the `contentsData` as `(isFile, numReadableUsers, readableUserPathIds, fileBytes)`.
1. The client constructs a new `contentsData` as the following table and encrypts it with the `sharedKey`, which results in the `contentsCT`.
1. The client posts the `sharedKeyHash` and `contentsCT` to the remote server.
1. The remote server inserts the provided data into the Contents table.

| Field Name | Bit Size | Value |
| :---:| :------: | :-----      |
| `isFile` | 1 | 1 |
| `numReadableUsers` | 64 | `numReadableUsers` in Step 2.|
| `readableUserPathIds` | 64 * `numReadableUsers` | `readableUserPathIds` in Step 2.|
| `fileBytes` | variable | `newBytes`|

**Note that the other operations, e.g., file deletion, are not supported in the current implementation**.