# Sommelier DB
**Open source DB with public-key searchable encryption: discerning encrypted data without knowing the contents, just like a sommelier!**

## Overview of Sommelier DB and Drive
Sommelier DB is an open source DB library based on SQLite. In addition to existing SQLite features, it provides functions for public-key searchable encryption (PKSE) as below.
1. A new SQL function to test a keyword encryption of the PKSE scheme.
2. C functions to generate a keyword encryption and a trapdoor of the PKSE scheme.
The above functions allow users to have the database (DB) server search for appropriate records in the DB without revealing their search criteria.
For more detail about Sommelier DB, see [here](./db/intro.md).

Sommelier Drive is a remote file system developed as the first application of Sommelier DB. Stored files and their file paths are encrypted with the public key of the authorized user who has read access to the file. Therefore, other users who do not have read access or the administrator of the server hosting the file system cannot know what files exist and where they are located. Furthermore, legitimate users can generate trapdoors using the user's private key, allowing the server to search for the appropriate file without revealing the search criteria.
Its detailed specifications are available [here](./drive/intro.md).