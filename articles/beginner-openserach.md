---
title: "OpenSearch入門：Docker Composeで構築 → 検索・マッピング・ISM【初心者向け】"
emoji: "🍿"
type: "tech"
topics: ["opensearch", "aws", "docker", "elasticsearch", "初心者"]
published: true
published_at: 2025-10-20 06:00
publication_name: "secondselection"
---
## 0. はじめに

Amazon OpenSearch Serviceで採用されているOpenSearchをローカル環境で構築し、基本的な操作方法についてまとめました。

**💡対象読者**
・OpenSearchを学びたい初心者。
・ローカル環境で基本的なデータ操作の手順(挿入・検索など)を確認したい方。

**💡この記事を通して得られること**
・OpenSearchの基本を知ることができます。

## 1. OpenSearchとは

OpenSearchとは、オープンソースの全文検索・分析エンジンです。
AWS上でOpenSearchを管理するためのサービスが「Amazon OpenSearch Service」です。

OpenSearchは、データを登録・検索・分析するエンジン本体で、API（HTTPリクエスト）経由でドキュメントを登録したり検索クエリを投げたり、集計などの処理を行います。
OpenSearchをGUIで操作するOpenSearch Dashboardsというツールがあります。OpenSearch Dashboardsを使うと、検索結果をグラフや表にして表示したり、ダッシュボードを作成して複数の可視化をまとめて表示したり、時系列での変化を追うことができます。

### 構成要素

OpenSearchは主に下記の3つの構成要素から成り立っています。

- **ドキュメント**：JSON形式で保存される、データを格納する最小単位。
- **インデックス**：ドキュメントの集合体で、データを格納・検索する集合体。
- **マッピング**：データを保存する際に、事前にどのような項目（フィールド）があり、それぞれの型を定義する仕組み。

