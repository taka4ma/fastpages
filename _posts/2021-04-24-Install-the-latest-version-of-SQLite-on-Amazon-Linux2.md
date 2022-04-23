---
toc: true
layout: post
description: In Creating a Django Development Environment on Cloud9, the steps to install the latest version of SQLite on Amazon Linux2, the OS running Cloud9, in order to build a Django development environment on Cloud9.
categories: [sqlite, amazon_linux2]
title: Amazon Linux2に最新バージョンのSQLiteをインストールする
---

# Amazon Linux2に最新バージョンのSQLiteをインストールする

この記事は、[Cloud9にDjango開発環境を作る](https://taka4ma.github.io/contents/0008.html)で、Cloud9でDjangoの開発環境を構築するため、Cloud9を実行しているOS、Amazon Linux2に最新バージョンのSQLiteをインストールする手順をまとめたもの。

まずは、yumでインストールできるか確認する。

```
yum list | grep ^sqlite
sqlite.x86_64                          3.7.17-8.amzn2.1.1            @amzn2-core
sqlite-devel.x86_64                    3.7.17-8.amzn2.1.1            @amzn2-core
sqlite.i686                            3.7.17-8.amzn2.1.1            amzn2-core 
sqlite-doc.noarch                      3.7.17-8.amzn2.1.1            amzn2-core 
sqlite-tcl.x86_64                      3.7.17-8.amzn2.1.1            amzn2-core 
sqlite2.x86_64                         2.8.17-17.el7                 epel       
sqlite2-devel.x86_64                   2.8.17-17.el7                 epel       
sqlite2-tcl.x86_64                     2.8.17-17.el7                 epel       
sqlite3-dbf.x86_64                     2011.01.24-3.el7              epel
```

yumでインストールsqlite3は3.7.17だけのようだ。

Amazon Linux2のExtras Libraryではどうだろうか。[Amazon Linux 2 EC2 インスタンスに Extras Library からソフトウェアをインストールする](https://aws.amazon.com/jp/premiumsupport/knowledge-center/ec2-install-extras-library-software/)を参考に進める。

`amazon-linux-extras` がインストールされているか確認する。

```
$ which amazon-linux-extras
/usr/bin/amazon-linux-extras
```

インストールされているようだ。
`amazon-linux-extras`を実行すると、利用可能なトピックが表示されるはずなので実行してみる。

```
$ sudo amazon-linux-extras | tail
 46  collectd                        available    [ =stable ]
 47  aws-nitro-enclaves-cli          available    [ =stable ]
 48  R4                              available    [ =stable ]
 49  kernel-5.4                      available    [ =stable ]
 50  selinux-ng                      available    [ =stable ]
  _  php8.0                          available    [ =stable ]
 52  tomcat9                         available    [ =stable ]
 53  unbound1.13                     available    [ =stable ]
  _  mariadb10.5                     available    [ =stable ]
 55  kernel-5.10                     available    [ =stable ]
```

実行できるようだ。sqliteはあるだろうか。

```
$ sudo amazon-linux-extras | grep sqlite
$
```

残念ながら、Extras Libraryにはないようだ。

[Amazon Linux 2でSQLite3を最新バージョンにする \- Qiita](https://qiita.com/kai_kou/items/c18b68a7916251231f6d)を参考に、SQLiteのソースコードを取得して、ビルドすることにする。

まずは、ビルドに必要なパッケージをインストールする

```

$ sudo yum install -y wget tar gzip gcc make
Loaded plugins: extras_suggestions, langpacks, priorities, update-motd
amzn2-core                            | 3.7 kB  00:00:00     
244 packages excluded due to repository priority protections
Package wget-1.14-18.amzn2.1.x86_64 already installed and latest version
Package 2:tar-1.26-35.amzn2.x86_64 already installed and latest version
Package gzip-1.5-10.amzn2.x86_64 already installed and latest version
Package gcc-7.3.1-12.amzn2.x86_64 already installed and latest version
Package 1:make-3.82-24.amzn2.x86_64 already installed and latest version
Nothing to do
```

[SQLite Download Page](https://www.sqlite.org/download.html)にアクセスし、ソースコードのアーカイブのURLを調べる。必要なアーカイブは `sqlite-autoconf-xxxxxxx.tar.gz` (xxxxxxxにはバージョンが入る。この記事作成時点のファイル名は `sqlite-autoconf-3350500.tar.gz` )

URLがわかったらダウンロードする。(environmentディレクトリへ含む必要がないので、ホームディレクトリへダウンロードすることにする。)

```
$ cd
$ wget https://www.sqlite.org/2021/sqlite-autoconf-3350500.tar.gz
--2021-05-02 18:43:00--  https://www.sqlite.org/2021/sqlite-autoconf-3350500.tar.gz
Resolving www.sqlite.org (www.sqlite.org)... 45.33.6.223, 2600:3c00::f03c:91ff:fe96:b959
Connecting to www.sqlite.org (www.sqlite.org)|45.33.6.223|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2956627 (2.8M) [application/x-gzip]
Saving to: ‘sqlite-autoconf-3350500.tar.gz’

100%[===============================================================================================>] 2,956,627   2.31MB/s   in 1.2s   

2021-05-02 18:43:02 (2.31 MB/s) - ‘sqlite-autoconf-3350500.tar.gz’ saved [2956627/2956627]
```

ダウンロードしたら、configure, make, make installする。

まずは解凍してconfigureする。configureに与えている引数 `--prefix` でインストール先を設定する。(今回は `/usr/local` 。実際には `/usr/loca/bin` へインストールされる。)

```
$ tar zxvf sqlite-autoconf-3350500.tar.gz
< tarの出力は省略 >
$ cd sqlite-autoconf-3350500/
$ ./configure --prefix=/usr/local
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for a thread-safe mkdir -p... /usr/bin/mkdir -p
checking for gawk... gawk
checking whether make sets $(MAKE)... yes
checking whether make supports nested variables... yes
< 中略 >
configure: creating ./config.status
config.status: creating Makefile
config.status: creating sqlite3.pc
config.status: executing depfiles commands
config.status: executing libtool command
```

configureできたらmakeする。


```
$ make
/bin/sh ./libtool  --tag=CC   --mode=compile gcc -DPACKAGE_NAME=\"sqlite\" -DPACKAGE_TARNAME=\"sqlite\" -DPACKAGE_VERSION=\"3.35.5\" -DPACKAGE_STRING=\"sqlite\ 3.35.5\" -DPACKAGE_BUGREPORT=\"http://www.sqlite.org\" -DPACKAGE_URL=\"\" -DPACKAGE=\"sqlite\" -DVERSION=\"3.35.5\" -DSTDC_HEADERS=1 -DHAVE_SYS_TYPES_H=1 -DHAVE_SYS_STAT_H=1 -DHAVE_STDLIB_H=1 -DHAVE_STRING_H=1 -DHAVE_MEMORY_H=1 -DHAVE_STRINGS_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_STDINT_H=1 -DHAVE_UNISTD_H=1 -DHAVE_DLFCN_H=1 -DLT_OBJDIR=\".libs/\" -DHAVE_FDATASYNC=1 -DHAVE_USLEEP=1 -DHAVE_LOCALTIME_R=1 -DHAVE_GMTIME_R=1 -DHAVE_DECL_STRERROR_R=1 -DHAVE_STRERROR_R=1 -DHAVE_READLINE_READLINE_H=1 -DHAVE_READLINE=1 -DHAVE_POSIX_FALLOCATE=1 -DHAVE_ZLIB_H=1 -I.    -D_REENTRANT=1 -DSQLITE_THREADSAFE=1 -DSQLITE_ENABLE_MATH_FUNCTIONS -DSQLITE_ENABLE_FTS4 -DSQLITE_ENABLE_FTS5 -DSQLITE_ENABLE_JSON1 -DSQLITE_ENABLE_RTREE -DSQLITE_ENABLE_GEOPOLY -DSQLITE_HAVE_ZLIB  -g -O2 -MT sqlite3.lo -MD -MP -MF .deps/sqlite3.Tpo -c -o sqlite3.lo sqlite3.c
< 中略 >
libtool: link: gcc -D_REENTRANT=1 -DSQLITE_THREADSAFE=1 -DSQLITE_ENABLE_MATH_FUNCTIONS -DSQLITE_ENABLE_FTS4 -DSQLITE_ENABLE_FTS5 -DSQLITE_ENABLE_JSON1 -DSQLITE_ENABLE_RTREE -DSQLITE_ENABLE_GEOPOLY -DSQLITE_HAVE_ZLIB -DSQLITE_ENABLE_EXPLAIN_COMMENTS -DSQLITE_ENABLE_DBPAGE_VTAB -DSQLITE_ENABLE_STMTVTAB -DSQLITE_ENABLE_DBSTAT_VTAB -g -O2 -o sqlite3 sqlite3-shell.o sqlite3-sqlite3.o  -lreadline -ltermcap -lz -lm -ldl -lpthread
```

最後にmake installする。

```
$ sudo make install
make[1]: Entering directory `/home/ec2-user/sqlite-autoconf-3350500'
 /usr/bin/mkdir -p '/usr/local/lib'
 /bin/sh ./libtool   --mode=install /usr/bin/install -c   libsqlite3.la '/usr/local/lib'
libtool: install: /usr/bin/install -c .libs/libsqlite3.so.0.8.6 /usr/local/lib/libsqlite3.so.0.8.6
libtool: install: (cd /usr/local/lib && { ln -s -f libsqlite3.so.0.8.6 libsqlite3.so.0 || { rm -f libsqlite3.so.0 && ln -s libsqlite3.so.0.8.6 libsqlite3.so.0; }; })
libtool: install: (cd /usr/local/lib && { ln -s -f libsqlite3.so.0.8.6 libsqlite3.so || { rm -f libsqlite3.so && ln -s libsqlite3.so.0.8.6 libsqlite3.so; }; })
libtool: install: /usr/bin/install -c .libs/libsqlite3.lai /usr/local/lib/libsqlite3.la
libtool: install: /usr/bin/install -c .libs/libsqlite3.a /usr/local/lib/libsqlite3.a
libtool: install: chmod 644 /usr/local/lib/libsqlite3.a
libtool: install: ranlib /usr/local/lib/libsqlite3.a
libtool: finish: PATH="/sbin:/bin:/usr/sbin:/usr/bin:/sbin" ldconfig -n /usr/local/lib
----------------------------------------------------------------------
Libraries have been installed in:
   /usr/local/lib

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the '-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the 'LD_LIBRARY_PATH' environment variable
     during execution
   - add LIBDIR to the 'LD_RUN_PATH' environment variable
     during linking
   - use the '-Wl,-rpath -Wl,LIBDIR' linker flag
   - have your system administrator add LIBDIR to '/etc/ld.so.conf'


See any operating system documentation about shared libraries for
more information, such as the ld(1) and ld.so(8) manual pages.
----------------------------------------------------------------------
 /usr/bin/mkdir -p '/usr/local/bin'
  /bin/sh ./libtool   --mode=install /usr/bin/install -c sqlite3 '/usr/local/bin'
libtool: install: /usr/bin/install -c sqlite3 /usr/local/bin/sqlite3
 /usr/bin/mkdir -p '/usr/local/include'
 /usr/bin/install -c -m 644 sqlite3.h sqlite3ext.h '/usr/local/include'
 /usr/bin/mkdir -p '/usr/local/share/man/man1'
 /usr/bin/install -c -m 644 sqlite3.1 '/usr/local/share/man/man1'
 /usr/bin/mkdir -p '/usr/local/lib/pkgconfig'
 /usr/bin/install -c -m 644 sqlite3.pc '/usr/local/lib/pkgconfig'
make[1]: Leaving directory `/home/ec2-user/sqlite-autoconf-3350500'
```

makeが終わったら確認する。

```
$ which sqlite3
/usr/local/bin/sqlite3
$ sqlite3 --version
3.35.5 2021-04-19 18:32:05 1b256d97b553a9611efca188a3d995a2fff712759044ba480f9a0c9e98fae886
```

SQLite 3.35.5がインストールでき、Pathも通っているようだ。Pythonからはどうだろうか。
```
$ python -c "import sqlite3;print(sqlite3.sqlite_version)"
3.7.17
```

pythonからは3.7.17のSQLiteが見えているようだ。make installした時に表示された ↓ のメッセージに従って、 `/usr/local/lib` にインストールされたライブラリとリンクする必要がある。

```
Libraries have been installed in:
   /usr/local/lib

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the '-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the 'LD_LIBRARY_PATH' environment variable
     during execution
   - add LIBDIR to the 'LD_RUN_PATH' environment variable
     during linking
   - use the '-Wl,-rpath -Wl,LIBDIR' linker flag
   - have your system administrator add LIBDIR to '/etc/ld.so.conf'
```

[CentOS7でldconfigを使って共有ライブラリを追加する \- Qiita](https://qiita.com/Esfahan/items/0064d845ca6faf7f3d47) を参考進めていく。

現在認識されているsqlite3のライブラリのパスを確認する。

```
$ sudo ldconfig -p | grep sqlite
        libsqlite3.so.0 (libc6,x86-64) => /lib64/libsqlite3.so.0
        libsqlite3.so (libc6,x86-64) => /lib64/libsqlite3.so
```

`/etc/ld.so.conf` の内容を確認する。

```
$ cat /etc/ld.so.conf
include ld.so.conf.d/*.conf
```

`/etc/ld.so.conf.d/` に `.conf` で終わるファイルをおけば良さそう。

```
$ sudo bash -c "echo /usr/local/lib > /etc/ld.so.conf.d/sqlite3.conf"
$ sudo ls /etc/ld.so.conf.d/
bind-export-x86_64.conf  kernel-4.14.165-131.185.amzn2.x86_64.conf  kernel-4.14.231-173.360.amzn2.x86_64.conf
dyninst-x86_64.conf      kernel-4.14.225-168.357.amzn2.x86_64.conf  sqlite3.conf
$ sudo cat /etc/ld.so.conf.d/sqlite3.conf
/usr/local/lib
```

キャッシュファイルを更新し、sqlite3のライブラリのパスを確認する。

```
$ sudo ldconfig
$ sudo ldconfig -p | grep sqlite
        libsqlite3.so.0 (libc6,x86-64) => /usr/local/lib/libsqlite3.so.0
        libsqlite3.so.0 (libc6,x86-64) => /lib64/libsqlite3.so.0
        libsqlite3.so (libc6,x86-64) => /usr/local/lib/libsqlite3.so
        libsqlite3.so (libc6,x86-64) => /lib64/libsqlite3.so
```

追加できている。
あらためて、Pythonからどう見えるか確認する。


```
$ python -c "import sqlite3;print(sqlite3.sqlite_version)"
3.35.5
```

インストールした3.35.5が見えるようになった。