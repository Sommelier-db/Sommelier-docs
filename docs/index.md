# Sommelier DB
**Open source DB with public-key searchable encryption: discerning encrypted data without knowing the contents, just like a sommelier!**
<img src="./assets/figs/logo.png" width="50%" style="display: block; margin: auto;">

## Overview of Sommelier DB and Drive
Sommelier DB is **an open source DB library combining SQLite and public-key searchable encryption (PKSE)**. In addition to existing SQLite features, it provides functions for PKSE as below.


1. A new SQL function to test a keyword encryption of the PKSE scheme.
2. C and Rust functions to generate a keyword encryption and a trapdoor of the PKSE scheme.


The above functions allow users to have the database (DB) server search for appropriate records in the DB **without revealing their search criteria**.
For more detail about Sommelier DB, see [here](./db/intro.md).

Sommelier Drive is a remote file system developed as the first application of Sommelier DB. Its stored files and their file paths are encrypted with the public key of the legitimate user who has read permission to the file. Therefore, **other users who do not have the read permission or the administrator of the server hosting the file system cannot know what files exist and where they are located**. Furthermore, the legitimate users can generate trapdoors using the user's secret key, allowing the server to search for the appropriate file without revealing the search criteria.
Its detailed specifications are available [here](./drive/intro.md).