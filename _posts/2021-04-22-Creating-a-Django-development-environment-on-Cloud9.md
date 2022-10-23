---
toc: true
layout: post
description: [Cloud9でPython開発環境を作る](https://taka4ma.github.io/contents/0007/0007.html)でCloud9に作ったPython開発環境にDjangoの開発環境を作っていく
categories: [cloud9, django]
title: Cloud9にDjango開発環境を作る
---

# Cloud9にDjango開発環境を作る

この記事では、[Cloud9でPython開発環境を作る](https://taka4ma.github.io/contents/0007/0007.html)でCloud9に作ったPython開発環境にDjangoの開発環境を作っていく。

## 仮想環境を起動する

[Cloud9でPython開発環境を作る](https://taka4ma.github.io/contents/0007/0007.html) で `~/environment` ディレクトリに`myenv`という名前の仮想環境を作成した。その仮想環境を起動する

```
cd ~/environment
source myenv/bin/activate
```

## Djangoをインストールする

ここからは、[Djangoのインストール · HonKit](https://tutorial.djangogirls.org/ja/django_installation/)を参考に進めていく。


最初に、requirments.txtを作成する。

// requirments.txtとは、pipでインストールするライブラリのリストが書かれたファイルのこと。詳細は
[User Guide \- pip documentation v21\.0\.1](https://pip.pypa.io/en/stable/user_guide/#requirements-files)を参照。

```
$ cd ~/environments
$ echo Djangp~=2.2.4 > requirements.txt
```

一応、ファイルの内容を確認しておく。

```
$ cat requirements.txt 
Django~=2.2.4
```

requirements.txt ができたら、pipを使ってインストールする。

```
$ pip install -r requirements.txt 
Collecting Django~=2.2.4
  Downloading Django-2.2.20-py3-none-any.whl (7.5 MB)
     |████████████████████████████████| 7.5 MB 13.0 MB/s 
Collecting pytz
  Downloading pytz-2021.1-py2.py3-none-any.whl (510 kB)
     |████████████████████████████████| 510 kB 53.6 MB/s 
Collecting sqlparse>=0.2.2
  Downloading sqlparse-0.4.1-py3-none-any.whl (42 kB)
     |████████████████████████████████| 42 kB 2.3 MB/s 
Installing collected packages: pytz, sqlparse, Django
Successfully installed Django-2.2.20 pytz-2021.1 sqlparse-0.4.1
```

インストールが終わったら、`pip freeze`でインストールされたライブラリとバージョンを確認しておく。

```
$ pip freeze
Django==2.2.20
pytz==2021.1
sqlparse==0.4.1
```
## Djangoプロジェクトを作成する

ここからは[プロジェクトを作成しよう！ · HonKit](https://tutorial.djangogirls.org/ja/django_start_project/)を参考に進めていく。

以下のコマンドを実行し、"mysite"という名前のDjangoのプロジェクトを`~/environment`に作成する。

```
$ cd ~/environment
$ django-admin startproject mysite .
```

実行すると、以下のようなファイル構造が作成される。

```
$ tree -L 2
.
├── manage.py
├── myenv
│   ├── bin
│   ├── include
│   ├── lib
│   ├── lib64 -> lib
│   └── pyvenv.cfg
├── mysite
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── README.md
└── requirements.txt
```

// これはtreeコマンドで2階層表示した結果。  
// treeコマンドはAmazon Linuxにデフォルトでは入っていないので、インストールする場合は `sudo yum install -y tree` を実行する。

`manage.py`と`mysite`がスクリプトによって作られたファイル。   
myenvとrequirements.txtはこのセクションよりも前に作ってあったもの。  
README.mdはcloud9のファイル。

## 設定を変更する

作成したDjangoプロジェクトの`mysite/settings.py`をエディタで編集するが、まずは編集前のファイルをコピーしておく。

```
cp mysite/settings.py mysite/settings.py.bk
```

コピーをとったら、オリジナルファイルをエディタで開いて編集する。

- タイムゾーン

  ```mysite/settings.py
  TIME_ZONE = 'Asia/Tokyo'
  ```

- 言語コード

  ```
  LANGUAGE_CODE = 'ja'
  ```

- 静的ファイルのパス(STATIC_URLの次の行にSTATIC_ROOTを追加する)

  ```
  STATIC_URL = '/static/'
  STATIC_ROOT = os.path.join(BASE_DIR, 'static')
  ```

- アクセス許可ホスト
  ```
  ALLOWED_HOSTS = ['.amazonaws.com']
  ```

編集が終わったら、コピーしておいた編集前のファイルとの差分を確認する。以下のような結果になればOK。

```
$ diff mysite/settings.py.bk mysite/settings.py
28c28
< ALLOWED_HOSTS = []
---
> ALLOWED_HOSTS = ['.amazonaws.com']
106c106
< LANGUAGE_CODE = 'en-us'
---
> LANGUAGE_CODE = 'ja'
108c108
< TIME_ZONE = 'UTC'
---
> TIME_ZONE = 'Asia/Tokyo'
120a121
> STATIC_ROOT = os.path.join(BASE_DIR, 'static')
```

## DBをセットアップする

`mysite/settings.py`にはデフォルトのDB設定が書かれている。

```
$ grep DATABASES -A 6 mysite/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
```

今回は、この設定をそのまま使い、sqlite3をDBとして使用する。DBを作成するために、以下のコマンドを実行すると、

```
$ python manage.py migrate
Traceback (most recent call last):
  File "manage.py", line 21, in <module>
    main()
  File "manage.py", line 17, in main
    execute_from_command_line(sys.argv)
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/core/management/__init__.py", line 381, in execute_from_command_line
    utility.execute()
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/core/management/__init__.py", line 357, in execute
    django.setup()
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/__init__.py", line 24, in setup
    apps.populate(settings.INSTALLED_APPS)
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/apps/registry.py", line 114, in populate
    app_config.import_models()
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/apps/config.py", line 211, in import_models
    self.models_module = import_module(models_module_name)
  File "/usr/lib64/python3.7/importlib/__init__.py", line 127, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
  File "<frozen importlib._bootstrap>", line 1006, in _gcd_import
  File "<frozen importlib._bootstrap>", line 983, in _find_and_load
  File "<frozen importlib._bootstrap>", line 967, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 677, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 728, in exec_module
  File "<frozen importlib._bootstrap>", line 219, in _call_with_frames_removed
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/contrib/auth/models.py", line 2, in <module>
    from django.contrib.auth.base_user import AbstractBaseUser, BaseUserManager
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/contrib/auth/base_user.py", line 47, in <module>
    class AbstractBaseUser(models.Model):
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/db/models/base.py", line 117, in __new__
    new_class.add_to_class('_meta', Options(meta, app_label))
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/db/models/base.py", line 321, in add_to_class
    value.contribute_to_class(cls, name)
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/db/models/options.py", line 204, in contribute_to_class
    self.db_table = truncate_name(self.db_table, connection.ops.max_name_length())
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/db/__init__.py", line 28, in __getattr__
    return getattr(connections[DEFAULT_DB_ALIAS], item)
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/db/utils.py", line 201, in __getitem__
    backend = load_backend(db['ENGINE'])
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/db/utils.py", line 110, in load_backend
    return import_module('%s.base' % backend_name)
  File "/usr/lib64/python3.7/importlib/__init__.py", line 127, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/db/backends/sqlite3/base.py", line 66, in <module>
    check_sqlite_version()
  File "/home/ec2-user/environment/myenv/lib64/python3.7/site-packages/django/db/backends/sqlite3/base.py", line 63, in check_sqlite_version
    raise ImproperlyConfigured('SQLite 3.8.3 or later is required (found %s).' % Database.sqlite_version)
django.core.exceptions.ImproperlyConfigured: SQLite 3.8.3 or later is required (found 3.7.17).
```

エラーが出た。エラーメッセージによると、

> django.core.exceptions.ImproperlyConfigured: SQLite 3.8.3 or later is required (found 3.7.17).

DjangoがSQLite 3.8.3以降を必要としているのに対して、3.7.17がインストールされているためのようだ。

念の為確認しておく

```
$ python -c "import sqlite3;print(sqlite3.sqlite_version)"
3.7.17
```

確かに、pythonからは3.7.17のSQLiteが見えるようだ。

// このコマンドは、pythonのコマンドオプション `-c` を使ってコマンドとして渡したPythonコードを実行している。  参考:[1\. コマンドラインと環境 — Python 3\.9\.4 ドキュメント](https://docs.python.org/ja/3/using/cmdline.html)

システム上ではどうだろうか。pathが通っているsqlite3のバージョンを確認してみる。

```
$ which sqlite3
/usr/bin/sqlite3
$ sqlite3 --version
3.7.17 2013-05-20 00:56:22 118a3b35693b134d56ebd780123b7fd6f1497668
```

pathが通っているsqlite3は3.7.17だ。rootのPATHは一般ユーザと異なる場合があり、そこに別バージョンがインストールされている可能性がある。確認してみる。

```
$ sudo which sqlite3
/bin/sqlite3
$ /bin/sqlite3 --version
3.7.17 2013-05-20 00:56:22 118a3b35693b134d56ebd780123b7fd6f1497668
```

rootのpath上には `/bin/sqlite3` があるようだが、バージョンはこちらも3.7.17だ。

Djangoが要求するバージョンのSQLiteはデフォルトではインストールされていないようなので、最新バージョンのSQLiteをインストールすることにする。

この記事は、[Cloud9でPython開発環境を作る](https://taka4ma.github.io/contents/0007/0007.html)でCloud9に作ったPython開発環境を前提としているので、サーバはAmazon Linux2を前提としている。Amazon Linux2へ最新のSQLite3をインストールする方法は、[Amazon Linux2へ最新バージョンのSQLiteをインストールする](https://taka4ma.github.io/contents/0009.html) にまとめたので、そちらを参照して最新のSQLiteをインストールする。

最新のSQLiteをインストールできたら、改めてDBを作成する。

```
$ python manage.py migrate
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying sessions.0001_initial... OK
```

今度はエラーも出ず終了した。

## webサーバを起動する

DBのマイグレーションができたら、webサーバ(Django開発サーバ)を起動する。  

// 起動するサーバはあくまで開発用なので運用環境でしようしてはいけない。

`python manage.py runserver`で起動すると、8000ポートを使用するがCloud9では8000ポートは使えないようなので、8080で起動するよう引数を与える。

```
$ python manage.py runserver 8080
Watching for file changes with StatReloader
Performing system checks...

System check identified no issues (0 silenced).
May 04, 2021 - 21:50:22
Django version 2.2.20, using settings 'mysite.settings'
Starting development server at http://127.0.0.1:8080/
Quit the server with CONTROL-C.
```

Cloud9で開発サーバを起動するとウィンドウの右上に、

> Cloud9 Help  
> Your code running at  
> https://(a bunch of letters and numbers).vfs.cloud9.us-west-2.amazonaws.com

のように書かれたウィンドウが表示されるので、そのURLをクリックするとブラウザでWebサイトが開かれる。
ブラウザには、

> インストールは成功しました！おめでとうございます！

と書かれた、ロケットが離陸しているページが表示される。

以上で、Cloud9でのDjango開発環境の構築は終わり。  
最後に、Clou9のコンソールに戻り、Ctrl+C で開発サーバを停止しておく。