![画像](/images/beginner-openserach/opensearch_component.drawio.png)
*[参照:OpenSearch の構成要素](https://zenn.dev/kouichi_itagaki/articles/d77360a5577e7a)*

## 2. 環境構築

OpenSearchとOpenSearch Dashboardsを動かすための基本構成を作っていきます。
（[公式ドキュメント](https://docs.opensearch.org/latest/getting-started/quickstart/)の記載内容をベースに作成）

### 2-1. セットアップ

ディレクトリを作成し、`docker-compose.yml`ファイルを作成します。

```bash
mkdir opensearch-cluster
touch docker-compose.yml
```

`docker-compose.yml`ファイルに記載します。
（今回は公式ドキュメントの[開発用のDocker Composeファイル](https://docs.opensearch.org/latest/install-and-configure/install-opensearch/docker/#sample-docker-compose-file-for-development)のクラスタ構成をそのまま使用しました）。

```yml: docker-compose.yml
services:
  opensearch-node1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node1
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - "DISABLE_INSTALL_DEMO_CONFIG=true"
      - "DISABLE_SECURITY_PLUGIN=true"
    ulimits:
      memlock: { soft: -1, hard: -1 }
      nofile: { soft: 65536, hard: 65536 }
    volumes:
      - opensearch-data1:/usr/share/opensearch/data
    ports:
      - 9200:9200
      - 9600:9600
    networks:
      - opensearch-net
  opensearch-node2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster
      - node.name=opensearch-node2
      - discovery.seed_hosts=opensearch-node1,opensearch-node2
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2
      - bootstrap.memory_lock=true
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m"
      - "DISABLE_INSTALL_DEMO_CONFIG=true"
      - "DISABLE_SECURITY_PLUGIN=true"
    ulimits:
      memlock: { soft: -1, hard: -1 }
      nofile: { soft: 65536, hard: 65536 }
    volumes:
      - opensearch-data2:/usr/share/opensearch/data
    networks:
      - opensearch-net
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    environment:
      - 'OPENSEARCH_HOSTS=["http://opensearch-node1:9200","http://opensearch-node2:9200"]'
      - "DISABLE_SECURITY_DASHBOARDS_PLUGIN=true"
    ports:
      - 5601:5601
    networks:
      - opensearch-net

volumes:
  opensearch-data1:
  opensearch-data2:

networks:
  opensearch-net:
```

### 2-2. パスワードを設定

パスワードを決め、下記のように環境変数として設定します。

```bash
export OPENSEARCH_INITIAL_ADMIN_PASSWORD=<password>
```

### 2-3. コンテナを起動

以下のコマンドを実行して、コンテナをバックグラウンドで起動します。

```bash
docker compose up -d
```

起動後<http://localhost:5601/>にアクセスすれば、ダッシュボードが表示されます。

## 3. データの登録方法

OpenSearch DashboardsのDev toolsを使って、OpenSearchに対してCRUD操作をします。
OpenSearch Dashboardsのホーム画面の右上にあるDev toolsをクリックするとコマンド入力画面が表示されます。

![画像](/images/beginner-openserach/opensearch_ui.drawio.png)

### 3-1. インデックス作成

以下のサンプルは、センサーから送られる`device_id`や`timestamp`、`temperature`、`humidity`、`status`といった項目想定し、インデックスを登録します。
それぞれに適切なデータ型（keyword, date, floatなど）を指定することで、検索や集計を効率的かつ正確に実行できるようになります。

```json
PUT /iot-sensor
{
  "mappings": {
    "properties": {
      "device_id":   { "type": "keyword" },
      "timestamp":   { "type": "date" },
      "temperature": { "type": "float" },
      "humidity":    { "type": "float" },
      "status":      { "type": "keyword" }
    }
  }
}
```

### 3-2. ドキュメント登録(単一)

まずはドキュメントを1件だけ、指定したIDで登録します。各フィールドは、インデックス作成時のマッピングに合わせた型で保存するPUTリクエストです。

```json
PUT /iot-sensor/_doc/sensor-001
{
  "device_id": "sensor-001",
  "timestamp": "2025-09-30T12:34:56Z",
  "temperature": 25.7,
  "humidity": 60.2,
  "status": "OK"
}
```

### 3-3. ドキュメントの一括登録方法

複数のドキュメントをまとめて投入する場合は、`_bulk`を使います。
1行ごとに「操作コマンド」＋「データ本体」を書く形式で、インデックスや更新などをまとめて処理できます。

```json
POST /iot-sensor/_bulk
{ "index": { "_index": "iot-sensor", "_id": "1" } }
{ "device_id": "sensor-001", "timestamp": "2025-09-10T06:00:00Z", "temperature": 25.3, "humidity": 60.2, "status": "ok" }
{ "index": { "_index": "iot-sensor", "_id": "2" } }
{ "device_id": "sensor-002", "timestamp": "2025-09-10T06:01:00Z", "temperature": 25.8, "humidity": 61.0, "status": "ok" }
{ "index": { "_index": "iot-sensor", "_id": "3" } }
{ "device_id": "sensor-003", "timestamp": "2025-09-10T06:02:00Z", "temperature": 29.5, "humidity": 55.1, "status": "alert" }
{ "index": { "_index": "iot-sensor", "_id": "4" } }
{ "device_id": "sensor-004", "timestamp": "2025-09-10T06:03:00Z", "temperature": 26.1, "humidity": 62.5, "status": "ok" }
```

## 4. データの検索方法

代表的な検索を3種類ご紹介します。

### 4.1 全件取得（match_all）

指定されたインデックス内のすべてのドキュメントを取得するシンプルな検索方法です。

```json
GET /iot-sensor/_search
{ "query": { "match_all": {} } }
```

### 4.2 キーワード検索（term）

あるフィールドの値が「完全に値が一致するもの」を探す検索方法です。

```json
GET /iot-sensor/_search
{
  "query": {
    "term": { "status": "alert" }
  }
}
```

### 4.3 集計

検索結果をグループ分けして統計を取ることが可能です。

```json
GET /iot-sensor/_search
{
  "size": 0,
  "aggs": {
    "temperature_trend": {
      "date_histogram": {
        "field": "timestamp",
        "calendar_interval": "1m"
      },
      "aggs": {
        "avg_temp": { "avg": { "field": "temperature" } }
      }
    }
  }
}
```

## 5. ダッシュボードの作成方法

環境構築・ダミーデータの作成ができれば、Visualizationを新規作成しデータを用途に応じて可視化ができます。グラフは下記のような種類が用意されています。

![画像](/images/beginner-openserach/opensearch_dashboard.drawio.png)

こちらの記事に作成方法の詳細が記載されていますので、お試しされたい方はぜひご参照ください。
> @[card](https://blog.shikoan.com/opensearch-dashboards-docker/)

## 6. 不要になったデータを自動的に破棄する方法

データストアにおいて「データを自動的に有効期限切れにして削除する仕組み」を指す概念は、TTL(Time To Live)と一般的に呼ばれています。
OpenSearchでのTTLは、主にIndex State Management (以下ISM)というプラグインを使って、インデックス単位でデータのライフサイクルを管理します。

削除までの流れを自動化するために、「どの状態でどのアクションを行うか」「次の状態への遷移条件」を定義します。

```json
PUT _plugins/_ism/policies/iot-data-policy
{
  "policy": {
    "description": "IoT data: rollover daily, delete after 7 days",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [
          {
            "rollover": {
              "min_index_age": "1d" 
            }
          }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": { "min_index_age": "7d" }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [
          { "delete": {} }
        ]
      }
    ]
  }
}
```

### 有効期限の設定方法

- `"rollover": {"min_index_age": "1d" }`
  - インデックスが作成されてから1日が経過したらロールオーバー（新しいインデックスに切り替え）を行います。

- `"conditions": { "min_index_age": "7d" }`
  - ロールオーバー後、インデックスが7日経過したら削除対象とします。

### 登録内容

登録内容は下記の通りです。
![画像](/images/beginner-openserach/opensearch_ism_cli.drawio.png)

![画像](/images/beginner-openserach/opensearch_ism_gui.drawio.png)

## 7. さいごに

今回Amazon OpenSearch Serviceの機能を確認するため、ローカルでコスト発生なしで色々お試しできました。この記事が同じような初学者の方の学習の助けになりましたら幸いです。

少し余談ですが、AWS OpenSearchで蓄積された固有の知識ベースを活かしてRAG（検索拡張生成）の実装をした記事（参考記事は[こちら](https://aws.amazon.com/jp/blogs/news/multi-tenant-rag-implementation-with-amazon-bedrock-and-amazon-opensearch-service-for-saas-using-jwt/)）を見かけました。AWSの生成AIアプリケーション(Amazon Bedrock)で構築できる事例を知り、興味深かったです。

最近業務外で[AWS Kiro](https://kiro.dev/)を触ったり[マナビDX](https://service.signate.jp/manabi-dx-quest-2025)を通してAI関連も少しずつ学んでいるので、いずれそんな内容もアウトプットできるようになりたいと考えています。

最後までご覧いただき、ありがとうございました。

## 参照

> @[card](https://docs.opensearch.org/latest/getting-started/quickstart/)
> @[card](https://dev.classmethod.jp/articles/how-to-build-opensearch-with-docker/)
> @[card](https://zenn.dev/kouichi_itagaki/articles/d77360a5577e7a)
> @[card](https://blog.shikoan.com/opensearch-dashboards-docker/)
