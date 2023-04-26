---
toc: true
layout: post
description: このドキュメントは、Rocky Linux 8をインストールした複数のノードにmariadbを使ってGalera Clusterを構築する手順を提供します
categories: [galera, rocky-linux]
title: GaleraCluster構築手順
---

## はじめに

このドキュメントは、Rocky Linux 8をインストールした複数のノードにmariadbを使ってGalera Clusterを構築する手順を提供します。

Galera Clusterは奇数のノード数で構成することが推奨されます。
これは、クラスタを構成するノードがネットワーク障害等によって分断された場合、各ノードは通信可能な他ノードを調べ、過半数を超えている場合は動作を継続し、下回っている場合は動作を停止するためです。

しかし、本ドキュメントではノード1, ノード2の2ノードでクラスタを構成します。
これは分かりやすさを優先するためです。
とはいえ、ノード2に対する操作をノード3, ノード4...に適用することでより多い数のノードでクラスタを構成する場合でも本ドキュメントを利用できるよう記述してあります。

- Galera Clusterとは

    Galera Clusterは、MySQLとMariaDBのマルチマスタークラスタリングと同期レプリケーションを提供します。
    詳細は公式サイト( https://galeracluster.com/ )を参照してください。

## MariaDBをインストールする

まずは、クラスタを構成する各ノードにMariaDBをインストールします。

1. リポジトリを追加する

    MariaDBのパッケージリポジトリセットアップスクリプトを使います。  
    このスクリプトは、以下のタスクを行います。

    - /etc/yum.repos.d/mariadb.repo へのリポジトリ設定ファイル作成
    - MariaDBソフトウェアパッケージの署名検証に使用するGPG公開鍵をdownloads.mariadb.comからインポート

    スクリプト実行時に、`--mariadb-server-version`を使用することで、バージョンを指定します。今回は、本ドキュメント作成時点で最新の10.11を指定しています。

    ```
    $ sudo curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version="mariadb-10.11"
    # [info] Checking for script prerequisites.# [info] MariaDB Server version 10.11 is valid
    # [info] Repository file successfully written to /etc/yum.repos.d/mariadb.repo
    # [info] Adding trusted package signing keys...
    /etc/pki/rpm-gpg /home/rocky
    /home/rocky
    # [info] Successfully added trusted package signing keys
    # [info] Cleaning package cache...
    41 files removed
    ```

    作成されたリポジトリ設定ファイルは以下です。

    ```
    $ cat /etc/yum.repos.d/mariadb.repo
    
    [mariadb-main]
    name = MariaDB Server
    baseurl = https://dlm.mariadb.com/repo/mariadb-server/10.11/yum/rhel/8/x86_64
    gpgkey = file:///etc/pki/rpm-gpg/MariaDB-Server-GPG-KEY
    gpgcheck = 1
    enabled = 1
    module_hotfixes = 1
    
    
    [mariadb-maxscale]
    # To use the latest stable release of MaxScale, use "latest" as the version
    # To use the latest beta (or stable if no current beta) release of MaxScale, use "beta" as the version
    name = MariaDB MaxScale
    baseurl = https://dlm.mariadb.com/repo/maxscale/latest/yum/rhel/8/x86_64
    gpgkey = file:///etc/pki/rpm-gpg/MariaDB-MaxScale-GPG-KEY
    gpgcheck = 1
    enabled = 1
    
    
    [mariadb-tools]
    name = MariaDB Tools
    baseurl = https://downloads.mariadb.com/Tools/rhel/8/x86_64
    gpgkey = file:///etc/pki/rpm-gpg/MariaDB-Enterprise-GPG-KEY
    gpgcheck = 1
    enabled = 1
    ```

2. パッケージをインストールする

    リポジトリ追加後、パッケージをインストールします。
    
    ```
    $ sudo dnf install -y MariaDB-server MariaDB-client MariaDB-backup
    << 中略  >>
    Complete!
    ```
    

## データベースの設定

MariaDBをインストールできたら、初期設定を行なっていきます。

### サービスの自動起動が無効になっていることを確認する

Galera Clusterを構成する場合、OS起動時にDBが自動起動すると困る場合があるので、自動起動しない設定になっていることを確認しておきます。

このセクションはクラスタを構成する全てのノードで行います。

```
$ sudo systemctl is-enabled mariadb
disabled
```

`systemctl is-enabled`の結果が`disabled`であればOKです。

### ログ出力設定

MariaDBはインストール直後の状態ではログが出力されないので、ログを出力するよう設定します。
本ドキュメントでは`/var/log/mariadb`に出力することにします。

このセクションはクラスタを構成する全てのノードで行います。

1. ログの出力先ディレクトリを作成する

    ```
    $ sudo install -d -m 0755 -o mysql -g mysql /var/log/mariadb
    ```

2. ログの出力先を設定する

    `/etc/my.cnf.d/server.cnf`をエディタで開いて、`[mysqld]`セクションに、以下の行を追加します。

    ```
    log-error=/var/log/mariadb/mariadb.log
    ```

### rootアカウントを設定する

mariadbをインストールすると、rootアカウントが作成されますが、パスワードが設定されていないので、パスワードを設定します。

このセクションは、クラスタを構成するノードの最初の1台になるノード(本ドキュメントではノード1)のみで行います。

1.  MariaDBを開始する

    アカウントを設定するにはmariadbが動作していなければならないので、まずはmariadbを開始します。

    ```
    $ sudo systemctl start mariadb
    ```

 1. MariaDBサーバへrootユーザで接続する
 
     ```
     $ sudo mysql
     ```
 
     MariaDBサーバへの接続はmysqlコマンドを使用します。
     mysqlコマンドは`-u`オプションでログインするユーザを指定しますが、`-u`オプションを設定しない場合はLinuxの現在のユーザー名をユーザとして使用します。
     (`sudo mysql`とした場合、ユーザはrootになります。) 
     ログインに成功すると、`MariaDB >`プロンプトが表示されます。

1. rootにパスワードを設定する
    
    `ALTER USER`ステートメントを使って、パスワードを設定します。

     ```
     > ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';
     ```

     new_passwordの箇所を実際に設定したいパスワードにしてください。
    
1. MariaDBサーバから切断する

    パスワードを設定したら、一度、mariadbサーバから切断します。

   ```
   > exit;
   ```
    
1. もう一度MariaDBサーバへ接続し、パスワードが設定できていることを確認する

    もう一度mariadbサーバへ接続し、パスワードが設定できていることを確認します。今回は`-p`オプションをつけます。`Enter password:`というプロンプトが表示されたら、先ほど設定したパスワードを入力します。

    ```
    $ sudo mysql -p
    Enter password: ********
    ```

    `********`の箇所に、実際に設定したパスワードを入力します。
    ログインに成功すれば、`MariaDB >`プロンプトが表示されます。
    
1. MariaDBサーバから切断する

    パスワードを設定できていることが確認できたら、mariadbサーバから切断します。

   ```
   > exit;
   ```
    
1. (オプション) .my.cnf を作成する

    セキュリティポリシー上許されるのであれば、以下のファイルを`/root/.my.cnf`に作成します。
    (`********`の箇所は実際に設定したパスワードを記述してください。)
    このファイルを作成しておくと、`sudo mysql`でパスワードを入力せずにログインできるようになります。

    ```
    [mysql]
    user=root
    host=localhost
    password='********'
    socket=/var/lib/mysql/mysql.sock
    
    [client]
    user=root
    host=localhost
    password='********'
    socket=/var/lib/mysql/mysql.sock
    
    [mysqldump]
    user=root
    host=localhost
    password='********'
    socket=/var/lib/mysql/mysql.sock
    
    [mysqladmin]
    user=root
    host=localhost
    password='********'
    socket=/var/lib/mysql/mysql.sock
    
    [mysqlcheck]
    user=root
    host=localhost
    password='********'
    socket=/var/lib/mysql/mysql.sock
    ```

### 匿名ユーザを削除する

MariaDBをインストールすると、匿名のユーザが作成されているので、削除しておきます。

このセクションは、クラスタを構成するノードの最初の1台になるノード(本ドキュメントではノード1)のみで行います。

1. ユーザを確認する

    mysqlコマンドの`-e`オプションにSQL分を渡すことで、クエリーを実行することができます。今回はそれを利用して、現在のユーザ一覧を出力します。
    (`-e`オプションを使わず、一度ログインしてから実行しても構いません。)
    (`/root/.my.cnf`を設定している場合は`-p`オプションは不要です。`-p`オプションをつけると`/root/.my.cnf`の設定に関わらずパスワードを要求されます。)

   ```
   $ sudo mysql -p -e "select user,host,password from mysql.user;"
    Enter password:
    +-------------+------------------------+-------------------------------------------+
    | User        | Host                   | Password                                  |
    +-------------+------------------------+-------------------------------------------+
    | mariadb.sys | localhost              |                                           |
    | root        | localhost              | *0913BF2E2CE20CE21BFB1961AF124D4920458E5F |
    | mysql       | localhost              | invalid                                   |
    | PUBLIC      |                        |                                           |
    |             | localhost              |                                           |
    |             | hostname               |                                           |
    +-------------+------------------------+-------------------------------------------+
    ```

    `User`の箇所が空になっているレコードが匿名ユーザです。
    `Host`がhostnameになっている箇所は、実際はインストールされているマシンのhostnameが設定されています。

1. 匿名ユーザを削除する

    以下のようにクエリーを実行して、匿名ユーザを削除します。

    ```
    $ sudo mysql -p -e "delete from mysql.user where user = '';"
    Enter password:
    ```

1. 結果を確認しておく

    匿名ユーザが削除されていることを確認しておきます。

    ```
    $ sudo mysql -p -e "select user,host,password from mysql.user;"
    Enter password:
    +-------------+-----------+-------------------------------------------+
    | User        | Host      | Password                                  |
    +-------------+-----------+-------------------------------------------+
    | mariadb.sys | localhost |                                           |
    | root        | localhost | *0913BF2E2CE20CE21BFB1961AF124D4920458E5F |
    | mysql       | localhost | invalid                                   |
    | PUBLIC      |           |                                           |
    +-------------+-----------+-------------------------------------------+
    ```

1.  MariaDBを停止する

    アカウントを設定が終わったら、MariaDBを停止しておきます。

    ```
    $ sudo systemctl stop mariadb
    ```

###  wsrepオプションを設定する

`/etc/my.cnf.d/server.cnf`ファイルのwsrepオプションを設定します。
("wsrep"は"Write Set REPlication"の略で、Galera Clusterにおいてレプリケーションやデータ同期のために使用されるプロトコルおよびAPIです。)

このセクションはクラスタを構成する全てのノードで行います。

- wsrep_on

    レプリケーションを行うかの設定です。コメントアウトされているので、アンコメントします。

    ```
    [galera]
    # Mandatory settings
    wsrep_on=ON
    ```

- wsrep_provider

    レプリケーションプラグインのパスを設定します。コメントアウトされているので、アンコメントした上で、パスを設定します。
    Rocky8にMariaDB10.11をインストールした場合のパスは`/usr/lib64/galera-4/libgalera_smm.so`です。
    (他のOS, バージョンの場合はfindコマンドを使うなどの方法で、`libgalera_smm.so`を探してください。」)

    ```
    [galera]
    # Mandatory settings
    wsrep_on=ON
    wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so
    ```

- binlog_format

    バイナリログのフォーマットです。コメントアウトされているので、アンコメントします。

    ```
    binlog_format=row
    ```

- default_storage_engine

    デフォルトのストレージエンジンです。コメントアウトされているので、アンコメントします。

    ```
    default_storage_engine=InnoDB
    ```

- innodb_autoinc_lock_mode

    自動インクリメント値を生成するためのInnoDB ロックモードの設定です。
    コメントアウトされているので、アンコメントします。
    **この設定は必ず2** でなければなりません。

    ```
    innodb_autoinc_lock_mode=2
    ```

### wsrep_cluster設定

`/etc/my.cnf.d/server.cnf`ファイルにクラスタ関連のオプションを設定します。

このセクションはクラスタを構成する全てのノードで行います。

- wsrep_cluster_name

    クラスタの名前を設定します。
    ノードがクラスタに接続する際にこの値を照合し、一致した場合のみ接続します。
    そのため、**クラスタを構成する全てのノードで同じ値**を設定する必要があります。
    コメントアウトされているので、アンコメントした上で値を設定します。
    本ドキュメントではクラスタ名を仮に`sample_cluster`とします。

    ```
    wsrep_cluster_name=sample_cluster
    ```

- wsrep_node_name

    ノードの論理名を設定します。
    この設定は**ノードごとに異なる値**を設定する必要があります。
    本ドキュメントではノード1のノード名を仮に`node1`、ノード2のノード名を仮に`node2`とします。

    - ノード1
        ```
        wsrep_node_name=node1
        ```
    - ノード2
        ```
        wsrep_node_name=node2
        ```

- wsrep_node_address

    ノードのIPアドレスを設定します。
    本ドキュメントではノード1のIPアドレスを仮に`192.168.1.1`、ノード1のIPアドレスを仮に`192.168.1.2`とします。

    - ノード1
        ```
        wsrep_node_address=192.168.1.1  # <= 実際の値を設定してください
        ```
    - ノード2
        ```
        wsrep_node_address=192.168.1.2  # <= 実際の値を設定してください
        ```

## ファイアウォールの設定

クラスタを構成する各ノードが相互に通信できるように、以下のポートを解放してください。
環境により必要な設定が異なるので、本ドキュメントでは設定方法は省略します。

- 3306  
    - 通常、MySQL/MariaDBはポート3306を使用します。このポートは、クライアントアプリケーションがデータベースに接続するために必要です。
- 4444
    - SSTのために使用します。SST（State Snapshot Transfer）は、クラスタ内のノード間でデータの同期を行うプロセスの一種です。SSTは、新しく追加されたノードや、オフライン状態で大量のデータ変更があった場合に、そのノードが他のノードと完全なデータの同期を行うために使用されます。
- 4567
    - クラスタ内のレプリケーションとクラスタ通信に使用します。
- 4568
    - ISTのために使用します。IST（Incremental State Transfer）は、クラスタ内のノード間でデータの同期を行うプロセスの一種です。ISTは、ノードが一時的にオフライン状態になった後、再びオンラインになった際に、そのノードがオフライン時に失ったデータを迅速かつ効率的に同期するために使用されます。


## GaleraClusterの初期化

データベースの設定が終わったら、クラスタの初期化を行います。
このセクションは、クラスタを構成するノードの最初の1台になるノード(本ドキュメントではノード1)のみで行います。

### クラスタを構成するノードとバックエンドスキーマを設定する

`/etc/my.cnf.d/server.cnf`ファイルにオプションを設定します。

- wsrep_cluster_address

    ノードがクラスタに接続する際のバックエンドスキーマとIPアドレスを設定します。
    コメントアウトされているので、アンコメントした上で値を設定します。
    サポートされるバックエンドスキーマは`gcomm`のみとなります。
    最初に構成する際には、ノード1のIPアドレスのみ記述しておきます。

    ```
    wsrep_cluster_address=gcomm://192.168.1.1
    ```

### クラスタを起動する

以下のコマンドを実行して新しいクラスタを起動します。
`sudo systemctl start mariadb`でmariadbを起動すると新しいクラスタにならないので、必ず`galera_new_cluster`コマンドで起動します。

```
$ sudo galera_new_cluster mariadb
```

クラスタを起動したら、`systemctl status mariadb`でステータスを確認します。
`Active`の項目が｀active (running)`になっていれば、起動に成功しています。

```
$ sudo systemctl status mariadb
● mariadb.service - MariaDB 10.11.2 database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; disabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/mariadb.service.d
           └─migrated-from-my.cnf-settings.conf
   Active: active (running) since Sat 2023-05-06 07:26:56 UTC; 3min 31s ago
     Docs: man:mariadbd(8)
           https://mariadb.com/kb/en/library/systemd/
  Process: 863935 ExecStartPost=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
  Process: 863865 ExecStartPre=/bin/sh -c [ ! -e /usr/bin/galera_recovery ] && VAR= ||   VAR=`cd /usr/bin/..; /usr/bin/galera_recovery`; [ $? -eq 0 ]   && systemctl set-environment _WSREP_START_POSITION=$VAR >
  Process: 863860 ExecStartPre=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0/SUCCESS)
 Main PID: 863920 (mariadbd)
   Status: "Taking your SQL requests now..."
    Tasks: 13 (limit: 822260)
   Memory: 218.6M
   CGroup: /system.slice/mariadb.service
           └─863920 /usr/sbin/mariadbd --wsrep-new-cluster --wsrep_start_position=00000000-0000-0000-0000-000000000000:-1
