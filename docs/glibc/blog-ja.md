# glibcの書き換えによるドライブ上のファイルへのアクセス
#### Author 西井遥紀

## はじめに
ここではSommelier Driveが提供するクライアント側のAPIを利用して、Linuxマシン上で既存のコマンドを置き換えることなく通常のファイルシステムと同様にドライブ上のファイルやディレクトリを扱うという試みについて述べる。

Sommelier Driveが提供するドライブを操作するAPIを利用する実装なので、厳密にはSommelier Driveとは別物だが実装の一貫として作成したので、ここに記載させていただく。

## 目的
リモートにあるドライブを通常のファイルシステム同様に扱うという主旨で、今回はシンプルにターミナル上で扱うlsやcatやtouchといったコマンドをリモートのファイルやディレクトリに対しても行えるようにすることを目指した。NFSのようなネットワーク越しのファイルシステムをシステムコールを一つ一つ定義することで新しいファイルシステムを作って、ローカルのファイルシステムにマウントして使えるようにするのが通常だが、実験の時間の都合上、カーネルレベルでの実装はかなり厳しい。そこで、比較的着手しやすいglibcの書き換えによるユーザーレベルでの疑似ファイルシステムを実現を試みた。

なぜglibcかというと、glibcの標準ライブラリlibc.soや、動的リンカld-linux.soというのはreadelfやlddで調べるとわかるように大抵のプログラムやライブラリからリンクされて使用されるからである。これはC言語で書かれたプログラムに限った話ではなく、例えば今回メンバーが作成したRustで書かれたライブラリを調べると、libc.so含めglibcの標準ライブラリが複数リンクされていることがわかる。
``` shell
$ readelf -d librust_searchable_pke.so

Dynamic section at offset 0xf8058 contains 29 entries:
 タグ        タイプ                       名前/値
 0x0000000000000001 (NEEDED)             共有ライブラリ: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             共有ライブラリ: [libm.so.6]
 0x0000000000000001 (NEEDED)             共有ライブラリ: [libc.so.6]
 0x0000000000000001 (NEEDED)             共有ライブラリ: [ld-linux-x86-64.so.2]
 ~~~~~~~~~~~
```
実際にltraceを使用したり、gdbでブレークポイントを貼るなどして調べると、ファイルの操作にはlibc.soのopen/close/read/writeといった関数が呼ばれ、ヒープからメモリの割り当てと解放には最終的にmalloc/free関数が呼ばれる。他にもPython,Rubyといったインタプリタ言語もCで書かれていることから同様にlibcの関数を使用している。
つまりlibcの挙動そのものを変えればプログラムの挙動を変えることができると考え、ファイルシステムを自作するよりは手をつけやすいと感じ、このような試みに至った。しかしアセンブリで書かれたプログラムやGoなど独自の標準ライブラリを実装している言語で書かれたプログラムなどにはこの手法は使えない。

