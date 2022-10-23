---
toc: true
layout: post
description: test-kitchenでkitchen-dockerを使う場合に.kitchen.ymlでテストインスタンスのベースイメージを設定する方法
categories: [kitchen-docker,]
title: kitchen-docker用Dockerfile
---

# kitchen-docker用Dockerfile

test-kitchenでkitchen-dockerを使う場合、.kitchen.ymlでテストインスタンスのベースイメージを設定する方法として、以下の3種類があります。

1. 設定しない
    (内部ではKitchen::Driver::Docker.default_image()によって、platform.nameからname:TAGが作られる)

    ```yaml:.kitchen.yml
    ~~~ 前略 ~~~    
    platforms:
      - name: centos-7.1                   # <= ここからcentos:centos7が生成される
        driver_config:
          privileged: true
          run_command: /sbin/init; sleep 3
    ~~~ 後略 ~~~
    ```

2. name:TAGで設定

    ```yaml:.kitchen.yml
    ~~~ 前略 ~~~    
    platforms:
      - name: centos-7.1
        driver_config:
          image: centos:centos7            # <= コレ
          privileged: true
          run_command: /sbin/init; sleep 3
    ~~~ 後略 ~~~
    ```

3. Dockerfileのパスを設定
    (内部ではerbテンプレートとして扱われるので、eRubyタグでマークアップ可能。)

    ```yaml:.kitchen.yml
    ~~~ 前略 ~~~
    platforms:
      - name: centos-7.1
        driver_config:
          dockerfile: test/platforms/centos-7.1/Dockerfile # <= コレ
          privileged: true
          run_command: /sbin/init; sleep 3
    ~~~ 後略 ~~~
    ```    

    ```:test/platforms/centos-7.1/Dockerfile
    FROM centos:centos7
    ```


kitchen-dockerでは上記1. 2. の場合ベースイメージとして設定されたイメージをFROMとし、test-kitchenで使用する場合に必要となるパッケージや設定を加えたDockerfileを作って`docker build`を行い、ビルドされたコンテナイメージをテスト環境として使用します。
しかし、3. の場合はパスで設定されたDockerfileをそのまま使って`docker build`するので、Dockerfileに1. 2.の場合にビルドされる内容が含まれていないとテスト環境として使用できません。

ということで、それに対応しているDockerfileが以下になります。

```:Dockerfile
<%=     from = "FROM #{@image}"
        platform = case @platform
        when 'debian', 'ubuntu'
          disable_upstart = <<-eos
            RUN dpkg-divert --local --rename --add /sbin/initctl
            RUN ln -sf /bin/true /sbin/initctl
          eos
          packages = <<-eos
            ENV DEBIAN_FRONTEND noninteractive
            RUN apt-get update
            RUN apt-get install -y sudo openssh-server curl lsb-release
          eos
          @disable_upstart ? disable_upstart + packages : packages
        when 'rhel', 'centos', 'fedora'
          <<-eos
            RUN yum clean all
            RUN yum install -y sudo openssh-server openssh-clients which curl
            RUN ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N ''
            RUN ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key -N ''
          eos
        when 'arch'
          <<-eos
            RUN pacman -Syu --noconfirm
            RUN pacman -S --noconfirm openssh sudo curl
            RUN ssh-keygen -A -t rsa -f /etc/ssh/ssh_host_rsa_key
            RUN ssh-keygen -A -t dsa -f /etc/ssh/ssh_host_dsa_key
          eos
        when 'gentoo'
          <<-eos
            RUN emerge sync
            RUN emerge net-misc/openssh app-admin/sudo
            RUN ssh-keygen -A -t rsa -f /etc/ssh/ssh_host_rsa_key
            RUN ssh-keygen -A -t dsa -f /etc/ssh/ssh_host_dsa_key
          eos
        when 'gentoo-paludis'
          <<-eos
            RUN cave sync
            RUN cave resolve -zx net-misc/openssh app-admin/sudo
            RUN ssh-keygen -A -t rsa -f /etc/ssh/ssh_host_rsa_key
            RUN ssh-keygen -A -t dsa -f /etc/ssh/ssh_host_dsa_key
          eos
        else
          raise ActionFailed,
          "Unknown platform '#{@platform}'"
        end

        username = @username
        password = @password
        public_key = IO.read(@public_key).strip
        homedir = username == 'root' ? '/root' : "/home/#{username}"

        base = <<-eos
          RUN if ! getent passwd #{username}; then \
                useradd -d #{homedir} -m -s /bin/bash #{username}; \
              fi
          RUN echo #{username}:#{password} | chpasswd
          RUN echo '#{username} ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
          RUN mkdir -p /etc/sudoers.d
          RUN echo '#{username} ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers.d/#{username}
          RUN chmod 0440 /etc/sudoers.d/#{username}
          RUN mkdir -p #{homedir}/.ssh
          RUN chown -R #{username} #{homedir}/.ssh
          RUN chmod 0700 #{homedir}/.ssh
          RUN touch #{homedir}/.ssh/authorized_keys
          RUN chown #{username} #{homedir}/.ssh/authorized_keys
          RUN chmod 0600 #{homedir}/.ssh/authorized_keys
        eos
        custom = ''
        Array(@provision_command).each do |cmd|
          custom << "RUN #{cmd}\n"
        end
        ssh_key = "RUN echo '#{public_key}' >> #{homedir}/.ssh/authorized_keys"
        # Empty string to ensure the file ends with a newline.
        [from, platform, base, custom, ssh_key, ''].join("\n")
%>

```

これをdriver_config.dockerfileに設定したパスへ保存し、個別のテスト環境に必要なDockerbuildを追記してあげれば使えるハズです。


実のところ、Kitchen::Driver::Docker.build_dockerfileをコピーしつつ若干手をいれて、erbテンプレートとして使えるようにしただけです。

