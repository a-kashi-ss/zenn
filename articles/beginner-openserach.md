---
title: "opensearch【】"
emoji: "🌎"
type: "tech"
topics: ["vscode", "vscode", "vscode", "vscode", "vscode"]
published: true
published_at: 2025-12-31 06:00
publication_name: "secondselection"
---
## OpenSearchの使い方

### 1. DockerHubからpullする

```bash
docker pull opensearchproject/opensearch:latest && docker run -it -p 9200:9200 -p 9600:9600 -e "discovery.type=single-node" -e "DISABLE_SECURITY_PLUGIN=true" opensearchproject/opensearch:latest
```

### 2. curlで接続確認

```bash
~/sample-test/opensearch$ curl http://localhost:9200
{
  "name" : "c1b0bc962b3f",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "Ybkj0rsdSCiRRsnt70GWnQ",
  "version" : {
    "distribution" : "opensearch",
    "number" : "3.2.0",
    "build_type" : "tar",
    "build_hash" : "6adc0bf476e1624190564d7fbe4aba00ccf49ad8",
    "build_date" : "2025-08-12T03:55:01.226522683Z",
    "build_snapshot" : false,
    "lucene_version" : "10.2.2",
    "minimum_wire_compatibility_version" : "2.19.0",
    "minimum_index_compatibility_version" : "2.0.0"
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}
```

### 3. docker-compose.ymlを配置する

```bash
mkdir opensearch-cluster
touch docker-compose.yml
```

```yml
services:
  opensearch-node1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster # Name the cluster
      - node.name=opensearch-node1 # Name the node that will run in this container
      - discovery.seed_hosts=opensearch-node1,opensearch-node2 # Nodes to look for when discovering the cluster
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2 # Nodes eligibile to serve as cluster manager
      - bootstrap.memory_lock=true # Disable JVM heap memory swapping
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Set min and max JVM heap sizes to at least 50% of system RAM
      - "DISABLE_INSTALL_DEMO_CONFIG=true" # Prevents execution of bundled demo script which installs demo certificates and security configurations to OpenSearch
      - "DISABLE_SECURITY_PLUGIN=true" # Disables Security plugin
    ulimits:
      memlock:
        soft: -1 # Set memlock to unlimited (no soft or hard limit)
        hard: -1
      nofile:
        soft: 65536 # Maximum number of open files for the opensearch user - set to at least 65536
        hard: 65536
    volumes:
      - opensearch-data1:/usr/share/opensearch/data # Creates volume called opensearch-data1 and mounts it to the container
    ports:
      - 9200:9200 # REST API
      - 9600:9600 # Performance Analyzer
    networks:
      - opensearch-net # All of the containers will join the same Docker bridge network
  opensearch-node2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster # Name the cluster
      - node.name=opensearch-node2 # Name the node that will run in this container
      - discovery.seed_hosts=opensearch-node1,opensearch-node2 # Nodes to look for when discovering the cluster
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2 # Nodes eligibile to serve as cluster manager
      - bootstrap.memory_lock=true # Disable JVM heap memory swapping
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Set min and max JVM heap sizes to at least 50% of system RAM
      - "DISABLE_INSTALL_DEMO_CONFIG=true" # Prevents execution of bundled demo script which installs demo certificates and security configurations to OpenSearch
      - "DISABLE_SECURITY_PLUGIN=true" # Disables Security plugin
    ulimits:
      memlock:
        soft: -1 # Set memlock to unlimited (no soft or hard limit)
        hard: -1
      nofile:
        soft: 65536 # Maximum number of open files for the opensearch user - set to at least 65536
        hard: 65536
    volumes:
      - opensearch-data2:/usr/share/opensearch/data # Creates volume called opensearch-data2 and mounts it to the container
    networks:
      - opensearch-net # All of the containers will join the same Docker bridge network
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    ports:
      - 5601:5601 # Map host port 5601 to container port 5601
    expose:
      - "5601" # Expose port 5601 for web access to OpenSearch Dashboards
    environment:
      - 'OPENSEARCH_HOSTS=["http://opensearch-node1:9200","http://opensearch-node2:9200"]'
      - "DISABLE_SECURITY_DASHBOARDS_PLUGIN=true" # disables security dashboards plugin in OpenSearch Dashboards
    networks:
      - opensearch-net

volumes:
  opensearch-data1:
  opensearch-data2:


networks:
  opensearch-net:

```

### 4. コンテナが正常に起動したか確認します

```bash

~/sample-test/opensearch/opensearch-cluster$ curl http://localhost:9200
{
  "name" : "c1b0bc962b3f",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "Ybkj0rsdSCiRRsnt70GWnQ",
  "version" : {
    "distribution" : "opensearch",
    "number" : "3.2.0",
    "build_type" : "tar",
    "build_hash" : "6adc0bf476e1624190564d7fbe4aba00ccf49ad8",
    "build_date" : "2025-08-12T03:55:01.226522683Z",
    "build_snapshot" : false,
    "lucene_version" : "10.2.2",
    "minimum_wire_compatibility_version" : "2.19.0",
    "minimum_index_compatibility_version" : "2.0.0"
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}

```

