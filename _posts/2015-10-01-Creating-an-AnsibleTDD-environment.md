# AnsibleTDD環境をtest-kitchen, serverspec, Dockerで作る(2015年10月版)

これはtest-kitchen, serverspec, DockerをつかってAnsibleのテスト駆動開発を行う環境を構築する手順です。
Ubuntu14,CentOS7でApache2をインストールし、サービスを起動し、ブート時のサービス自動起動を設定するベストプラクティス構成[^1]のAnsible playbookを例として取りあげます。

## この記事の目標

AnsibleのTDD環境として、test-kitchenで次のことが出来る環境を作ります。

 - Docker上にUbuntu14, CentOS7のコンテナを立ち上げ
 - 各コンテナをAnsibleでプロビジョニング
 - 各コンテナのプロビジョニング結果をServerspecで検証

## 前提条件等

この記事の前提条件は以下のとおりです。

- 作成するAnsible playbookの構成はベストプラクティスのディレクトリレイアウトに従う[^1]　
- 作業環境は
  - CentOS6.7(x86_64)
  - Rubyインストール済みのこと
  - bundlerインストール済みのこと
  - Dockerインストール済みのこと

なお、この記事の検証に使用した環境は以下のとおりです。

```bash:この記事の検証に使用した環境
$ cat /etc/redhat-release
CentOS release 6.7 (Final)
$ arch
x86_64
$ ruby -v
ruby 2.1.5p273 (2014-11-13 revision 48405) [x86_64-linux]
$ bundler -v
Bundler version 1.9.2
$ docker -v
Docker version 1.7.1, build 786b29d
```

(2015-12-04追記)
環境構築用のVagrantfileを作りました。
https://github.com/takasix/vagrant_kitchen_ansible

(元々別の用があって作ったものなので、バージョン指定が入っておらず、この記事の検証に使用した環境と全く同じ環境は構築されません。追記時点では記事の内容が全て動くことを確認済みです。)


## 本文

### プロジェクト用ディレクトリの作成

プロジェクト用のディレクトリを作成します。これ以降の手順はこのディレクトリをベースに行います。

```bash:プロジェクト用ディレクトリの作成
$ mkdir ~/ansibletdd
$ cd ~/ansibletdd
```

### test-kitchenインストール

プロジェクトディレクトリにGemfileを作成し、bundlerを使ってtest-kitchenをインストールします。

```rb:Gemfile
source 'https://rubygems.org'

gem 'test-kitchen'
```

書けたらbundle installします。

```bash:test-kitchenインストール
$ bundle install
Fetching gem metadata from https://rubygems.org/..........
Fetching version metadata from https://rubygems.org/...
Fetching dependency metadata from https://rubygems.org/..
Resolving dependencies...
Using mixlib-shellout 2.2.1
Using net-ssh 2.9.2
Using net-scp 1.2.1
Using safe_yaml 1.0.4
Using thor 0.19.1
Using test-kitchen 1.4.2
Using bundler 1.9.2
Bundle complete! 1 Gemfile dependency, 7 gems now installed.
Use `bundle show [gemname]` to see where a bundled gem is installed.
```

### test-kitchen初期化

`kitchen init`コマンドでtest-kitchenを初期化します。このとき--driver(または-D)オプションでドライバを、--provisioner(または-P)オプションでプロビジョナを指定できます。
今回は、テストを行う環境にDockerを使うのでドライバに"kitchen-docker"、プロビジョニングにはAnsibleを使うのでプロビジョナに"ansible_playbook"をそれぞれ指定します。

```bash:test-kitchenの初期化
$ bundle exec kitchen init --driver=kitchen-docker --provisioner=ansible_playbook
      create  .kitchen.yml
      create  chefignore
      create  test/integration/default
      append  Gemfile
You must run `bundle install' to fetch any new gems.
```

初期化を行うと.kitchen.ymlファイルといくつかのディレクトリが作成されます。また、Gemfileが変更されます。

```bash
$ cat .kitchen.yml
---
driver:
  name: docker

provisioner:
  name: ansible_playbook

platforms:
  - name: ubuntu-14.04
  - name: centos-7.1

suites:
  - name: default
    run_list:
    attributes:
$ 
$ tree
.
├── Gemfile
├── Gemfile.lock
├── chefignore
└── test
    └── integration
        └── default
$ 
$ cat Gemfile
source 'https://rubygems.org'

gem 'test-kitchen'
gem "kitchen-docker" 
```

### kitchen-ansibleとServerspecインストール

`kitchen init`によって変更されたGemfileにkitchen-ansibleとserverspecを加え、bundlerを使ってインストールします。

```rb:Gemfile
source 'https://rubygems.org'

gem 'test-kitchen'
gem "kitchen-docker"
gem 'kitchen-ansible'
gem 'serverspec'
```

変更したら、もう一度`bundle install`します。

```bash:kitchen-ansibleとServerspecインストール
$ bundle install
Resolving dependencies...
Using diff-lcs 1.2.5
Using multipart-post 2.0.0
Using faraday 0.9.2
Using highline 1.7.8
Using thor 0.19.1
Using librarian 0.1.2
Using librarian-ansible 1.0.6
Using mixlib-shellout 2.2.1
Using net-ssh 2.9.2
Using net-scp 1.2.1
Using safe_yaml 1.0.4
Using test-kitchen 1.4.2
Using kitchen-ansible 0.0.27
Using kitchen-docker 2.3.0
Using multi_json 1.11.2
Using net-telnet 0.1.1
Using rspec-support 3.3.0
Using rspec-core 3.3.2
Using rspec-expectations 3.3.1
Using rspec-mocks 3.3.2
Using rspec 3.3.0
Using rspec-its 1.2.0
Using sfl 2.2
Using specinfra 2.43.10
Using serverspec 2.24.1
Using bundler 1.9.2
Bundle complete! 4 Gemfile dependencies, 26 gems now installed.
Use `bundle show [gemname]` to see where a bundled gem is installed.
```

### .kitchen.ymlの設定

`kitchen init`で作成された.kitchen.ymlを編集しtest-kitchenの設定を行います。
(編集が終わった.kitchen.ymlはこのセクションの一番最後にあります。)

```yaml:初期状態の.kitchen.yml
---
driver:
  name: docker

provisioner:
  name: ansible_playbook

platforms:
  - name: ubuntu-14.04
  - name: centos-7.1

suites:
  - name: default
    run_list:
    attributes:
```

#### driver

driverは、変更する必要はありません。

```yaml:.kitchen.ymlのdriver
driver:
  name: docker
```

#### provisioner

provisionerにはプロビジョナーのオプションを以下のように設定します。
なお、ansible_playbookのオプションは[Provisioner Options](https://github.com/neillturner/kitchen-ansible/blob/master/provisioner_options.md)も参照してください。

```.kitchen.ymlのprovisioner
provisioner:
  name: ansible_playbook
  playbook: site.yml
  roles_path: ./roles
  group_vars_path: ./group_vars
  host_vars_path: ./host_vars
  filter_plugins: ./filter_plugins
  additional_copy_path:
    - webservers.yml
  hosts: webservers
  require_ansible_omnibus: true
  require_ruby_for_busser: true
