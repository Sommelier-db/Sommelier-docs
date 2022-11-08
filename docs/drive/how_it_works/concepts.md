Here, we describe new concepts of Sommelier Drive and how they optimize the processes of the data provider and the remote server.

## Permission hash for the optimization of file path testing on the remote server.
First, we introduce a permission hash. This is a hash value calculated from the ID of a legitimate user having read permission and the file path of the parent directory. The data provider, in addition to the file path encryption, stores the permission hash on the remote server. When requesting a file, the user also provides the remote server with the permission hash corresponding to the requested file path, allowing the remote server to narrow down the candidates for file path encryptions to test. In this way, the permission hash optimizes the file path testing on the remote server.

The attentive reader may think that the permission hash helps the malicious server administrator learn information about the file path. In fact, **it reveals only information about sibling files and directories, i.e., those contained in the same parent directory**. This is because the administrator cannot learn the actual file paths from their permission hash or encryptions but can specify the siblings by comparing the permission hashes.

## Shared key for the optimization of encrypting a file for multiple users.
A shared key is just a secret key of symmetric key encryption (SKE), e.g., AES. It improves the efficiency when multiple users have read permission for the same file. In more detail, the file contents are encrypted with a fresh shared key, and the shared key is encrypted with each user's PKE public key. The legitimate user can obtain the file contents by (1) recovering the shared key with the PKE private key and (2) decrypting the SKE ciphertexts with the shared key. 

To analyze the efficiency, we compare the ciphertext size in the above scheme with that of the case where the file contents are directly encrypted with each user's PKE public key. Let $N, d, k$ be the number of the legitimate users, the file contents size, and the shared key size. In the former case, the ciphertext size grows in the order $\mathcal{O}(kN)$, whereas the order of the latter case is $\mathcal{O}(dN)$. Therefore, our scheme is more efficient when $d$ is sufficiently large.