## 挙動の変更方法について
glibcに変更を加えて独自のlibc.soを作成したとして、既存の実行ファイルのリンク先を変更するには、再度リンクするかシステム内のlibc.soを置き換える必要がある。しかし、ソースコードが必ずしも存在するとは限らないし、システムにはなるべく変更を加えたくない。そこで関数のフックという手法を用いることにした。今回使用したのは**LD_PRELOAD**という環境変数で、共有ライブラリを指定すると、実行開始時に動的リンカが指定したライブラリを、実行ファイルで指定されたライブラリより先にメモリにロードするというものである。命令の実行時に共有ライブラリの関数が初めて呼ばれたときに動的リンカに処理が移り、関数を探索してアドレスを解決するという流れがあるが、その時の関数の探索の順番をこのLD_PRELOADにライブラリを指定することで、前に持ってくることができる。つまりLD_PRELOADに本来呼ばれるはずの共有ライブラリの関数を実装したライブラリを指定すると、関数の処理を横取りできることになる(関数を**フック**するという)。この手法を用いて既存のコマンドを書き換えずにlibc.soのファイルを扱う関数の挙動を変更することができる。
　ただし問題はlibc.so内の関数を新たに定義して作成したライブラリをpreloadするだけでは、その関数を呼び出すlibc.so内の関数をフックすることはできない。例えば、write関数を定義したライブラリをpreloadしても、libc.soのfwrite関数が実際に呼び出すのは自身のwrite関数であってフックしたwrite関数ではない。これは同じライブラリにある関数のアドレスは相対的なものなので、わざわざ動的リンカを呼ぶことはないからである。ただファイルをオープンする処理をフックするために、creatやfopenといったすべての関数の処理を一つ一つ定義するのは労力を要するうえ見逃す可能性もある。そこで今回とった方針はglibc全体のソースを持ち出して、フックしたい関数が最終的に行き着く関数(例えばfopenに対してopen)のみを変更してpreloadするライブラリを作成するというもので、かなり無駄があるがlibc.soの関数はすべて揃うのでフックをしそこなう心配はなくなる。


## 具体的な設計の方針
基本的にはファイルシステムを扱うシステムコールを呼び出す関数(正式ではないがここではシステムコール関数という。実際にソース内ではこれらの関数はマクロを用いてシステムコールを呼び出す)を書き換えればいいことになる。Linuxではこれらの関数はglibcソースコード内の`sysdeps/unix/sysv/linux/` 以下にまとまっている。

実際にどの関数をフックする必要があるかは、簡単なコマンドをltraceやstraceでトレースしながら判定する。例えば以下は一つのファイルに対するcatをltraceした結果である。
``` shell
$ ltrace cat file 
################ 必要な部分のみ抜粋
open("file", 00, 037777400000) = open("file", O_RDONLY)        = 3
fstat(3, 0x7ffd1d8874b0)                                       = 0
posix_fadvise(3, 0, 0, 2)                                      = 0
aligned_alloc(4096, 0x20000, 0, 0x7f44742fe47a)                = 0x7f4474462000
read(3, "this is a source file", 131072)                       = 21
write(1, "this is a source file", 21this is a source file)     = 21
read(3, "", 131072)                                            = 0
free(0x7f4474462000)                                           = <void>
close(3)                                                       = 0
################
```
まずfileをread onlyでopenし、返って来たファイル識別子に対して、fstat,readを実行して、最後にcloseする。この場合、リモートのファイルをread権限でopenする際はファイルの内容をDriveのAPIを利用してリモートから取得してローカルのファイルシステムに移しそのファイル識別子を返せば良い。その後のreadはこのローカルのファイルに対して行うので、フックの必要はない。
他の例として、cpのltraceの結果を示す。
``` shell
$ ltrace cp file dup
#################### 必要な部分のみ抜粋
open("dup", O_RDONLY|O_PATH|O_DIRECTORY, 027516111125)      = -1
__errno_location()                                          = 0x7f8fb847f698
fstatat(AT_FDCWD, "file", 0x7fffbd387960)                   = 0
open("file", 00, 00)                                        = 3
fstat(3, 0x7fffbd387b10)                                    = 0
openat(AT_FDCWD, "dup", O_WRONLY|O_CREAT|O_EXCL, 0644)      = 4
__errno_location()                                          = 0x7f8fb847f698
ioctl(4, 1074041865, 0x3)                                   = -1
fstat(4, 0x7fffbd387a80)                                    = 0
posix_fadvise(3, 0, 0, 2)                                   = 0
copy_file_range(3, 0, 4, 0)                                 = 21
copy_file_range(3, 0, 4, 0)                                 = 0
close(4)                                                    = 0
close(3)                                                    = 0
####################
```
まずコピー先のパスがファイルかディレクトリか存在するかどうかをopenで判定し、ファイルである場合はwrite権限でそのファイルを開く。その後内容をコピーしてcloseする。この場合、write権限でドライブ上のパスを指定してopenした場合、ファイルがドライブ上になければ作成し、ローカルのファイルとして開く。更にこのファイルをcloseするときは、変更をドライブに保存するように変更する必要がある。