```

1. playbook
    master playbook(site.yml)を.kitchen.ymlからの相対パスで設定します。

1. roles_path, group_vars_path, host_vars_path, filter_plugins
    それぞれのディレクトリを.kitchen.ymlからの相対パスで設定します。

1. additional_copy_path
    他にテスト環境へコピーするファイル・ディレクトリを.kitchen.ymlからの相対パスで設定します。この設定は配列なので複数の項目を設定できます。

    ベストプラクティス構成では、サーバー群(tier)ごとのPlay bookを作成し、site.ymlでサーバ群ごとのPlay bookをincludeすることで、site.ymlをインフラ全体の定義とします。[^2] しかし、kitchen-ansibleはplaybookに設定したymlをテスト環境へコピーしますが、その中でincludeしているymlまではコピーしてくれません。そこでサーバー群別Play bookもテスト環境へコピーされるようadditional_copy_pathに設定します。
    (rolesディレクトリにあるrole別のPlay bookはroles_pathの設定によってコピーされます。)

    今回は、Apacheをインストールしたいので、サーバー群Play bookはwebservers.ymlという名前で作ることにして、とりあえずadditional_copy_pathに書いておきます。

1. hosts
    hostsオプションにはテスト環境がどのホストグループに含まれるホストかを定義します。
    テスト環境にはhostsの設定を元に以下のようなinventryファイルが作成され、プロビジョニングに使用されます。

    ```text:/tmp/kitchen/hosts
    localhost ansible_connection=local
    [webservers]  # <= ここにhostsオプションが設定される
    localhost
    ```

    今回は、ホストグループもwebserversという名前で作ることにして、とりあえずhostsに書いておきます。

1. require_ansible_omnibus
    trueにすると、テスト環境へのAnsibleインストールを、`ansible_omnibus_url`オプション(デフォルトは　[https://raw.githubusercontent.com/neillturner/omnibus-ansible/master/ansible_install.sh](https://raw.githubusercontent.com/neillturner/omnibus-ansible/master/ansible_install.sh))で定義されたスクリプトで行います。

    (このオプションのデフォルトはfalseです。falseの場合、テスト環境へのAnsibleのインストールはyumまたはaptで行うのですが、記事執筆時点ではsyntax errorとなるため、オムニバスインストーラーを使用します。)

1. require_ruby_for_busser
    trueの場合、テスト環境でbusserの実行に使用するrubyをインストールします。(busserはtest-kitchenのテストフレームワークです。)

    このオプションのデフォルトはfalseで、その場合はChefをインストールし、それに同梱されているrubyでbusserを実行します。今回はAnsibleを使うためChefは不要、その上rubyよりもインストールに時間がかかるのでtrueにします。

#### platforms

platformsにはテスト環境を以下のように設定します。

```yaml:.kitchen.ymlのplatforms
platforms:
  - name: ubuntu-14.04
  - name: centos-7.1
    driver_config:
      privileged: true
      run_command: /sbin/init; sleep 3
```

本記事執筆時点のcentos:centos7イメージでは、systemctlを使用するためには特権モードかつ起動コマンドを/sbin/initにしなければならない[^3]ので、そのためのオプションを追加します。

#### verifier

.kitchen.ymlにverifierを追加しテストツールのオプションを以下のように設定します。

```yaml:.kitchen.ymlのverifier
verifier:
  ruby_bindir: '/usr/bin'
```

verifierの`ruby_bindir`にはテストツールの実行に使用するrubyがインストールされているディレクトリを設定します。
デフォルトは`/opt/chef/embedded/bin`で、これはChefに同梱されるrubyのディレクトリを指しています。
今回はprovisioner設定でChefを入れず、代わりにrubyをインストールするため、`ruby_bindir`にはそちらのパスを設定します。

#### suites

suitesにはテストスイートを以下のように設定します。

```.kitchen.ymlのsuites
suites:
  - name: default
    attributes:
```

run_listはkitchen-ansibleでは使用しないため消してしまいます。
attributesは今回は使用しませんが、extra_varsやtagsを設定する場合はここにぶら下げるので、残しておきます。



以上で、.kitchen.ymlの設定は終わりです。
最終的に.kitchen.ymlは以下のようになります。

```yaml:編集後の.kitchen.yml
---
driver:
  name: docker

provisioner:
  name: ansible_playbook
  playbook: site.yml
  roles_path: ./roles
  group_vars_path: ./group_vars
  host_vars_path: ./host_vars
  filter_plugins: ./filter_plugins
  additional_copy_path:
    - webservers.yml
  hosts: webservers
  require_ansible_omnibus: true
  require_ruby_for_busser: true

platforms:
  - name: ubuntu-14.04
  - name: centos-7.1
    driver_config:
      privileged: true
      run_command: /sbin/init; sleep 3

verifier:
  ruby_bindir: '/usr/bin'

suites:
  - name: default
    attributes:
```

.kitchen.ymlの編集が終わったら、`kitchen list`コマンドを叩くと以下のようにテスト環境が表示されるはずです。

```bash:kitchen_list
$ bundle exec kitchen list
Instance             Driver  Provisioner      Verifier  Transport  Last Action
default-ubuntu-1404  Docker  AnsiblePlaybook  Busser    Ssh        <Not Created>
default-centos-71    Docker  AnsiblePlaybook  Busser    Ssh        <Not Created>
```

テスト環境は[テストスイート名]-[プラットフォーム名]になり、platformとsuiteの組み合わせになります。

### Serverspecの初期化

test-kitchenのルール[^4]にしたがって、`test/integration/default`ディレクトリへServerspecファイルを配置します。
(`test/integration/default`のdefaultの部分は.kitchen.ymlのsuitesのnameに対応します。テストスイートの名前を変更している場合はServerspecを配置するパスも変更しなければなりません。)

```bash:Serverspec初期化
$ cd test/integration/default
$ bundle exec serverspec-init
Select OS type:

  1) UN*X
  2) Windows

Select number: 1

Select a backend type:

  1) SSH
  2) Exec (local)

Select number: 2

 + spec/
 + spec/localhost/
 + spec/localhost/sample_spec.rb
 + spec/spec_helper.rb
 + Rakefile
```

Serverspecの初期化によって作成されたspecディレクトリをserverspecへリネームします。
これは、test-kitchenがテスト環境へインストールするテストランナー(busser)のプラグインをディレクトリ名から決めるためです。ディレクトリ名を変更しない場合busser-specという存在しないプラグインをインストールしようとしてエラーになります。)

```bash
$ mv spec/ serverspec/
```

Serverspecの初期化を行うと、サンプルのspecファイルが作成されます。
中身は以下のとおりで、今回作成したいAnsible play bookにちょうど良いので名前だけ変更して中身はそのまま使います。

```bash:sample_spec.rbのリネーム
mv test/integration/default/serverspec/localhost/sample_spec.rb test/integration/default/serverspec/localhost/webservers_spec.rb
```

```rb:test/integration/default/serverspec/localhost/webservers_spec.rb
require 'spec_helper'

describe package('httpd'), :if => os[:family] == 'redhat' do
  it { should be_installed }
end

describe package('apache2'), :if => os[:family] == 'ubuntu' do
  it { should be_installed }
end

describe service('httpd'), :if => os[:family] == 'redhat' do
  it { should be_enabled }
  it { should be_running }
end

describe service('apache2'), :if => os[:family] == 'ubuntu' do
  it { should be_enabled }
  it { should be_running }
end

describe service('org.apache.httpd'), :if => os[:family] == 'darwin' do
  it { should be_enabled }
  it { should be_running }
end

describe port(80) do
  it { should be_listening }
end
```

リネームしたserverspecディレクトリにGemfileを追加します。

```rb:test/integration/default/serverspec/Gemfile
source 'https://rubygems.org'

gem 'rake'
gem 'serverspec'
```

参考:[Rake dependency missing for older rubies · Issue #28](https://github.com/test-kitchen/busser-serverspec/issues/28)


### 仮のsite.yml作成

以上でTDDの準備が出来ました。と、言いたいところですが、この状態で`kitchen test`コマンドを叩くとエラーが発生します。

```bash:kitche_test
$ cd ~/ansibletdd
$ bundle exec kitchen test
-----> Starting Kitchen (v1.4.2)
-----> Cleaning up any prior instances of <default-ubuntu-1404>

~~~中略~~~

Preparing playbook
>>>>>> ------Exception-------
>>>>>> Class: Kitchen::ActionFailed
>>>>>> Message: Failed to complete #converge action: [unknown file type: site.yml]
>>>>>> ----------------------
>>>>>> Please see .kitchen/logs/kitchen.log for more details
>>>>>> Also try running `kitchen diagnose --all` for configuration
```

とりあえずエラーを解消するため仮のsite.ymlとwebservers.ymlを作成します。

```yaml:site.yml
---
- include: webservers.yml
```

```yaml:webservers.yml
---
- hosts: webservers
  tasks:
```

作成したらもう一度`kitchen test`を行います。

```bash:kitchen_test
$ bundle exec kitchen test
-----> Starting Kitchen (v1.4.2)
-----> Cleaning up any prior instances of <default-ubuntu-1404>

