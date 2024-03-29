---
toc: true
layout: post
description: k8sの概要
categories: [k8s,]
title: k8s概要
---

# k8s overview

k8s = 複数のホストを束ねてDockerを利用するためのオーケストレーションツール
あたかも1台のサーバのように、透過的にアクセスできる。

## 主な機能

- 複数サーバ間でのコンテナ管理
- コンテナ間のネットワーキング
- コンテナの負荷分散
- コンテナの監視
- 無停止でのアップデート

## k8sのサーバ構成

- マスターサーバ(kubernetes master)
- データストア(backend database)
- ノード

### マスタサーバ(kubernetes master)

- コンテナを操作するためのサーバ
- kubctlコマンドからのリクエストを受けて処理を行う
- ノードのリソース使用状況を確認してコンテナを起動するノードを選択する

### データストア(backend database)

- etcdというKVSでクラスタの構成情報を管理
- マスタサーバに構築する構成もありうる

### ノード

- コンテナを動作させるサーバ

## アプリケーションの構成管理

- Pod
- Replica set
- Deployment

### Pod

- k8sでは複数のコンテナをまとめてPodとして管理する
    - webサーバとプロキシなど
- Podがデプロイの単位になる
    - 開始/停止/作成/削除 がpod単位で行われる
- Pod(pod内のコンテナ)は同じノードに同時にデプロイされる
    - Pod内のコンテナは仮想NICを共有する
        - コンテナ同士はlocalhost軽油で通信できる
        - 共有ディレクトリを介してログ情報をやり取りできる

### Replica set

- k8sクラスタ上で予めpodを作成/起動しておく仕組み
    - クラスタ上に決められた数のpodを起動しておく
    - 起動しておくpodの数をレプリカ数という
- Replica setは起動中のpodを監視し、障害など何らかの理由で停止したpodを削除し、新たなpodを起動する
- pod数を同的に変更してオートスケーリングを実現することも可能

### Deployment

- PodとReplica setをまとめたもの
- Replica setの履歴を管理するもの
    - 履歴を管理できるので、コンテナのアップデートやロールバックができる
- Replica setのテンプレートを持ち、それに従ってReplica setを作る
    - テンプレートでpodの構成を定義する

## ネットワークの管理(Service)

- Serviceはlocalhostのネットワークを管理するもの
- Podに対して外部からアクセスする時はServiceを定義する
- Serviceにはいくつかの種類がある
    - Load Balancer
        - Serviceに対応するIPアドレス:ポート番号にアクセスするとL4レベルの負荷分散が行われる
- サービスによって割り当てられるIPアドレスにはCluster IPとExternal IPがある
    - Cluster IPはクラスタ内のPod同士で通信するためのプライベートIPアドレス
    - External IPはクラスタ外部から通信するためのパブリックIPアドレス
- Podを起動すると、既存のサービスのIPアドレスとポートは環境変数として参照できるようになる
- IngressというPodへの通信を制御する昨日もある
    - k8sが動作する環境によって実装が異なる
        - GCPの場合はHTTP Load Balanser(L7)

## Labelによるリソースの識別

- k8sではリソースを識別するためにランダムな文字列が自動的付与される
- リソースには任意のLabelをつけて管理できる
- Labelはkey-value型の任意の文字列
- サービス定義のSelectorでLabelを指定すると、そのLabelを保つPodのみにリクエストを転送することが可能

## k8sの仕組み

1. Master
    - API Server
        - フロントエンドのREST API
        - 各コンポーネントから情報を受け取ってetcdに保存する
            - 各コンポーネントはREST API経由でetcdの情報にアクセスする
        - 人間はkubctlコマンドはWebのGUIツールからアクセスする
        - 認証・認可の機能も持つ
    - Scheduler
        - Podをどのノードで動かすか制御するバックエンドコンポーネント
    - Controller Manager
        - k8sクラスタの状態を監視するバックエンドコンポーネント
        - 定義ファイルで指定したものと実際の状態をまとめて管理する
1. Datastore
    - k8sクラスタの構成を保持する分散KVS
    - APIサーバから参照される
1. Node
    - kublet
        - ノードで動作するエージェント
        - Dockerコンテナの実行やストレージのマウント機能を持つ
        - ノードのステータスを定期的に監視し、ステータスが変わるとAPI Serverに通知する