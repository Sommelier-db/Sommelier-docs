# Access Sommelier Drive with shell commands
#### Author: Nishii Haruki

## Introduction
 
 I aimed to access files and directorA user level file system for Sommelier Driveies in Sommelier Drive as well as ones in local file system. I tried to make it possible to create, read and edit them with simple commands run on the terminal such as “cat” and “ls” without re-compiling them. The simple way to achieve this may be implementing original file system like NFS by defining behavior of system calls. However, it should be tough because there are too many system calls to define and testing original file system requires knowledge about how to emulate kernel. Another reason I didn’t choose to build file system was because I couldn’t take advantage of public-key search encryption with file system. When we search a file whose name you only know, or get all the entries in a directory, we may use commands such as “find” and these commands seek it recursively in the directory. But with public-key search encryption algorithm, we only have to generate a trapdoor corresponding to the search criteria, send it to the drive server, and then receive a set of data which matches the criteria. As far as I know, there is no system call that searches files in such an efficient way. Considering this advantage, I decided to extend glibc.

## How to hook functions of glibc

 The reason I extended glibc was that most executables and libraries link libraries of glibc. While most parts of glibc are written in C or assembly language, programs which is written in C/C++ (needless to say), Rust, Python and so on depend heavily on glibc. These programs use malloc/free of libc.so for memory allocation, and open/close/read/write for manipulating files and directories. Therefore, if we alter the behavior of libc.so, we will also alter the behavior of most programs without changes in programs themselves as a result.

## behavior of original glibc

 Then I’m going to explain the example of implementation of my original glibc. When a program opens a file for the purpose of reading, it calls a function of libc.so like open() and fopen() with the file path and flags. I had to alter the behavior of such function so that if the file path is in Sommelier Drive, the function fetches the contents of file from remote server, copies it to local file system, and then returns a file descriptor of the local file. Another example is that when a program opens a file with creating and writing flags, the open function should check if the file path is remote one, and if so, create a file in the remote drive, and returns a file descriptor of the local file. When the program closes it by calling a function such as close() and fclose(), the function saves the contents of file in the remote drive.

## For more detail
 Sorry, but I have no time to write everything about this attempt. Please refer to Japanese version of this blog.
