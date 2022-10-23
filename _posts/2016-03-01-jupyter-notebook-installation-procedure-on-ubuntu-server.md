---
toc: true
layout: post
description: AWS EC2のUbuntu Server 14.04 LTSへjupyter notebookをインストールし起動するための手順です
categories: [jupyter,]
title: Ubuntu Serverへのjupyter notebookインストール手順
---

# Ubuntu Serverへのjupyter notebookインストール手順


これは、AWS EC2のUbuntu Server 14.04 LTSへjupyter notebookをインストールし起動するための手順です。(AWSの設定手順については、この記事では触れません。)

## 前提条件

- EC2インスタンスは作成済みのこと
- セキュリティグループのインバウンドルールに以下を追加しておくこと
 - タイプ:カスタムTCPルール, ポート範囲:8888, 送信元は自身の環境に合わせて適切に設定すること 

なお、この手順は以下の環境で動作確認しています。

- リージョン: 東京
- AMI: Ubuntu Server 14.04 LTS (HVM), SSD Volume Type - ami-936d9d93
- インスタンスタイプ: t2.micro

## 手順

[Installation — Jupyter Documentation 4.1.0b1 documentation](http://jupyter.readthedocs.org/en/latest/install.html#new-users-new-to-python-and-jupyter)ではanacondaの利用を勧めているので、pyenvを使ってpythonとanacondaをインストールし、anacondaでjupyter notebookをインストールします。

### EC2インスタンスへのsshアクセス

EC2インスタンスにはsshでアクセスします。

```bash
$ ssh -i aws-key ubuntu@public-ip
```

### git インストール

pyenvのインストールにgitが必要になるのでインストールします。

```bash
$ sudo apt-get install -y git
```

### pyenv インストール

pyenvの[Installation-Basic GitHub Checkout](https://github.com/yyuu/pyenv#basic-github-checkout)の1~4の手順でインストールします。
ただし、basic-github-checkoutの手順そのままでは上手く行きません。幾つか手順を変える必要があります。

1. Ubuntu 14.04では.bash_profileが.profileに変わっているので、そこを読み替える [^1]
2. 手順4はシェルのリスタートなので`exec`に`-l`オプションが不足している [^2]

実際に叩くコマンドは以下になります。

```bash
$ git clone https://github.com/yyuu/pyenv.git ~/.pyenv
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.profile
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.profile
$ echo 'eval "$(pyenv init -)"' >> ~/.profile
$ exec -l $SHELL
```

### anacondaインストール

pyenvででpythonとanacondaをインストールします。
まず、pyenvでインストール出来るanacondaのリストを確認しておきます。

```bash
$ pyenv install -l | grep anaconda
  anaconda-1.4.0
  anaconda-1.5.0
  anaconda-1.5.1
  anaconda-1.6.0
  anaconda-1.6.1
  anaconda-1.7.0
  anaconda-1.8.0
  anaconda-1.9.0
  anaconda-1.9.1
  anaconda-1.9.2
  anaconda-2.0.0
  anaconda-2.0.1
  anaconda-2.1.0
  anaconda-2.2.0
  anaconda-2.3.0
  anaconda-2.4.0
  anaconda2-2.4.0
  anaconda2-2.4.1
  anaconda3-2.0.0
  anaconda3-2.0.1
  anaconda3-2.1.0
  anaconda3-2.2.0
  anaconda3-2.3.0
  anaconda3-2.4.0
  anaconda3-2.4.1
```

このとき表示されたバージョンをインストールするとanacondaに加えてpythonもインストールされます。`anaconda-*`及び`anaconda2-`がPython2系、`anaconda3-*`がPython3系です。[^3]

今回の手順ではanaconda2.4.1をPython3系で使うことにします。
なお、この時ダウンロードされる`Anaconda3-2.4.1-Linux-x86_64.sh`は270MBほどのサイズです。EC2ではあまり問題にならないと思いますが、回線の細い環境でインストールする場合は時間がかかる場合がありあす。

```bash
$ pyenv install anaconda3-2.4.1
Downloading Anaconda3-2.4.1-Linux-x86_64.sh...
-> http://repo.continuum.io/archive/Anaconda3-2.4.1-Linux-x86_64.sh
Installing Anaconda3-2.4.1-Linux-x86_64...
Installed Anaconda3-2.4.1-Linux-x86_64 to /home/ubuntu/.pyenv/versions/anaconda3-2.4.1
```

インストールが終わったらanaconda3-2.4.1をpyenvのグローバルにします。

```bash
$ pyenv global anaconda3-2.4.1 
```

なお、グローバルに設定したPythonとanacondaのバージョンは`python --version`で確認できます。

```bash
$ python --version
Python 3.5.1 :: Anaconda 2.4.1 (64-bit)
```


### jupyterのインストール
anacondaをインストールできたらいよいよjupyter notebookのインストールです。
コマンドは`conda install -y jupyter`だけで大丈夫です。

```bash
$ conda install -y jupyter
Fetching package metadata: ....
Solving package specifications: ..................................................................
Package plan for installation in environment /home/ubuntu/.pyenv/versions/anaconda3-2.4.1:

The following packages will be downloaded:

    package                    |            build
    ---------------------------|-----------------
    openssl-1.0.2e             |                0         3.2 MB  defaults
    sqlite-3.9.2               |                0         3.9 MB  defaults
    decorator-4.0.6            |           py35_0           6 KB  defaults
    pyzmq-15.2.0               |           py35_0         792 KB  defaults
    requests-2.9.1             |           py35_0         648 KB  defaults
    setuptools-19.2            |           py35_0         348 KB  defaults
    conda-3.19.0               |           py35_0         180 KB  defaults
    ipython-4.0.2              |           py35_0         970 KB  defaults
    ipykernel-4.2.2            |           py35_0         115 KB  defaults
    nbconvert-4.1.0            |           py35_0         275 KB  defaults
    notebook-4.1.0             |           py35_0         4.4 MB  defaults
    ipywidgets-4.1.1           |           py35_0          99 KB  defaults
    ------------------------------------------------------------
                                           Total:        14.8 MB

The following packages will be UPDATED:

    conda:      3.18.8-py35_0 defaults --> 3.19.0-py35_0 defaults
    decorator:  4.0.4-py35_0  defaults --> 4.0.6-py35_0  defaults
    ipykernel:  4.1.1-py35_0  defaults --> 4.2.2-py35_0  defaults
    ipython:    4.0.1-py35_0  defaults --> 4.0.2-py35_0  defaults
    ipywidgets: 4.1.0-py35_0  defaults --> 4.1.1-py35_0  defaults
    nbconvert:  4.0.0-py35_0  defaults --> 4.1.0-py35_0  defaults
    notebook:   4.0.6-py35_0  defaults --> 4.1.0-py35_0  defaults
    openssl:    1.0.2d-0      defaults --> 1.0.2e-0      defaults
    pyzmq:      14.7.0-py35_1 defaults --> 15.2.0-py35_0 defaults
    requests:   2.8.1-py35_0  defaults --> 2.9.1-py35_0  defaults
    setuptools: 18.5-py35_0   defaults --> 19.2-py35_0   defaults
    sqlite:     3.8.4.1-1     defaults --> 3.9.2-0       defaults

Fetching packages ...
openssl-1.0.2e 100% |################################| Time: 0:00:01   2.16 MB/s
sqlite-3.9.2-0 100% |################################| Time: 0:00:01   2.43 MB/s
decorator-4.0. 100% |################################| Time: 0:00:00   7.56 MB/s
pyzmq-15.2.0-p 100% |################################| Time: 0:00:01 770.63 kB/s
requests-2.9.1 100% |################################| Time: 0:00:01 552.84 kB/s
setuptools-19. 100% |################################| Time: 0:00:00 467.58 kB/s
conda-3.19.0-p 100% |################################| Time: 0:00:00 284.50 kB/s
ipython-4.0.2- 100% |################################| Time: 0:00:01 826.19 kB/s
ipykernel-4.2. 100% |################################| Time: 0:00:00 245.78 kB/s
nbconvert-4.1. 100% |################################| Time: 0:00:00 369.84 kB/s
notebook-4.1.0 100% |################################| Time: 0:00:01   2.37 MB/s
ipywidgets-4.1 100% |################################| Time: 0:00:00 229.46 kB/s
Extracting packages ...
[      COMPLETE      ]|###################################################| 100%
Unlinking packages ...
[      COMPLETE      ]|###################################################| 100%
Linking packages ...
[      COMPLETE      ]|###################################################| 100%
```

### jupyterの設定

jupyter notebookをEC2で使うには、いくつかの設定を変更しなければなりません。
まずは、コンフィグファイルを作成します。[^4]

```bash
$ jupyter notebook --generate-config
Writing default config to: /home/ubuntu/.jupyter/jupyter_notebook_config.py
```

出力のとおり`/home/ubuntu/.jupyter/jupyter_notebook_config.py`にコンフィグファイルが作られるので、以下の項目を変更します。

```Python:jupyter_notebook_config.py
c.NotebookApp.ip = '*' # ローカルホスト以外からもアクセス可能にする
c.NotebookApp.open_browser = False # ブラウザが自動で開かないようにする
```

なお、各項目の変更前の値は以下です。

```Python:jupyter_notebook_config.py
# c.NotebookApp.ip = 'localhost'
# c.NotebookApp.open_browser = True
```

sedで処理する場合は次のように。

```Bash
$ sed -ri "s/# c.NotebookApp.ip = 'localhost'/c.NotebookApp.ip = '*'/g" /home/ubuntu/.jupyter/jupyter_notebook_config.py
$ sed -ri "s/# c.NotebookApp.open_browser = True/c.NotebookApp.open_browser = False/g" /home/ubuntu/.jupyter/jupyter_notebook_config.py
```

なお、コンフィグファイルの各項目について知りたい場合は、[Config file and command line options — Jupyter Notebook 4.1.0rc1 documentation](http://jupyter-notebook.readthedocs.org/en/latest/config.html#config-file-and-command-line-options)を参照してください。


### jupyter notebookの開始

以上でjupyter notebookのインストールと設定が出来ました。
コンソールで`jupyter notebook`と入力するとjupyter notebookがスタートします。

```Bash
$ jupyter notebook
```

ブラウザでEC2インスタンスのpublic_ip_address:8888にアクセスし、jupyter notebookのトップページが表示されればインストール・設定は完了です。


## オプション

### パスワードの設定

これまでの手順でjupyter notebookを立ち上げてアクセスすることは可能ですが、このまま立ち上げると誰でもアクセス可能(=サーバ上でPythonコードを実行し放題)なので、パスワードを設定します。[^5]

まずサーバのコンソールでipythonを立ち上げパスワードのハッシュを得ます。

```Bash
$ ipython
In [1]: from notebook.auth import passwd

In [2]: passwd()
Enter password:     # パスワードを入力
Verify password:    # パスワードを再入力
Out[2]: 'sha1:371793e28601:458c6bb9fc9371410b37cfc0d5833bfb54a26d7c' #ハッシュ値が出力されるのでこれをコピーする

In [3]: quit()    # ipythonを終了
```

次に`/home/ubuntu/.jupyter/jupyter_notebook_config.py`にパスワードのハッシュを書き込みます。
`c.NotebookApp.password`はコメントアウトされているので、`#`も忘れずに消します。

```Python:jupyter_notebook_config.py
c.NotebookApp.password = 'sha1:371793e28601:458c6bb9fc9371410b37cfc0d5833bfb54a26d7c'
```

### 暗号化通信の設定

パスワードを設定してもこのままでは平文で送信されるため、パスワードもjupyter notebookとブラウザ間の通信も盗聴される恐れがあります。

今回は **とりあえず** 、[Using SSL for encrypted communication](http://jupyter-notebook.readthedocs.org/en/latest/public_server.html#using-ssl-for-encrypted-communication)の内容で自己署名証明書を設定して通信経路の暗号化を行ってみます。

```Bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:1024 -keyout .jupyter/mykey.key -out .jupyter/mycert.pem
```

作成したcertfileとkeyfileを使用するようコンフィグを設定します。

```Python:jupyter_notebook_config.py
c.NotebookApp.certfile = '/home/ubuntu/.jupyter/mycert.pem'
c.NotebookApp.keyfile = '/home/ubuntu/.jupyter/mykey.key'
```


jupyter notebookをスタートしたら、httpsでのアクセスのみを受け付けるようになります。


[^1]: [Ubuntu14.04 - Ubuntu 14.04 の .bash_profile ファイル のファイル名 は、.profile に変わっていた件 - Qiita](http://qiita.com/HirofumiYashima/items/4b1db4f05fc11d160b4e)

[^2]: [shell - シェルの再起動 - Qiita](http://qiita.com/rentalname@github/items/14b32c0b50b277837528)

[^3]: [AnacondaでPythonの分析環境をまとめてインストール - TASK NOTES](http://www.task-notes.com/entry/20151116/1447642800)

[^4]: [Configuring Jupyter applications — Jupyter Documentation 4.1.0b1 documentation](http://jupyter.readthedocs.org/en/latest/config.html#python-config-files)

[^5]: [ipython notebookをリモートサーバ上で動かす。 - 忘れないようにメモっとく](http://akiniwa.hatenablog.jp/entry/2013/11/25/001805)

