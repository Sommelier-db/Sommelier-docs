## Storing files on a remote server without revealing what files exist and where they are located.
As the first application of Sommelier DB, we developed a new remote file system, Sommelier Drive. Like existing services, such as Dropbox, Google Drive, and One Drive, files can be stored on a remote server, but the file contents and their file paths are all encrypted. Therefore, **even a malicious administrator of that server cannot know what files exist and where they are located**.

In a nutshell, a data provider who creates a new file, a user who has read permission to that file, and the remote server work as below.

1. The data provider encrypts the file contents and paths with the user's public key and stores their encryptions on the remote server.
2. The user generates a trapdoor of the PKSE scheme with the user's private key and provides it for the remote server.
3. The remote server returns the file contents encryption of which the corresponding file path encryption matches with the provided trapdoor.
4. The user decrypts the returned encryption with the user's private key, which results in the file contents.

The above scheme may seem almost the same as the Sommelier DB. The difference is that the user's private and public keys consist of those of PKSE and public-key encryption (PKE). The PKSE scheme is used to allow the server to search for files in the queried file path without revealing them. In addition, the PKE scheme is necessary to obtain the file contents because only with the PKSE scheme the user cannot recover the encrypted message even if the PKSE private key is available. In other words, Sommelier Drive requires the PKSE and PKE scheme to test and recover the file path and contents, respectively.