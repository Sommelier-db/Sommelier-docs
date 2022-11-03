# Sommelier DB
**Open source DB with public-key searchable encryption: discerning encrypted data without knowing the contents, just like a sommelier!**

## What is Sommelier DB ?
Sommelier DB is a open source DB library developed based on SQLite. In addition to existing SQLite features, it provides functions for public-key searchable encryption (PKSE) as below.
1. A new SQL function to test a keyword encryption of the PKSE scheme.
2. C functions to generate a keyword encryption and a trapdoor of the PKSE scheme.
Using the above functions, a user can allow a database (DB) server to search for the appropriate records in the DB without revealing the search criteria.


## What is Sommelier Drive ?
Sommelier Drive is a remote file system using Sommelier DB. The stored files and their file path are encrypted with the public key of the authorized user who has read permission for the files. Therefore, other users who do not have read permission and the administrator of the server hosting the file system cannot know where and what kind of file exists. Moreover, the legitimate user can generate a trapdoor using the user's secret key, which allows the server to search for the appropriate file without revealing the search criteria.