~~~ 中略 ~~~

       Finished in 0.27052 seconds (files took 0.46124 seconds to load)
       4 examples, 4 failures

       Failed examples:

       rspec /tmp/verifier/suites/serverspec/localhost/webservers_spec.rb:8 # Package "apache2" should be installed
       rspec /tmp/verifier/suites/serverspec/localhost/webservers_spec.rb:17 # Service "apache2" should be enabled
       rspec /tmp/verifier/suites/serverspec/localhost/webservers_spec.rb:18 # Service "apache2" should be running
       rspec /tmp/verifier/suites/serverspec/localhost/webservers_spec.rb:27 # Port "80" should be listening

~~~ 後略 ~~~
```

今度は、default-ubuntu-1404でServerspecによる検証まで行われ、全てfailしました。
(test-kitchenはテストが失敗するとそのインスタンスでテストをやめてしまうので、それ以降のインスタンスでテストを行いたい場合は、`bundle exec kitchen test default-centos-71`のようにインスタンス名を指定します。インスタンス名は正規表現でも指定できるので、条件に合う複数のインスタンスをテストすることも出来ます。)

これでようやくTDDの準備が出来ました。
あとは、全てのテストが成功するよう、「書く -> テスト」を繰り返しながらAnsible playbookを書いていけばOKです。

#### Ansible playbookの実装

TDD環境が出来たので、あとはPlay bookを実装していくだけです。
書き方の詳細は本記事の趣旨から外れるのでGoogle先生に聞いてもらうとして、とりあえず全てのテストが通るPlay bookを置いておきます。

```YAML:site.yml
---
- include: webservers.yml
```

```YAML:webservers.yml
---
- hosts: webservers
  roles:
    - apache
```

```YAML:roles/apache/tasks/main.yml
- apt: pkg=apache2
  when: "ansible_os_family == 'Debian'"

- yum: pkg=httpd
  when: "ansible_os_family == 'RedHat'"

- service: name=apache2 state=started enabled=yes
  when: "ansible_os_family == 'Debian'"

- service: name=httpd state=started enabled=yes
  when: "ansible_os_family == 'RedHat'"
```

これで、`kitchen test`すれば全てのテストが成功するはずです。

```bash:kitchen_test
$ bundle exec kitchen test
-----> Starting Kitchen (v1.4.2)
-----> Cleaning up any prior instances of <default-ubuntu-1404>

~~~ 中略 ~~~

       Package "apache2"
         should be installed

       Service "apache2"
         should be enabled
         should be running

       Port "80"
         should be listening

       Finished in 0.1665 seconds (files took 1.22 seconds to load)
       4 examples, 0 failures

       Finished verifying <default-ubuntu-1404> (0m28.28s).

~~~ 中略 ~~~

       Package "httpd"
         should be installed

       Service "httpd"
         should be enabled
         should be running

       Port "80"
         should be listening

       Finished in 0.16315 seconds (files took 1.07 seconds to load)
       4 examples, 0 failures

       Finished verifying <default-centos-71> (0m24.25s).

~~~　後略 ~~~
```

## まとめ

AnsibleをTDDするための環境構築の一連の手順を追いかけました。
test-kitchenを使ってトライ&エラーを簡単に行えるTDD環境を用意するとAnsible(やChef)の開発をサクサク進められる用になりますが、デフォルト設定では動かない箇所などハマりどころがいくつかあるので注意が必要です。ハマった場合はログと関連ドキュメントを読んで頑張りましょう。

## appendix

### Docker imageを指定したい場合

kitchen-docerは`Kitchen::Driver::Docker.default_image`によってplatform.nameからテスト環境のベースイメージのタグを生成しますが、任意のタグを指定することも出来ます。
その場合は、.kitchen.ymlを以下のように変更します。

```yaml.kitchen.yml
platforms:
  - name: ubuntu-14.04
    driver_config:         # <= 追加
      image: ubuntu:14.04  # <= 追加
  - name: centos-7
    driver_config:
       image: centos:centos7 # <= 追加
       privileged: true
       run_command: /sbin/init; sleep 3