## 5. 環境変数にパスワードを設定し、コンテナを起動します

$ export OPENSEARCH_INITIAL_ADMIN_PASSWORD=90@A1qwe#Pa_q
今回はこれでやった。
<https://dev.classmethod.jp/articles/how-to-build-opensearch212-with-docker/>

```bash
export OPENSEARCH_INITIAL_ADMIN_PASSWORD=<password>
docker-compose up -d
```

## 6. アクセスする

<http://localhost:5601/>

ダッシュボードが表示されました。

APIレスポンスの確認方法：
<http://localhost:9200>

## ダッシュボードを使用せずにデータを操作する方法

<https://qiita.com/rumblekat03/items/2b0a697d973b4fb0337e>

## ダッシュボードを試してみよう

### Level1 OpenSearchを導入し、簡単なAPIを実行

Dev ToolsからAPIを実行してみる。
ダッシュボード内に用意されている「Dev Tools」からAPIを実行してみます。
Dev Toolsには左上のハンバーガーメニューからアクセスできます。

<https://sheltie-garage.xyz/tech/2023/01/opensearchservice%E3%81%AE%E7%92%B0%E5%A2%83%E3%82%92%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%AB%E3%81%AB%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B/>

<https://docs.opensearch.org/latest/api-reference/cluster-api/cluster-health/>

左側のテキストエリアに実行したいAPIを入力し、再生ボタンを押すことでAPIが実行され、右側に結果が表示されます。
クラスターの状態はグリーンで、特に問題ない状態になっています。

**スクリーンショット 2025-09-10 143326**

### Level2 インデックスの作成

### level3 トークナーザーの設定

### level4 SQLライクな検索クエリの記述

<https://jacoyutorius.hatenablog.com/entry/2022/12/24/074336>

##### 1. 全件取得 (`match_all`)

すべてのドキュメントを取得する最もシンプルな検索。

```http
GET /<index>/_search
{
  "query": {
    "match_all": {}
  }
}

```

##### 2. キーワード検索（match クエリ）

```http
GET /<index>/_search
{
  "query": {
    "match": {
      "<field>": {
        "query": "検索したい語",
        "operator": "and",         # 'or' or 'and'
        "fuzziness": "AUTO",       # ファジー検索対応
        "boost": 1.2,              # スコア重み調整
        "lenient": true            # 型ミスマッチを無視して継続
      }
    }
  }
}
```

3. 範囲検索（rangeクエリ）

```http
GET /<index>/_search
{
  "query": {
    "range": {
      "<numeric_or_date_field>": {
        "gte": 10,   # または "2025-01-01"
        "lte": 20    # または "2025-12-31"
      }
    }
  }
}


```

4. URIパラメータでの簡易検索

```http
GET /<index>/_search?q=artist:KISS
```

### level5

### 5-1.明示的マッピング

事前に構造を定義してインデックス作成。安全性・検索精度に優れ、型変更不可という制約あり。

```http
PUT /fruit
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "name": { "type": "keyword" },
      "color": { "type": "keyword" },
      "taste": { "type": "text" },
      "sweetness": { "type": "integer" },
      "harvest_date": { "type": "date" }
    }
  }
}


```

#### 5-2. ドキュメント登録

```http
POST /fruit/_doc
{ "name": "apple", "color": "red", "taste": "sweet", "sweetness": 7, "harvest_date": "2025-09-01" }
```

#### 5-3.　検索クエリ

```http
GET /fruit/_search
{
  "query": {
    "term": { "color": "red" }
  }
}
```

## とりま試してみる

デフォルトで用意されているサンプルデータを投入してみます。「Add sample data」をクリック。

<https://dev.classmethod.jp/articles/how-to-build-opensearch-with-docker/>

## 役割の違い

🔹 OpenSearch
データの保存・検索・分析をする「検索エンジン本体」。
REST API(curlやクライアントSDK)で直接操作可能。
インデックス作成やクエリ実行など、根幹の機能を担う。

🔹 OpenSearch Dashboards

OpenSearchに接続してGUIで操作するためのフロントエンド。
データの可視化、ダッシュボード作成、アラート管理などを提供。
本体にはデータを保存しない（検索エンジンではなくUIサーバー）。
例えると「MySQL(DB本体)とphpMyAdmin(管理UI)」みたいな関係。

## 今日やったこと

