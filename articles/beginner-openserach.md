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

OpenSearch DashboardsのDev toolsを使って、OpenSearchを操作します。
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

環境構築およびデータ登録後、Visualizationを新規作成しデータを用途に応じて可視化ができます。グラフは下記のような種類が用意されています。

![画像](/images/beginner-openserach/opensearch_dashboard.drawio.png)

### サンプル

「3-1. インデックス作成」で登録したデータをもとに簡単なテーブルを作成します。
![画像](/images/beginner-openserach/opensearch_visualization.drawio.png)

- 1.ホーム画面左上の「≡」をクリック
- 2.OpenSearchDashboardsの「∨」記号をクリック
- 3.「Dashboards」をクリック
- 4.Dashboardsのページが表示されたことを確認し、「Create」「Dashboard」をクリック

  ![画像](/images/beginner-openserach/opensearch_visualization1.drawio.png)

- 5.DashboardsのEditing New Dashboard画面が表示されたことを確認し、「Create new」ボタンをクリック
- 6.New Visualizationのポップアップが表示されたことを確認し、「Data Table」をクリック
- 7.New Data Table/Choose a sourceのポップアップが表示されたことを確認し、「iot-sensor」をクリック

  ![画像](/images/beginner-openserach/opensearch_visualization2.drawio.png)

- 8.右側のサイドバー内のMetrics下の「＞」をクリック
- 9.「Agrregation」欄に集計方法を選択し、「Field」に集計する項目を選択
  - 補足:画面のサンプルでは、Abrregation欄にAbverage(平均)、Field欄はtemparature(温度)を選んでいます。
- 10.画面右下の「Updateボタン」をクリックし、想定している内容が画面上で取得できているか確認
- 11.Metricを追加する場合「＋add」ボタンをクリックし、6と同じ要領で登録
  ![画像](/images/beginner-openserach/opensearch_visualization3.drawio.png)

- 12.Metricの登録がすべて完了すれば、画面右上の「Save」ボタンをクリック
- 13.Save visualizationのポップアップが表示されたことを確認し、Titleを登録後「Save」ボタンをクリック
  ![画像](/images/beginner-openserach/opensearch_visualization4.drawio.png)

- 14.DashboardsのEditingページに遷移するので、右上部「Save」ボタンをクリック
  ![画像](/images/beginner-openserach/opensearch_visualization6.drawio.png)

:::message

**indexに格納済みのデータも再計算できます。**

- 画像サンプルは、inputで入力済の摂氏を華氏で表示させる内容です。
- Metricを追加し、Advanced欄のJSONinputに処理内容を記述します。
  ![画像](/images/beginner-openserach/opensearch_visualization5.drawio.png)

:::

## 6. 不要になったデータを自動的に破棄する方法

データストアにおいて「データを自動的に有効期限切れにして削除する仕組み」を指す概念は、TTL(Time To Live)と一般的に呼ばれています。
OpenSearchでのTTLは、主にIndex State Management (以下ISM)というプラグインを使って、インデックス単位でデータのライフサイクルを管理します。

### 手順

テスト用のインデックスとして、下記を用意しました。

```json
PUT test-000001/_doc/1
{
  "user": "testuser",
  "post_date": "2020-05-08T14:12:12",
  "message": "ISM testing"
}
```

#### 1. エイリアスを設定します

rolloverを行うために、インデックスを指すエイリアスを作成します。

```json
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "test-000001",
        "alias": "test-alias"
      }
    }
  ]
}
```

#### 2. ISMポリシーを作成します

:::message

### 有効期限の設定方法

- `"rollover": {"min_index_age": "10m" }`
  - インデックスが作成されてから10分が経過したらロールオーバー（新しいインデックスに切り替え）を行います。

- `"conditions": { "min_index_age": "20m" }`
  - ロールオーバー後、インデックスが作成されてから20分が経過したら削除対象とします。

:::

```json
PUT _plugins/_ism/policies/test_policy
{
  "policy": {
    "description": "test delete",
    "last_updated_time": 1642027350875,
    "schema_version": 1,
    "error_notification": null,
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [
          {
            "rollover": {
              "min_index_age": "10m"
            }
          }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": {
              "min_index_age": "20m"
            }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [
          {
            "delete": {}
          }
        ],
        "transitions": []
      }
    ],
    "ism_template": {
      "index_patterns": [
        "test-*"
      ],
      "priority": 100
    }
  }
}
```

#### 3. ISMポリシーをインデックスにアタッチします

- 左のサイドバーから`Index Management`タブを選択
- `Indexes`欄の横の「∨」記号をクリックしたあと表示される`Indexes`を選択。
- ISMポリシーをアタッチするインデックスの左横のチェックボックスを選択
- 右上の`Actions`をクリックし、`Apply policy`をクリック
- `Apply policy`のポップアップが表示されたあと、`policyId`の選択肢から`test_policy`を選択し`Apply`ボタンをクリック
- `Rollover alias`欄でエイリアスを指定し、`Apply`ボタンをクリック
- 左のサイドーのリスト表示される`Policy managed indexes`をクリック
- `Policy managed index`画面が表示されるので、正しくアタッチできたかどうかを確認

## 7. さいごに

今回Amazon OpenSearch Serviceの機能を確認するため、ローカルでコスト発生なしで色々お試しできました。この記事が同じような初学者の方の学習の助けになりましたら幸いです。

少し余談ですが、Amazon OpenSearch Serviceで蓄積された固有の知識ベースを活かしてRAG（検索拡張生成）の実装をした記事（参考記事は[こちら](https://aws.amazon.com/jp/blogs/news/multi-tenant-rag-implementation-with-amazon-bedrock-and-amazon-opensearch-service-for-saas-using-jwt/)）を見かけました。AWSの生成AIアプリケーション(Amazon Bedrock)で構築できる事例を知り、興味深かったです。

最近業務外で[AWS Kiro](https://kiro.dev/)を触ったり[マナビDX](https://service.signate.jp/manabi-dx-quest-2025)を通してAI関連も少しずつ学んでいるので、いずれそんな内容もアウトプットしたいと考えています。

最後までご覧いただき、ありがとうございました。

## 参照

> @[card](https://docs.opensearch.org/latest/getting-started/quickstart/)
> @[card](https://dev.classmethod.jp/articles/how-to-build-opensearch-with-docker/)
> @[card](https://zenn.dev/kouichi_itagaki/articles/d77360a5577e7a)
> @[card](https://blog.shikoan.com/opensearch-dashboards-docker/)
