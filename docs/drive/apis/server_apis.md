A server of the Sommelier Drive provides the following REST APIs.

## REST APIs
- GET /user

Returns a record in the User table where a value of the UserID column is `userId`.

Request:
```
{
    "userId": Int
}
```

Response:
```
{
    "userId": Int,
    "dataPK": String,
    "keywordPK": String,
    "nonce": Int
}
```

- POST /user

Takes as input public keys and file path encryptions of the PKE and PKSE schemes, inserts them in the User table and Path table, and returns a new `userId`.

Request:
```
{
    "dataPK": String,
    "keywordPK": String,
    "dataCT": String,
    "keywordCT": String,
}
```

Response: `Int`

- GET /file-path

Returns a record in the Path table where a value of the PathID column is `pathId`.

Request:
```
{
    "pathId": Int
}
```

Response:
```
{
    "pathId": Int,
    "userId": Int,
    "permissionHash": String,
    "dataCT": String,
    "keywordCT": String
}
```

- GET /file-path/children

Returns a list of records in the Path table where a value of the PermissionHash column is equal to the `permissionHash`.

Request:
```
{
    "permissionHash": String
}
```

Response:
```
[
    {
        "pathId": Int,
        "userId": Int,
        "permissionHash": String,
        "dataCT": String,
        "keywordCT": String
    },
    ...
]
```

- GET /file-path/search

Takes as input a `userId` and a trapdoor `td` and returns records in the Path table where a value of the UserID column is `userId` and a ciphertext `ct` of the KeywordCT column satisfies `test_cipher(ct,td)==1`.

Request:

```
{
    "userId": Int,
    "trapdoor": String
}
```

Response:
```
[
    {
        "pathId": Int,
        "userId": Int,
        "permissionHash": String,
        "dataCT": String,
        "keywordCT": String
    },
    ...
]
```


- POST /file-path

Takes as input `readUserId`, `premissionHash`, `dataCT`, and `keywordCT`, inserts them into the Path table, and returns a new `pathId`.

Request:
```
{
    "readUserId": Int,
    "premissionHash": String,
    "dataCT": String,
    "keywordCT": String,
}
```

Response: `Int`

- GET /shared-key

Returns a record in the Shared Key table where a value of the PathID column is `pathId`.

Request:
```
{
    "pathId": Int
}
```

Response:
```
{
    "sharedKeyId": Int,
    "pathId": Int,
    "ct": String
}
```


- POST /shared-key

Takes as input `pathId` and `ct`, inserts them into the Shared key table, and returns a new `sharedKeyId`.

Request:
```
{
    "pathId": Int,
    "ct": String
}
```

Response: `Int`


- GET /contents

Returns a record in the Contents table where a value of the SharedKeyHash column is equal to the `sharedKeyHash`.

Request:
```
{
    "sharedKeyHash": String
}
```

Response:
```
{
    "contentsId": Int,
    "sharedKeyHash": String,
    //”authorizationPK”: String,
    "nonce": Int
    "ct": String
}
```

- POST /contents

Takes as input `sharedKeyHash` and `ct`, inserts them into the Contents table, and returns a new `contentsId`.

Request:
```
{
    "sharedKeyHash": String,
    "ct": String
}

```

Response: `Int`

- PUT /contents

Takes as input `sharedKeyHash` and `ct`, modifies a value of the ContentsCT column in the record where a value of the SharedKeyHash column is equal to the `sharedKeyHash`.

Request:
```
{
    "sharedKeyHash": String,
    "ct": String
}

```

Response: `Int`
