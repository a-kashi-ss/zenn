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

この記事では**Amazon ECS（Elastic Container Service）** の**サーバーレス実行環境である Fargate** に焦点を当てて、基本概念の理解を中心に解説します。

:::message

### 対象読者

- Dockerの基礎（build / run /pushなど）を理解している方
- AWSでコンテナを動かす方法の基礎知識を学びたい初学者
- Fargateの概要を知りたい方

:::

### 記事を読んで得られること

- AWS ECSとFargateの基本概念を体系的に理解できる
- ECSを使ったサーバーレスなコンテナデプロイの全体像がつかめる
- Docker → ECR → ECS(Fargate) の一連の流れがイメージできる

## 2. AWS ECSの概要

### 2-1. ECSを理解するための前提知識：コンテナとDocker

コンテナ技術である **Docker** は、アプリケーションとその実行環境をひとまとめにしてパッケージ化できる仕組みです。

実行環境が変わった場合に、アプリケーションが動かない...という悩みを軽減してくれるのがコンテナです。

![画像](/images/begginer-aws-ecs/merit_docker.drawio.png)*[参照元:今さら聞けないAWS ECSとは？Fargateとは？](https://qiita.com/K5K/items/0d8dbdb39fbb0375e2bd)*

### 2-2. AWS ECSを使う理由：なぜ必要なのか

本番環境にアプリケーションをデプロイし、コンテナを運用するには以下のような**管理**が必要です。

- どのサーバーでどのコンテナを何台動かすか、メモリは足りるか
- アクセスが増減した際のコンテナの自動スケーリング
- コンテナが落ちた場合の新規入れ替え

これらの監視・管理を自動化で一元的に行う技術として「コンテナオーケストレーションツール」と呼び、ECSもその1つです。ECSを利用することで、サービスを安定して提供し続けるための基盤を整えることができます。

![画像](/images/begginer-aws-ecs/merit_container.drawio.png)*[参照元:今さら聞けないAWS ECSとは？Fargateとは？](https://qiita.com/K5K/items/0d8dbdb39fbb0375e2bd)*

## 3. AWS ECSの基本構成要素

ECSを理解するためには、まず以下の4つの概念を押さえる必要があります。

### 3-1. ECSを構成する4つの重要要素

| 要素 | 役割 |
| :--- | :--- |
| **クラスター** | コンテナを動かすための**実行環境**の集合体|
| **タスク定義** | どのイメージをどんなリソースで動かすかを決めた**設計図**|
| **タスク** | タスク定義に基づき実際に起動したコンテナ|
| **サービス** | タスクを指定数だけ維持し、復旧・スケーリングを管理 |

![画像](/images/begginer-aws-ecs/ecs_role.drawio.png)*[参照元:【全図解】コンテナ・Dockerから学ぶAmazon ECR・Amazon ECS入門](https://qiita.com/K5K/items/0d8dbdb39fbb0375e2bd#ecs%E3%81%AE%E6%A7%8B%E6%88%90%E8%A6%81%E7%B4%A0)*

### 3-2. 2種類の実行環境

ECSでは、コンテナを実行するための基盤を以下の2つのタイプから選択できます。

| 項目 | ECS on EC2 | ECS on Fargate |
|------|-------------|------------------|
| **概要** | EC2インスタンス上でコンテナを動かす方式 | AWS が実行基盤をすべて管理するサーバーレス方式 |
| **インフラ管理** | 必要（OS、メンテナンス、容量管理など） | 不要（EC2 の準備・管理はすべて不要） |
| **運用工数** | 高い | 非常に低い |
| **スケーリング** | 自分で EC2 の増減を管理する必要あり | Fargate が自動で必要な分だけリソース確保 |
| **コスト傾向** | 長時間稼働では安くなる場合あり | 若干高くなる場合もある |
| **向いているケース** | 細かなチューニングが必要なシステム、既存 EC2 資産を活用したい場合 | 運用負荷を下げたい場合、初心者・小規模チーム、サーバーレス志向 |

**Fargate** は、インスタンス管理の運用コストがかからない点では、EC2より優勢です。
このあとの記事では、Fargateでのデプロイ方法の解説を進めます。

## 4. Fargateでコンテナをデプロイするステップ

コンテナをFargateを使ってデプロイする流れの概要を解説します。

### 4-1. Dockerイメージの準備とECRへの登録

ECSが利用できるように、まずコンテナイメージをAWSの専用レジストリである**ECR(Elastic Container Registry)**に保存する必要があります。

#### 【ECR の作成】

Fargateがコンテナを取得できるよう、まずはAWSのコンテナレジストリである  
**ECR（Elastic Container Registry）** にイメージを保存します。

#### ✅ ECRリポジトリを作成

- リポジトリ名を設定（例：`my-web-app`）

#### ✅ Dockerイメージのビルドとプッシュ

手元の開発環境でDockerイメージをビルドし、先ほど作成したECRにプッシュします。

- AWS CLIを使ったコマンド実行例

```bash
# AWS CLIを使ったコマンド実行例
## ECRリポジトリの認証情報を取得
aws ecr get-login-password --region <リージョン名> | docker login --username AWS --password-stdin <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com

## イメージをビルド
docker build -t my-web-app .

## イメージにタグ付け
docker tag my-web-app:latest <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com/my-web-app:latest

## ECRへpush
docker push <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com/my-web-app:latest
```

コンテナイメージがECSからアクセス可能な状態になりました。

#### ✅ 登録が正しくできたか確認（推奨）

:::details　👻私自身の失敗談。

恥ずかしながらここまでの作業で不備があったことに気付けず作業を進めた結果、後述の作業で発生したエラー解消に時間をとても消費してしまいました。私とよく似た初心者の方は特にこの部分の確認を大いにおすすめします。

:::

```bash
# AWS CLIを使ったコマンド実行例

## ECRからpull
docker pull <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com/my-web-app:latest

## ポートマッピング（HTTP通信の場合）
docker run -d -p 80:80  <アカウントID>.dkr.ecr.<リージョン名>.amazonaws.com/my-web-app:latest

## ローカルホストにアクセスし確認
http://localhost:80
```

:::details Tips📝 Amazon ECR Public Gallery

Amazon ECR Public Galleryは、Amazonのパブリックなコンテナイメージ検索・共有用ポータルで、認証不要で誰でもリポジトリを閲覧・イメージをプルできます。

@[card](https://gallery.ecr.aws/)

:::

### 4-2. Fargateの設定

#### ✅ タスク定義の作成

コンテナをFargateでどう動かすかを記述する設計図を用意する。

- CPU / メモリを指定
- コンテナ定義でECRのイメージURIを設定
- ポートマッピングを設定

#### ✅ クラスター・サービスの作成

最後にECSクラスターを作成し、その中でサービスを作成します。

- 使用するタスク定義
- 起動させるタスク数
- ネットワーク設定（VPC / サブネット / セキュリティグループ）  

:::message

#### ロードバランサーを使用するメリット

サービス作成時にALB（Application Load Balancer）を紐付けすることで、ECSが下記の処理を自動で行います。

- タスクの増減に応じた負荷分散
- ヘルスチェックの管理
- スケールアウトしたタスクを自動登録

:::

## 4-3. ロールについて

ECSのロールがとても混乱したので、概要を記載しておきます。
詳しくは参照した記事をご参照いただければ幸いです。

| ロール名              | 役割                           | 主な用途                                                                |
|-----------------------|----------------------------------------------|----------------------------------------------------------------------------------------|
| **タスク実行ロール**  | ECS エージェント（裏方処理）                 | ECR からイメージを pull・CloudWatch Logs にログ送信・Secrets/SSM の取得     |
| **タスクロール**      | コンテナ内アプリケーション                   | アプリが AWS リソースへアクセス（S3/RDS/ DynamoDB/API など）                         |
| **EC2ホストロール**   | ECS コンテナエージェント（EC2 上で動作）     | ECS エージェント自体の通信・EC2 ホストでの ECS 管理に必要な権限を付与           |

> 参考にしたリンクはこちらです。
> @[card](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/Welcome.html)

## 5. おわりに

この記事では、AWS ECSとFargateを使ってコンテナをクラウドで動かす方法を解説しました。
最後までご覧いただきありがとうございました。

## 参考記事

下記の記事を何度も参照させていただきました。
もっと詳しく知りたい方は、よかったらぜひ参考にしてみて下さい。

@[card](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/Welcome.html)
@[card](https://blog.serverworks.co.jp/container-docker-ecs-ecr-beginner-zukai)
@[card](https://qiita.com/K5K/items/0d8dbdb39fbb0375e2bd)
@[card](https://zenn.dev/mi_01_24fu/articles/aws-ecs-2024_03_18)
@[card](https://o2mamiblog.com/aws-ecs-beginner/)
