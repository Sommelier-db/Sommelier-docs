A remote server in Sommelier Drive stores file information in Sommelier DB. That is, it creates DB tables as below.

## User Table
**User table** stores the id of a user and the user's PKE and PKSE public keys. They are denoted by `userId`, `dataPK`, and `keywordPK`, respectively, in the following description. 

| UserID | DataPK | KeywordPK |
| :----: | :----: | :----:    |
| Int    | Text   | Text      |

## Path Table
**Path table** stores information about the file path. Each record in this table corresponds to a pair of the `userId` and the file path, `filePath`. Its columns describe the property of the file or directory located in the `filePath` as below.

| PathID | UserID | PermissionHash | DataCT | KeywordCT |
| :----: | :----: | :----:         | :----: | :----:    |
| Int    | Int    | Text           | Text   | Text      |

* UserID: A `userId` of the user who has read permission to the file or directory located in the `filePath`.
* PermissionHash: A `permissionHash` derived from the `userId` and `Parent(filePath)`, which represents a parent directory path of the `filePath`.  
* DataCT: A ciphertext of the `filePath` encrypted with the `dataPK` corresponding to the `userId`.
* KeywordCT: A ciphertext of the `filePath` encrypted with the `keywordPK` corresponding to the `userId`.

## Shared Key Table
**Shared Key table** stores a `pathId` and a shared key encryption, `sharedKeyCT`. The `sharedKeyCT` is required to recover the file contents located in the `filePath` of the `pathId`.

| SharedKeyID | PathID | SharedKeyCT |
| :----:      | :----: | :----:      |
| Int         | Int    | Text        |

## Contents Table
**Contents table** stores a file contents encryption, `contentsCT`. Each record corresponds to a `sharedKeyHash`, which is a hash value of the shared key. Therefore, users who hold the same shared key can access the same record.

| ContentsID | SharedKeyHash | ContentsCT |
| :----:     | :----:        | :----:     |
| Int        | Text          | Text       |
