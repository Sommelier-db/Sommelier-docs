## Requirements
To run the server, the following tools are required:

- [Sommelier-DB](https://github.com/Sommelier-db/sommelier-db)
- jasson (json library)

To run the client, the following tools are required:

- rustc 1.65.0-nightly
- cargo 1.65.0-nightly
- cbindgen 0.24.3

## Setup
We open the **first** terminal for running the server and clone the server repository.
```bash
git clone https://github.com/Sommelier-db/sommelier-drive-server
cd sommelier-drive-server
git submodule update --init --recursive
```

The sommelier-drive-server is built as below:
```bash
make init
make
```

We then open the **second** terminal for running the client CLI and clone the client repository.
```bash
git clone https://github.com/Sommelier-db/sommelier-drive-client
cd sommelier-drive-client
git submodule update --init --recursive
```

The sommelier-drive-client is built by simply executing the `build.sh`.
```bash
./build.sh
```

## Operate files with Sommelier Drive!
Now, let's operate files with Sommelier Drive!
You will notice that the UX of Sommelier Drive is almost the same as existing Unix file systems, which you probably use every day.

On the **first** terminal, we run the server.
```bash
build/main
```
If there is no issue, the server will stand at the URL `http://localhost:8000/api`.

On the **second** terminal, we manage the files on that server.
While performing the following operations, also look at the first terminal running the server.
You can find that no file path or contents appear on the server side!

We first create a new user account and creates an initial directory `/data1` under the root directory.
```bash
>> register /data1
``` 
We then create a file and a directory under the `/data1`.
```bash
>> touch /data1/hello.txt
>> mkdir /data1/subdata1
``` 
Say hello to Sommelier Drive in the `/data1/hello.txt`!
```bash
>> cat /data1/hello.txt
>> modify /data1/hello.txt Hello,SommelierDrive!
>> cat /data1/hello.txt
``` 
We can check the children files or directories under the `/data1`.
```bash
>> ls /data1
``` 
We finally create a file in a deeper location and find them.
```bash
>> touch /data1/subdata1/goodbye.c
>> find /data1
``` 