```

### ノードのステータスを確認する

mysqlコマンドでクエリーを実行し、ステータスを確認します。
(/root/.my.cnfを設定している場合は-pオプションは不要です。-pオプションをつけると/root/.my.cnfの設定に関わらずパスワードを要求されます。)

- wsrep_cluster_status  
    `wsrep_cluster_status`が`Primary`になっていれば、ノードはクラスタに接続し、同期されている、つまり正常であることを表しています。

    ```
    sudo mysql -p -e "show status;" | grep cluster_status
    Enter password:
    wsrep_cluster_status	Primary
    ```

- wsrep_cluster_size  
    `wsrep_cluster_size`はクラスタ内のノード数を表しています。この段階では1のはずです。

    ```
    sudo mysql -p -e "show status;" | grep cluster_size
    Enter password:
    wsrep_cluster_size	1
    ```

## ノードの追加

クラスタを起動したら、ノードの追加を行います。
このセクションは、クラスタを構成するノードの2台目以降のノード(本ドキュメントではノード2)で行います。

### クラスタを構成するノードとバックエンドスキーマの設定

`/etc/my.cnf.d/server.cnf`ファイルにオプションを設定します。

- wsrep_cluster_address

    以下のように、クラスタの既存ノードのIPアドレスに加えて、追加するノードのIPアドレスを設定します。
    それぞれのIPアドレスは`,`(カンマ)で区切ります。

    ```
    wsrep_cluster_address=gcomm://192.168.1.1,192.168.1.2
    ```

### MariaDBを開始する

`systemctl start mariadb`でMariaDBを開始します。
クラスタを起動したときに使用した`galera_new_cluster`を **実行してはいけません。**
`galera_new_cluster`を実行すると既存のクラスタとは別のクラスタになってしまうため、必ず`systemctl start mariadb`でMariaDBを開始します。

```
$ sudo systemctl start mariadb
```

MariaDBを起動したら、`systemctl status mariadb`でステータスを確認します。
`Active`の項目が｀active (running)`になっていれば、起動に成功しています。