```

### 複数のホストグループに対応したい場合

Play bookにホストグループ(例えば、apservers)を追加定義したい場合は、以下のようにします。

テストスイートではprovisionerのオプションを上書きできるので、テストスイートを増やし、スイートごとにホストグループオプションを設定します。

```yaml.kitchen.ymlのsuites
suites:
  - name: webservers    # <= defaultから変更
    provisioner:        # <= 追加
      hosts: webservers # <= 追加
  - name: apservers     # <= 追加
    provisioner:        # <= 追加
      hosts: apservers  # <= 追加
```

次にスイートごとのServerspecを用意します。
現在のtestディレクトリは以下のような状態です。


```bash:
$ tree
test
└── integration
    └── default
        ├── Rakefile
        └── serverspec
            ├── Gemfile
            ├── localhost
            │   └── webservers_spec.rb
            └── spec_helper.rb
```

このディレクトリに以下の操作を行います。

1. defaultディレクトリはテストスイート名なのでこれはwebserversに変更します
2. Rakefile, Gemfile, spec_helper.rbは複数のテストスイートで共通なので、helpersディレクトリを作りそこへ移動します
3. webserversディレクトリをコピーしapserversディレクトリとします
4. コピーしたディレクトリのwebservers_spec.rbをapservers_spec.rbにリネームします。

```bash
$ mv test/integration/default/ test/integration/webservers
$ mkdir -p test/integration/helpers/serverspec
$ cd test/integration
$ mv webservers/Rakefile helpers/
$ mv webservers/serverspec/Gemfile helpers/serverspec/
$ mv webservers/serverspec/spec_helper.rb helpers/serverspec/
$ cp -r webservers/ apservers
$ mv apservers/serverspec/localhost/webservers_spec.rb apservers/serverspec/localhost/apservers_spec.rb
$ tree test
test
└── integration
    ├── apservers
    │   └── serverspec
    │       └── localhost
    │           └── apservers_spec.rb
    ├── helpers
    │   ├── Rakefile
    │   └── serverspec
    │       ├── Gemfile
    │       └── spec_helper.rb
    └── webservers
        └── serverspec
            └── localhost
                └── webservers_spec.rb
```

以上で共通のファイルをhelpersへ移動し、各テストスイートに必要なファイル・ディレクトリを用意できました。あとはapservers_spec.rbファイルをapserversグループに適した形に書き換えれば完了です。

## 参考資料

- [Ansibleのテストをtest-kitchenとServerspec、dockerで行う - Qiita](http://qiita.com/r_takaishi/items/f3e68e3f7072b9327c3f)
- [Chef & Test Kitchen+Serverspec & Docker & PackerによるInfrastructure as Code事始め - Qiita](http://qiita.com/nmatsui/items/7a8a3cdd7cc50169627f)
- [test-kitchenのつかいかた - Qiita](http://qiita.com/sawanoboly/items/9f560bd63ad0712b17ba)
- [Introduction - KitchenCI](http://kitchen.ci/docs/getting-started/)
- [Kitchen — Chef Docs](https://docs.chef.io/kitchen.html)
- [Best Practices — Ansible Documentation](http://docs.ansible.com/ansible/playbooks_best_practices.html)
- [neillturner/kitchen-ansible](https://github.com/neillturner/kitchen-ansible)
- [portertech/kitchen-docker](https://github.com/portertech/kitchen-docker)
- [test-kitchen/busser](https://github.com/test-kitchen/busser)
- [test-kitchen/busser-serverspec](https://github.com/test-kitchen/busser-serverspec)


[^1]: [Best Practices — Ansible Documentation](http://docs.ansible.com/ansible/playbooks_best_practices.html)

[^2]: [Best Practices — Ansible Documentation#top-level-playbooks-are-separated-by-role](http://docs.ansible.com/ansible/playbooks_best_practices.html#top-level-playbooks-are-separated-by-role)

[^3]: [Chef &amp; Test Kitchen+Serverspec &amp; Docker &amp; PackerによるInfrastructure as Code事始め - Qiita](http://qiita.com/nmatsui/items/7a8a3cdd7cc50169627f#test-kitchen%E3%81%AE%E8%A8%AD%E5%AE%9A%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E4%BD%9C%E6%88%90)

[^4]: [Writing a Test - KitchenCI](http://kitchen.ci/docs/getting-started/writing-test)