今回Sommelier Driveから提供されるAPIは以下である。
- addFile
パス名を指定してファイルを作成する
- addDirectory
パス名を指定してディレクトリを作成する
- getChildrenPathes
ディレクトリを指定して、エントリ一覧を取得する
- isExistFilepath
パスの有無を取得する
- modifyFile
ファイルの内容を変更する
- openFilepath
パスを開きパーミッションやファイルの内容を取得する。
- searchDescendantPathes
ディレクトリを指定して子孫の一覧を取得する

これらのAPIを用いて実装できる範囲で、以下に実際に変更を加えたglibcの関数を以下に示す。

- ### open/openat
フックの大部分を構成するシステムコール関数。上に述べたとおり、引数であるファイルのパスとフラグを判定して、パス名がドライブのものであった場合は、フラグの組み合わせに応じてドライブ上のパスに対して特定の処理を行うように設計した。またopenatは第一引数にディレクトリの識別子を指定するが、これがドライブ上のものである場合も、特別な処理が必要になる。今回fopenやcreatなどで使用されるO_RDONLY,　O_WRONLY,　O_RDWR, O_CREAT, O_EXCL, O_TRUNC, O_APPEND, O_DIRECTORYの8つのフラグに対応した。

- ### close/fsync
O_WRONLYかO_RDWRを指定してopenしたドライブ上のファイルはこれらの関数が呼ばれる際にローカルで編集したファイルの内容をドライブに保存する必要がある。
fsyncはファイルの変更をディスクに反映するシステムコール関数である。close関数に加えてこの関数を変更したのは、nanoコマンドでリモートのプログラムを編集できるようにするためである。このコマンドの挙動をltraceで調べたところ、Ctrl-Sが押されるたびにfsync関数を呼び出すが、終了時にclose関数を呼び出さなかった。そのため、ドライブ上のファイルに対してfsyncが呼ばれるたびに変更を反映する処理が必要になった。

- ### fstatat
ファイルやディレクトリの情報を取得する関数で、本環境の実装では内部でnewfstatatというシステムコールを呼び出す。stat/fstat/lstat/statxといった関数も、フラグを指定してこの関数を呼び出すので、変更はこの関数のみでよい(ただし、環境によってはこれらの関数は同名のシステムコールを呼び出すのであくまで本環境のみでの実装である)。現ドライブの実装ではファイルが保持する情報の種類はさほど多くないので、パーミッションとファイルサイズのみ取得し、他の情報は0で埋める設計にした。

- ### opendir/fdopendir/readdir/closedir
ディレクトリのエントリ一覧を返す一連のライブラリ関数であり、ls, findといったコマンドやシェルのパス名補完機能などで用いられている。それぞれopen/getdent/closeといったシステムコール関数を呼び出すが、エントリ一覧を取得するためにこれらが直接呼ばれることはめったにないので、上記の関数のみを変更した。

- ### mkdir
ディレクトリを作成する関数で、リモートのパスが指定されたときに対応するAPIを呼び出すように変更すれば良い。しかしCのファイルが見当たらず、ビルドをたどったところスクリプトでアセンブリを生成しコンパイルしているようだった。一部のシステムコール関数は同様の方法でコンパイルされているようだったが、それらの関数を指定する部分が見当たらなかったので、C言語で記述して強引であるがもとのシンボルを上書きすることで対応した。

