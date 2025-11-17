---
title: "【初心者向け】サーバーレスコンテナ入門（AWS ECS × Fargate）"
emoji: "👩‍🍳"
type: "tech"
topics: ["aws","ecs","fargate","docker","初心者"]
published: true
published_at: 2025-12-01 05:00
publication_name: "secondselection"
---

## 1. はじめに

この記事では**Amazon ECS（Elastic Container Service）** の**サーバーレス実行環境であるFargate** に焦点を当てて、基本概念の理解を中心に解説します。

:::message

### 対象読者

- AWSでコンテナを動かす方法の基礎を学びたい初学者
- Fargateの概要を知りたい方

:::

### 記事を読んで得られること

- ECSとFargateの基本概念を理解できる
- Docker → ECR → Fargateの流れが概要を把握できる

## 2. ECSの概要

### 2-1. 前提知識

コンテナ技術である **Docker** は、アプリケーションとその実行環境をひとまとめにしてパッケージ化できる仕組みです。

コンテナを使うことによって、実行環境が変わった場合にアプリケーションが動かない...といった不測の事態の発生リスクを軽減できます。

:::details 📝コンテナを使うとき・使わないとき。

![画像](/images/begginer-aws-ecs/merit_docker.drawio.png)*[参照元:今さら聞けないAWS ECSとは？Fargateとは？](https://qiita.com/K5K/items/0d8dbdb39fbb0375e2bd)*

:::

### 2-2. ECSとは

本番環境においてアプリケーションをコンテナ化し運用する際には、次のような構成・運用上の制御が必要になります。

- どのサーバーで、どのコンテナを、どれだけのメモリで、何台稼働させるか
- アクセスの増減に応じて、コンテナを自動的にスケーリングすること
- コンテナが停止した場合に、自動で新しいインスタンスと入れ替えること

これらの運用ルールを、**監視・管理・自動化という観点からまとめて扱える仕組み**を「**コンテナオーケストレーションツール**」と呼び、そのうちの1つがECSです。

ECSを活用することによって、**サービスを安定的に提供し続けるための基盤を整備**できます。

:::details 📝オーケストレーションツールがある時・ない時。

![画像](/images/begginer-aws-ecs/merit_container.drawio.png)*[参照元:今さら聞けないAWS ECSとは？Fargateとは？](https://qiita.com/K5K/items/0d8dbdb39fbb0375e2bd)*

:::

## 3. ECSの基本構成要素

ECSを理解するためには、まず以下の4つの概念を押さえる必要があります。

### 3-1. ECSを構成する4つの重要要素

| 要素 | 役割 |
| :--- | :--- |
| **クラスター** | コンテナを動かすための**実行環境**の集合体|
| **タスク定義** | どのDockerイメージをどのリソースを使って動かすかを定めた**設計図**|
| **タスク** | タスク定義に基づき実際に起動した**コンテナ**|
| **サービス** | タスクを指定数だけ維持し、**復旧・スケーリングを管理** |

![画像](/images/begginer-aws-ecs/ecs_role.drawio.png)*[参照元:【全図解】コンテナ・Dockerから学ぶAmazon ECR・Amazon ECS入門](https://qiita.com/K5K/items/0d8dbdb39fbb0375e2bd#ecs%E3%81%AE%E6%A7%8B%E6%88%90%E8%A6%81%E7%B4%A0)*

### 3-2. Fargateについて

ECSでは、コンテナを実行するための基盤を以下の2つのタイプから選択できます。

- ECS on Fargate
- ECS on EC2

「ECS on Fargate」は、コンテナを実行するためのインフラ（サーバー）管理をAWSに完全にお任せできます。
一方で「ECS on EC2」はプロビジョニング（準備）、スケーリング（増減）、パッチ適用（OSの更新など）といったサーバーの管理はユーザー側で行う必要があります。

## 4. Fargateでコンテナをデプロイするステップ

Fargateを使ってコンテナをデプロイする流れは下記のとおりです。

### 4-1. Dockerイメージの準備とECRへの登録

ECSを利用する際コンテナの情報は、AWSの専用レジストリであるECR(Amazon Elastic Container Registry)に保存しておく必要があります。

#### ✅ ECRリポジトリを作成

まずはDockerイメージをECRに保存します。

- リポジトリ名を設定（例：`my-web-app`）

#### ✅ Dockerイメージのビルドとプッシュ

- 手元の開発環境でDockerイメージをビルドし、先ほど作成したECRにプッシュします。

```bash
# AWS CLIを使ったコマンド実行例
## ECRリポジトリの認証情報を取得
aws ecr get-login-password --region <リージョン名> | docker login --username AWS --password-stdin <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com
## Dockerイメージをビルド
docker build -t my-web-app .
## Dockerイメージにタグ付け
docker tag my-web-app:latest <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com/my-web-app:latest
## ECRへpush
docker push <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com/my-web-app:latest
```

- ECSからDockerイメージにアクセスが可能になりました。

#### ✅ 登録が正しくできたか確認する方法（推奨）

```bash
# AWS CLIを使ったコマンド実行例
## ECRからpull
docker pull <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com/my-web-app:latest
## ポートマッピング（HTTP通信の場合）
docker run -d -p 80:80  <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com/my-web-app:latest
## ローカルホストにアクセスし確認
http://localhost:80
```

:::details　👻私自身の失敗談。

恥ずかしながらここまでの作業で不備があったことに気付けず作業を進めた結果、後述の作業で発生したエラー解消に時間をとても消費してしまいました。
私とよく似た初心者の方は、特にこの部分の確認は実施することをおすすめします。

:::

### 4-2. Fargateの設定

#### ✅ タスク定義の作成

コンテナをFargate上でどのように実行するかを定める設計図を用意する。

- CPU / メモリを指定
- コンテナ定義でECRのイメージURIを設定
- ポートマッピングを設定

#### ✅ クラスター・サービスの作成

最後にECSクラスターを作成し、その中でサービスを作成します。

- 使用するタスク定義
- 起動させるタスク数
- ネットワーク設定（VPC / サブネット / セキュリティグループ）  

:::message

#### ロードバランサーの役割

サービス作成時にALB（Application Load Balancer）を紐付けすることで、ECSが下記の処理を自動で行います。

- タスクの増減に応じた負荷分散
- ヘルスチェックの管理
- スケールアウトしたタスクを自動登録

:::

:::details 📝 ロールについて。

ECSのロールで混乱した際にはこちらの記事を参照しました。

> @[card](https://zenn.dev/fdnsy/articles/d42db2c989a637)

:::

## 5. おわりに

この記事では、ECSとFargateを使ってコンテナをクラウドで動かす方法を解説しました。
最後までご覧いただきありがとうございました。

## 参考

下記の記事を何度も参照させていただきました。
もっと詳しく知りたい方は、よかったらぜひ参考にしてみて下さい。

@[card](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/Welcome.html)
@[card](https://blog.serverworks.co.jp/container-docker-ecs-ecr-beginner-zukai)
@[card](https://qiita.com/K5K/items/0d8dbdb39fbb0375e2bd)
@[card](https://zenn.dev/mi_01_24fu/articles/aws-ecs-2024_03_18)
@[card](https://o2mamiblog.com/aws-ecs-beginner/)