```markdown
# OpenSearch ローカル環境検証レポート

## 1. 環境構築
- Docker を用いて OpenSearch / OpenSearch Dashboards をローカルに起動  
- クラスターの稼働状況を `/_cluster/health` で確認し、正常稼働を確認済み  

---

## 2. 基本的なデータ操作
- **インデックス作成**：任意のインデックスを作成し、利用開始可能な状態に設定  
- **ドキュメント登録**：単一の登録と、`_bulk` API による複数件の一括投入を実施  

---

## 3. マッピング検証
- **動的マッピング**：投入データに応じて自動的に型が割り当てられる挙動を確認  
- **明示的マッピング**：フィールド型を指定して作成し、検索や集計精度への影響を確認  

---

## 4. 検索クエリの実行
- **全件取得**：`match_all`を利用してインデックス全体を検索  
- **キーワード検索**：`match`クエリで部分一致・全文検索を確認  
- **範囲検索**：日付や数値レンジでの絞り込みを実施  
- **URLパラメータでの簡易検索**：`q=keyword`を付与し、シンプル検索を実行  

---

## 5. 操作方法
- ダッシュボードを利用せず、**curlコマンド**で実行し、REST APIの挙動を直接確認  

```

## Rubyで検索をかける方法

<https://jacoyutorius.hatenablog.com/entry/2022/12/24/074336>

## リアルタイムをイメージした内容

リアルタイム監視をイメージした「IoTセンサーのデータ」を、OpenSearchに投入する簡易サンプルを作ってみます。
先ほど学んだインデックス作成→マッピング→バルク投入→クエリ検索の流れに沿っています。

1. インデックス作成（時系列データ用）

```bash
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

2. サンプルデータ投入（バルク）

```bash
POST /iot-sensor/_bulk
{ "index": { "_index": "iot-sensor", "_id": "1" } }
{ "device_id": "sensor-001", "timestamp": "2025-09-10T06:00:00Z", "temperature": 25.3, "humidity": 60.2, "status": "ok" }
{ "index": { "_index": "iot-sensor", "_id": "2" } }
{ "device_id": "sensor-001", "timestamp": "2025-09-10T06:01:00Z", "temperature": 25.8, "humidity": 61.0, "status": "ok" }
{ "index": { "_index": "iot-sensor", "_id": "3" } }
{ "device_id": "sensor-002", "timestamp": "2025-09-10T06:01:00Z", "temperature": 29.5, "humidity": 55.1, "status": "alert" }
{ "index": { "_index": "iot-sensor", "_id": "4" } }
{ "device_id": "sensor-001", "timestamp": "2025-09-10T06:02:00Z", "temperature": 26.1, "humidity": 62.5, "status": "ok" }


```

3. 検索クエリ（リアルタイム監視想定）
(a) 直近5分以内のデータを取得。

```bash
GET /iot-sensor/_search
{
  "query": {
    "range": {
      "timestamp": {
        "gte": "now-5m/m"
      }
    }
  },
  "sort": [{ "timestamp": "desc" }]
}

```

(b) 異常（alert）が発生したデバイスを抽出。

```bash
GET /iot-sensor/_search
{
  "query": {
    "term": { "status": "alert" }
  }
}

```

(c) 温度の平均値を1分ごとに集計。

```bash
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

‐ `enable_ttl`: Time To Live(TTL)機能を使用するかどうかを指定します。期限切れのデータは自動的にクリーンアップされます。デフォルト値はfalseです。

TTLについて今は廃止されている模様。
ドキュメント単位。ISMはインデックス単位。

- IoTデータの「リアルタイム監視用→直近7日だけ保持」という構成をILM（OpenSearchではISM: Index State Management）で実現する具体例を書きます。

🔹 ステップ1. ポリシー作成。

「7日経過したら削除」するポリシーを作成します。

これはGUIでやった。

```bash

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
              "min_index_age": "1d"   # 1日経過したらローテーション
            }
          }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": { "min_index_age": "7d" }  # 7日経過で削除
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

```bash
PUT iot-sensor-000001
{
  "aliases": {
    "iot-sensor-write": { "is_write_index": true }
  },
  "settings": {
    "opendistro.index_state_management.policy_id": "iot-data-policy"
  },
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

ステップ2. 最初のインデックス作成。

「エイリアス+ロールオーバー対応」のインデックスを作ります。

```bash

PUT iot-sensor-000001
{
  "aliases": {
    "iot-sensor-write": { "is_write_index": true }
  },
  "settings": {
    "opendistro.index_state_management.policy_id": "iot-data-policy"
  },
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

ステップ3. データ投入。

通常はエイリアスに書き込みます。

```bash

POST iot-sensor-write/_doc
{
  "device_id": "sensor-001",
  "timestamp": "2025-09-10T08:00:00Z",
  "temperature": 25.7,
  "humidity": 61.3,
  "status": "ok"
}


```

ステップ4. 検索。

検索はiot-sensor-*でまとめて取得できます。

```bash

GET iot-sensor-*/_search
{
  "query": {
    "range": {
      "timestamp": { "gte": "now-1d/d" }
    }
  }
}


```