## ビルドの方法と問題点
深刻な問題だがmakeによるビルドが失敗する。本来のビルドは単純にビルドディレクトリに移動して`<source path>/configure`と`make` で済むのはずだが、実装に変更を加えたためにこのビルド方法では以下のようなエラーにより止まってしまう。
```
for symbol in __GI___pthread_disable_asynccancel __GI___pthread_enable_asynccancel __pthread_disable_asynccancel __pthread_enable_asynccancel calloc free malloc realloc  __stack_chk_fail __stack_chk_fail_local; do \
        echo ".globl $symbol"; \
        echo "$symbol:"; \
done | gcc -o /home/sitianos/Project/Drive/build/elf/librtld.mapT.o -g -Werror=undef -Wa,--noexecstack  -c -x assembler -
gcc   -nostdlib -nostartfiles -r -o /home/sitianos/Project/Drive/build/elf/librtld.map.o /home/sitianos/Project/Drive/build/elf/librtld.mapT.o '-Wl,-(' /home/sitianos/Project/Drive/build/elf/dl-allobjs.os /home/sitianos/Project/Drive/build/libc_pic.a -lgcc '-Wl,-)' -Wl,-Map,/home/sitianos/Project/Drive/build/elf/librtld.mapT
/usr/bin/ld: /home/sitianos/Project/Drive/build/libc_pic.a(getcwd.os): in function `__GI___getcwd':
/home/sitianos/Project/Drive/glibc-drive/io/../sysdeps/unix/sysv/linux/getcwd.c:49: multiple definition of `__getcwd'; /home/sitianos/Project/Drive/build/elf/dl-allobjs.os:/home/sitianos/Project/Drive/glibc-drive/elf/../sysdeps/unix/sysv/linux/getcwd.c:49: first defined here
/usr/bin/ld: /home/sitianos/Project/Drive/build/libc_pic.a(dl-error.os): in function `__GI__dl_signal_exception':
/home/sitianos/Project/Drive/glibc-drive/elf/dl-error-skeleton.c:91: multiple definition of `_dl_signal_exception'; /home/sitianos/Project/Drive/build/elf/dl-allobjs.os:/home/sitianos/Project/Drive/glibc-drive/elf/dl-error-skeleton.c:91: first defined here
/usr/bin/ld: /home/sitianos/Project/Drive/build/libc_pic.a(dl-error.os): in function `__GI__dl_signal_error':
/home/sitianos/Project/Drive/glibc-drive/elf/dl-error-skeleton.c:109: multiple definition of `_dl_signal_error'; /home/sitianos/Project/Drive/build/elf/dl-allobjs.os:/home/sitianos/Project/Drive/glibc-drive/elf/dl-error-skeleton.c:109: first defined here
/usr/bin/ld: /home/sitianos/Project/Drive/build/libc_pic.a(dl-error.os): in function `__GI__dl_catch_exception':
/home/sitianos/Project/Drive/glibc-drive/elf/dl-error-skeleton.c:175: multiple definition of `_dl_catch_exception'; /home/sitianos/Project/Drive/build/elf/dl-allobjs.os:/home/sitianos/Project/Drive/glibc-drive/elf/dl-error-skeleton.c:175: first defined here
/usr/bin/ld: /home/sitianos/Project/Drive/build/libc_pic.a(dl-error.os): in function `__GI__dl_catch_error':
/home/sitianos/Project/Drive/glibc-drive/elf/dl-error-skeleton.c:225: multiple definition of `_dl_catch_error'; /home/sitianos/Project/Drive/build/elf/dl-allobjs.os:/home/sitianos/Project/Drive/glibc-drive/elf/dl-error-skeleton.c:225: first defined here
collect2: error: ld returned 1 exit status
```
表示にあるように関数の多重定義によるエラーであるが、指摘されている関数は実装を変更したファイル内では用いてない。実際にlibc_pic.aとdl-allobjs.os内のシンボルを確認したところ、確かに同じ関数が定義されているが、ファイルに変更を加えることで、そのような状況になる理由がわからなかった。その上同じ定義を指してのエラーであることから -fcommonオプションを加えたが解消しなかった。このレシピは動的リンカld.soを作成する一環であるが、目的のlibc.soを作成するにはld.soへのリンクが必要であるため、これ以降ビルドは進まない。ld.soを作成する一連のビルド関連ファイルを読んでみたものの、原因の特定には至らなかった。現段階ではかなり強引であるものの以下の方法でビルドできる。
1. `sysdeps/unix/sysv/linux/drive-common.h`の`DRIVE_EXT`マクロを0にセットして、`configure`と`make`でビルドする。
2. `sysdeps/unix/sysv/linux/drive-common.h`の`DRIVE_EXT`マクロを1にセットして、`make -k`を実行する。

