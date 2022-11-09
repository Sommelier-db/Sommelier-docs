Author: Sora Suegami

In this post, I describe how we implemented [our new library of PKSE](https://github.com/Sommelier-db/rust-searchable-pke). As described in its README.md, our library provides Public-key Encryption with Conjunctive and Disjunctive Keyword search (PECDK) [1] as an underlying scheme. The PECDK supports conjunction and disjunction of multiple keywords as search criteria, which is rather expressive in the research field of public-key searchable encryption (PKSE). However, to provide the same level of functionality as SQL, more complex search criteria need to be addressed. **We filled this gap without implementing new PKSE schemes by devising how to construct the keywords to be encrypted in the PECDK scheme.**

Specifically, our library supports **prefix** and **range** search. The former encrypts a string and retrieves the encryption whose string has the specified prefix, and the latter encrypts an unsigned integer and retrieves the encryption whose integer is within the specified range. In the following, I explain those background designs and implementations.

## Prefix search
In the prefix search, a string is decomposed into bytes, and a pair of each byte and its index forms a single keyword. For example, a string `"abcde"` will be keywords `[(0,"a"), (1, "b"), (2,"c"), (3,"d"), (4, "e")]`. In our library, this keyword construction is implemented in Rust as below.
```
let mut keywords = bytes
        .into_iter()
        .enumerate()
        .map(|(idx, byte)| {
            concat_multi_bytes(vec![
                region_name.as_bytes(),
                &idx.to_be_bytes(),
                &vec![1u8, *byte],
            ])
        })
        .collect::<Vec<Vec<u8>>>();
```
The variable `idx` and `byte` represent the index and the byte, respectively. The `region_name` is a fixed, application-specific string used to distinguish the same string encrypted for different applications. The byte `1` is also necessary to distinguish an empty keyword, namely all-zero bytes. The function `concat_multi_bytes` concats all of the given bytes. In this way, with minor exceptions, our implementation is consistent with the above description.



## Range search

## Reference
1. Zhang, Y., Li, Y., & Wang, Y. (2019). Secure and efficient searchable public key encryption for resource constrained environment based on pairings under prime order group. Security and Communication Networks, 2019.
2. Faber, S., Jarecki, S., Krawczyk, H., Nguyen, Q., Rosu, M., & Steiner, M. (2015, September). Rich queries on encrypted data: Beyond exact matches. In European symposium on research in computer security (pp. 123-145). Springer, Cham.