```
$ sudo systemctl status mariadb
● mariadb.service - MariaDB 10.11.2 database server
   Loaded: loaded (/usr/lib/systemd/system/mariadb.service; disabled; vendor preset: disabled)
  Drop-In: /etc/systemd/system/mariadb.service.d
           └─migrated-from-my.cnf-settings.conf
   Active: active (running) since Sat 2023-05-06 08:38:06 UTC; 15s ago
     Docs: man:mariadbd(8)
           https://mariadb.com/kb/en/library/systemd/
  Process: 24024 ExecStartPost=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=>
  Process: 23759 ExecStartPre=/bin/sh -c [ ! -e /usr/bin/galera_recovery ] && VAR= ||   VAR=`cd /usr/bin/..; /usr>
  Process: 23757 ExecStartPre=/bin/sh -c systemctl unset-environment _WSREP_START_POSITION (code=exited, status=0>
 Main PID: 23814 (mariadbd)
   Status: "Taking your SQL requests now..."
    Tasks: 19 (limit: 822260)
   Memory: 206.6M
   CGroup: /system.slice/mariadb.service
           └─23814 /usr/sbin/mariadbd --wsrep_start_position=00000000-0000-0000-0000-000000000000:-1
```

もし、`Active`の項目が｀active (running)`にならない場合はログを確認し、問題を特定してください。
追加するノードのログだけでなく、クラスタの既存ノードのログも確認してください。
(そちらに問題解決のヒントがある場合もあります。)

### ノードのステータスを確認する

追加したノードにて、mysqlコマンドでクエリーを実行し、ステータスを確認します。
このとき入力するパスワードは、ノード1で設定したMariaDBのrootのパスワードです。
クラスタに正しく参加し同期ができていれば、MariaDBのユーザ設定が入っている`mysql`データベースも同期されるため、同じユーザとパスワードでログインすることになります。
(ノード1で`/root/.my.cnf`を作成している場合は、追加ノードにコピーしておくと便利です。)


- wsrep_cluster_status  
    `wsrep_cluster_status`が`Primary`になっていれば、ノードはクラスタに接続し、同期されている、つまり正常であることを表しています。

    ```
    $ sudo mysql -p -e "show status;" | grep cluster_status
    Enter password:
    wsrep_cluster_status	Primary
    ```

- wsrep_cluster_size  
    `wsrep_cluster_size`はクラスタ内のノード数を表しています。この段階では2のはずです。

    ```
    $ sudo mysql -p -e "show status;" | grep cluster_size
    Enter password:
    wsrep_cluster_size	2
    ```

### 既存ノードにクラスタ構成ノードの情報を追加する

クラスタにすでに参加しているノードのwsrep_cluster_addressオプションに、追加したノードの情報を加えます。
もし、この変更を怠ると、既存ノードがなんらかの理由でクラスタから切断された場合、自動的にクラスタに再接続することができません。

ノード1の`/etc/my.cnf.d/server.cnf`ファイルは、この段階では以下のようになっているはずです。

```
wsrep_cluster_address=gcomm://192.168.1.1
```

これを追加したノードと同じく、以下のように変更します。

```
wsrep_cluster_address=gcomm://192.168.1.1,192.168.1.2
```

設定を変更したら、MariaDBを再起動します。
(このとき、ノード2で`sudo tail -f /var/log/mariadb/mariadb.log`しておくと、ノードがクラスタから切断され再接続する際に出力されるログを観測することができます。)

```
sudo systemctl restart mariadb
```

コマンドが終わったら、`mysql`コマンドを使って`wsrep_cluster_status`と`wsrep_cluster_size`を確認してください。ノードを追加した時と同じ値になっていれば、クラスタに再接続できています。

以上で、GaleraClusterの構築は完了です。
