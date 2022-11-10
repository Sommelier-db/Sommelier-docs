# Sommelier DB
**Open source DB with public-key searchable encryption: discerning encrypted data without knowing the contents, just like a sommelier!**
<img src="./assets/figs/logo.png" width="50%" style="display: block; margin: auto;">

## Overview of Sommelier DB
Sommelier DB is **an open source DB library combining SQLite and public-key searchable encryption (PKSE)**. In addition to existing SQLite features, it provides functions for PKSE as below.


1. A new SQL function to test a keyword encryption of the PKSE scheme.
2. C and Rust functions to generate a keyword encryption and a trapdoor of the PKSE scheme.


The above functions allow users to have the database (DB) server search for appropriate records in the DB **without revealing their search criteria**.
For more detail about Sommelier DB, see [here](./db/intro.md).

## Overview of Sommelier Drive
Sommelier Drive is a remote file system developed as the first application of Sommelier DB. Its stored files and their file paths are encrypted with the public key of the legitimate user who has read permission to the file. Therefore, **other users who do not have the read permission or the administrator of the server hosting the file system cannot know what files exist and where they are located**. Furthermore, the legitimate users can generate trapdoors using the user's private key, allowing the server to search for the appropriate file without revealing the search criteria.

This is a demo movie of the glibc client for Sommelier Drive!
![Demo](https://user-images.githubusercontent.com/51953536/201111228-d08b5477-8808-41d6-9f05-5bd4487f9ab1.gif)

Its detailed specifications are available [here](./drive/intro.md).

## Disclaimer
**DO NOT USE THIS LIBRARY IN PRODUCTION**. At this point, this is under development. It has known and unknown bugs and security flaws.

## Quick Links
- [Sommelier DB implementation](https://github.com/Sommelier-db/sommelier-db)
- [Server implementation of Sommelier Drive](https://github.com/Sommelier-db/sommelier-drive-server)
- [Client implementation of Sommelier Drive](https://github.com/Sommelier-db/sommelier-drive-client)
- [Glibc client implementation of Sommelier Drive](https://github.com/Sommelier-db/sommelier-drive-glibc/tree/drive)
- [PKSE implementation written in Rust](https://github.com/Sommelier-db/rust-searchable-pke)

## Acknowledgments
The logo illustration for Sommelier DB was provided by Kirari Suegami in 2022.