1により変更を含めない通常通りのビルドを実行してlibc.soに必要なファイルが生成する。2ではld.so作成上のエラーを無視して、変更を加えた関数を含めたlibc.soを作成する。依存上で必要なファイルは1で作成されているので、ld.soの作成でエラーは出力されるものの、目的のlibc.soは生成される。
このように強引なビルド方法なので、ビルド後のバイナリとしてdebパッケージを作成した。

2つ目の問題として、libc.soをLD_PRELOADに指定してコマンドを実行した際に、Segmentation Faultで落ちてしまうというものがあった。gdbで調べたところ、共有ライブラリをロードする際にアクセス違反が生じていた。同様のエラーが発生した例を探したところ、[CTFの記事](https://qiita.com/kusano_k/items/ab35d5982011eb0f742e#%E8%BF%BD%E8%A8%98 "libc.soを差し替えてプログラムを動かす（のは無理そう）")が見つかり、これによるとlibc.soはld-linux.soの構造体を参照しているが、このメンバの配置がバージョンごとに異なるため、実際に動いているld-linux.soとバージョンを合わせる必要があるという。そのため、glibcのバージョンを実機のものに揃えて再ビルドして、preloadしたところ問題なく関数のフックが成功した。この制約は非常に問題なので、フックに必要な関数の依存を調べ部分的にリンクして、動的リンカを必要としないライブラリを制作しようとしたが、ビルド関連の設定が複雑であったので、ひとまずglibcバージョンを揃えたVMなどの環境で動かすことで対応した。

## 挙動のテスト
※ 実機で行うと思わぬバグを生じる可能性があるため以下の環境を構築したVM上で行う。

### 環境
- OS: Linux (Debian系)
- Arch: amd64
- libc: libc6 2.35 (上に述べた理由から現状ではこのバージョンのみ)

### インストール
システムのglibcのバージョンを確認する
``` shell
$ /usr/lib/x86_64-linux-gnu/libc.so.6
GNU C Library (Debian GLIBC 2.35-3) stable release version 2.35.
Copyright (C) 2022 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
Compiled by GNU CC version 11.3.0.
libc ABIs: UNIQUE IFUNC ABSOLUTE
For bug reporting instructions, please see:
<http://www.debian.org/Bugs/>.
```
debパッケージは以下のような構成になっている
``` shell
$ dpkg-deb -c sommelier-drive-glibc_1.1_amd64.deb
drwxr-xr-x root/root         0 2022-11-09 04:38 ./
drwxr-xr-x root/root         0 2022-11-09 04:39 ./usr/
drwxr-xr-x root/root         0 2022-11-09 06:42 ./usr/bin/
-rwxr-xr-x root/root     16264 2022-11-09 06:30 ./usr/bin/sommelier-register
-rwxr--r-- root/root      1138 2022-11-09 06:42 ./usr/bin/sommelier-run
drwxr-xr-x root/root         0 2022-11-09 04:47 ./usr/include/
-rw-r--r-- root/root      2372 2022-11-09 04:47 ./usr/include/sommelier_drive_client.h
drwxr-xr-x root/root         0 2022-11-09 04:38 ./usr/lib/
drwxr-xr-x root/root         0 2022-11-09 06:42 ./usr/lib/x86_64-linux-gnu/
-rwxr-xr-x root/root  10318664 2022-11-09 04:47 ./usr/lib/x86_64-linux-gnu/libsommelier_drive_client.so
-rwxr-xr-x root/root  12711832 2022-11-09 06:32 ./usr/lib/x86_64-linux-gnu/libsommelier_libc.so
```
インストールを実行する
```
sudo dpkg -i sommelier-drive-glibc_1.1_amd64.deb
```

### ユーザーの登録
立ち上げたドライブのサーバに対して、urlとリージョン名とホームディレクトリのパスを指定してユーザを登録すると、以下のようなファイルが生成される。
``` shell
$ sommelier-register "http://localhost:8000/api" "region" "/alice"
successfully registered. user id is 1
configuration is saved in drive_envs

$ cat drive_envs
# key to encrypt data
print_data_sk () {
cat << END_OF_FILE
-----BEGIN PRIVATE KEY-----
MIIEvAIBADANBgkqhkiG9w0BAQEFAASCBKYwggSiAgEAAoIBAQDEvumJhkNvwPzx
1bB52SLrP6wS+DL+l+umOJ0rIsuuwWcbRS7cRejWwN7GYC0og6r0or
~~~~~~~~~~~~~~~~~~~~~~~~
j5WFyup4lgD0uwH9TLXsZj5s13fYs6HhyOBhJyDMbNvkz46Bi0U2kDQvAmiSAmCy
ixu5XBCophbUe+H8uevbNQ==
-----END PRIVATE KEY-----
END_OF_FILE
}

# key to use for searchable encryption
print_keyword_sk () {
cat << END_OF_FILE
{"alphas":[[8781683650687187445,1371472541923046913,15906317983419354644,7861644468431804003],
~~~~~~~~~~~~~~~~~~~~
,169,67,239,123,187,54,92,20,230,126,253,242,134,179,34,38,135,162,88,13,158,42,195,234,189,122,158,196,206]}
END_OF_FILE
}

# client user id
export SOMMELIER_DRIVE_USER_ID=1
# server to connect to
export SOMMELIER_DRIVE_BASE_URL=http://localhost:8000/api
# server region
export SOMMELIER_DRIVE_REGION_NAME=region
# remote home directory
export SOMMELIER_DRIVE_HOME_DIR=/alice
```
このファイルにドライブへの接続に必要なサーバの情報と、2種類の秘密鍵を含んだクライアントの情報を環境変数に設定する記述がなされている。

### コマンドの実行
コマンドを実行するラッパーとなる`sommelier-run`は以下のようなスクリプトである。
``` sh:/usr/bin/sommelier-run
#!/bin/sh
# directory on which files on the remote drive are put
export SOMMELIER_DRIVE_BASE_DIR=/tmp/drive

rm -rf $SOMMELIER_DRIVE_BASE_DIR
mkdir -p $SOMMELIER_DRIVE_BASE_DIR
chmod 700 $SOMMELIER_DRIVE_BASE_DIR

# whether to trace library calls
export SOMMELIER_DRIVE_TRACE=0

# load server info and user info
. $PWD/drive_envs
SOMMELIER_DRIVE_DATA_SK=$(mktemp -p $SOMMELIER_DRIVE_BASE_DIR)
print_data_sk > $SOMMELIER_DRIVE_DATA_SK
export SOMMELIER_DRIVE_DATA_SK

SOMMELIER_DRIVE_KEYWORD_SK=$(mktemp -p $SOMMELIER_DRIVE_BASE_DIR)
print_keyword_sk > $SOMMELIER_DRIVE_KEYWORD_SK
export SOMMELIER_DRIVE_KEYWORD_SK

# library path of Sommelier Drive API
export SOMMELIER_DRIVE_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/libsommelier_drive_client.so

# library to preload for function hooking
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libsommelier_libc.so

# execute privided command
exec $@
```
これに示す通り、ユーザ登録時に作成した設定ファイルと、Sommelier DriveのAPIを保持するライブラリを環境変数に指定し、LD_PRELOADに作成した関数フックのためのlibcライブラリを指定して与えられたコマンドを実行する。
このスクリプトを用いて、ファイルを扱うコマンドを以下のように実行してドライブ上のファイルを操作する。
``` shell
$ sommelier-run command [args]
```
実際に動作する標準コマンドをいくつか紹介する。
まずファイルとディレクトリを作成する。ドライブ上のパスは先頭に"sommelier:"をつけて指定する。
``` shell
$ sommelier-run touch sommelier:/alice/file1

$ sommelier-run mkdir sommelier:/alice/dir1
```
これで実際にドライブ上にファイルとディレクトリが作成された。cpコマンドにより、ローカル-リモート間、リモート-リモート間でファイルを複製できることも確認する。
``` shell
$ echo "this is a local file" > localfile

$ sommelier-run cp localfile sommelier:/alice/file2

$ sommelier-run cat sommelier:/alice/file2
this is a local file

$ sommelier-run cp sommelier:/alice/file2 sommelier:/alice/file3

$ sommelier-run cat sommelier:/alice/file3
this is a local file

$ sommelier-run cp sommelier:/alice/file2 remotefile

$ cat remotefile
this is a local file
```

次にlsコマンドによりリモートに作成したファイルとディレクトリを確認する。
``` shell
$ sommelier-run ls -l sommelier:/alice/
total 0
drw------- 0 alice alice  0 Jan  1  1970 dir1
-rw------- 0 alice alice  0 Jan  1  1970 file1
-rw------- 0 alice alice 21 Jan  1  1970 file2
```
ドライブの設計の都合でstatの返り値に空白が多いため、情報の表示がはいい加減であるが、エントリの表示ができていることは確認できる。

``` shell
$ sommelier-run nano sommelier:/alice/file3
```
最後にnanoエディタを試す。
``` shell
$ sommelier-run nano sommelier:/alice/file3
```

ファイルの編集は滞りなくできるものの、Ctrl-Sで変更を保存するのにはかなり時間を要する。
<img width="50%" src="../assets/figs/nano.png">

``` shell
$ sommelier-run cat sommelier:/alice/file3
nano opened
```

## 終わりに
今回、カーネルレベルで変更を加える代わりにシステムコールの挙動をLD_PRELOADという環境変数を用いてシステムコールを呼ぶ関数をフックするという手法で変更したが、この他にも関数のフックの手法はあるようだ。straceやgdbで使用されているptraceシステムコールを用いるものもあれば、この[記事](https://yasukata.hatenablog.com/entry/2021/10/14/145642)で紹介されている高速にシステムコールフックを実現する興味深い手法も存在する。LD_PRELOADをフックの用途で用いるのは、ソケット通信のラッパーでよく目にする(私が初めてLD_PRELOADによるフックを知ったのはproxychainsというコマンド内での通信をsocks5サーバ経由にするツールのスクリプトを読んだときであった)。しかし、ファイルシステムに対する処理を書き換える試みは一般的でないようで、すでに述べたビルド上の問題に加え、乱雑な実装になり課題を残す結果となった。それでも最低限の機能は実現できたため、この取り組みに意義はあったと感じている。本実装はSommelier Drive実装の延長としてAPIを利用する一つの方針としてオープンソースのプロジェクトであるglibcを変更するという案から実現したものである。機能の改善というより、個人利用の開発に近いものとなったが、巨大プロジェクトに触れるいい機会であった。

## 参考にした記事
[1] [libc.soを差し替えてプログラムを動かす（のは無理そう）](https://qiita.com/kusano_k/items/ab35d5982011eb0f742e#%E8%BF%BD%E8%A8%98)

[2] [ptrace より 100 倍速いシステムコールフック作った](https://yasukata.hatenablog.com/entry/2021/10/14/145642)
