---
title: "OpenSearch超入門：Docker Composeで構築して検索・マッピング・ISMまで一気に体験"
emoji: "🔎"
type: "tech"
topics: ["opensearch", "docker", "elasticsearch", "search"]
published: false
---
## 1. はじめに

OpenSearchはElasticsearchから派生したオープンソース検索エンジンです。
フルテキスト検索や集計、可視化ができるプラットフォームです。
この記事ではDocker Composeでローカル環境を構築し、検索とデータ登録を体験します。
さらにマッピングとISM(Index State Management)も触れて、実運用の入り口を学びます。

対象はこれからOpenSearchを学びたい初心者です。
コマンドとJSONの形が分かるように、最小限の例で説明します。

## 2. 環境構築

ComposeでOpenSearch2ノードとDashboardsを起動します。
メモリ割当は`-Xms512m -Xmx512m`に固定します。
データはボリュームに永続化します。

```yml
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

起動手順は次のとおりです。

```bash
docker compose up -d
curl http://localhost:9200
```

Dashboardsは`http://localhost:5601/`で開けます。

## 3. インデックスとドキュメント登録

まずは単一ドキュメント登録です。
`<index>`と`<id>`は任意名です。

<!-- 
📝この辺もサンプルデータとは異なるものを勝手にデータを作っているっぽい
```http
PUT /sample/_doc/1
{ "title": "hello", "tags": ["note"], "count": 1 }
``` -->

次に複数ドキュメントの一括登録です。
`_bulk`は1行1コマンドで書きます。

<!-- 📝この辺もサンプルデータとは異なるものを勝手にデータを作っているっぽい
```http
POST /sample/_bulk
{ "index": { "_id": "2" } }
{ "title": "bulk-1", "tags": ["bulk"], "count": 10 }
{ "index": { "_id": "3" } }
{ "title": "bulk-2", "tags": ["bulk"], "count": 20 }
``` -->

### 3.1 動的マッピングと明示的マッピング

動的マッピングは投入データから型を自動推定します。
素早く始められますが、後から型の変更が難しいです。

明示的マッピングは最初に型を決めます。
検索の精度や集計の安定性が高まります。

<!-- 📝この辺もサンプルデータとは異なるものを勝手にデータを作っているっぽい
```http
PUT /product
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "name": { "type": "keyword" },
      "desc": { "type": "text" },
      "price": { "type": "float" },
      "released_at": { "type": "date" }
    }
  }
}
``` -->

## 4. 検索実験

代表的な検索を3種類試します。

### 4.1 全件取得

```http
GET /sample/_search
{ "query": { "match_all": {} } }
```

### 4.2 キーワード検索

<!-- 
📝この辺もサンプルデータとは異なるものを勝手にデータを作っているっぽい
```http
GET /sample/_search
{
  "query": {
    "match": { "title": "hello" }
  }
}
``` -->

### 4.3 範囲検索

<!-- 
📝この辺もサンプルデータとは異なるものを勝手にデータを作っているっぽい
```http
GET /sample/_search
{
  "query": {
    "range": { "count": { "gte": 10, "lte": 30 } }
  }
}
``` -->

レスポンス例はDashboardsのDev Toolsや`curl`で確認できます。
フィールドの型に合ったクエリを選ぶことが大切です。

## 5. ダッシュボード作成

Dashboardsを開いて、次の順に試します。
<!-- 
📝この部分は探り探りやっていて、メモに残せていなかったので削除
### 5.1 Index Pattern登録

Index Patternsで`sample`を登録します。

### 5.2 Discoverで確認

Discoverで投入データを検索して確認します。

### 5.3 Visualizeで可視化

Visualizeでチャートを作成し、ダッシュボードに配置します。

可視化の例は次のとおりです。

- ドキュメント数の折れ線グラフ。
- タグ別件数の棒グラフ。
- 検索応答時間のメトリクス。

ダッシュボードに並べると全体像を把握しやすくなります。 -->

## 6. ISMでデータ寿命を管理

TTLフィールドは廃止されました。
代わりにISMでインデックスのライフサイクルを管理します。
ここでは1日でロールオーバーし、7日で削除する例を示します。

ポリシー定義は次のとおりです。

```json
{
  "policy": {
    "description": "rollover daily, delete after 7 days",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [ { "rollover": { "min_index_age": "1d" } } ],
        "transitions": [
          { "state_name": "delete", "conditions": { "min_index_age": "7d" } }
        ]
      },
      { "name": "delete", "actions": [ { "delete": {} } ] }
    ]
  }
}
```

作成したポリシーを最初のインデックスに適用します。
書き込みはエイリアスに向けます。

```http
PUT /logs-000001
{
  "aliases": { "logs-write": { "is_write_index": true } },
  "settings": {
    "opendistro.index_state_management.policy_id": "my-policy"
  },
  "mappings": {
    "properties": {
      "message": { "type": "text" },
      "level": { "type": "keyword" },
      "ts": { "type": "date" }
    }
  }
}
```

データ投入と検索は次のとおりです。

<!-- これも読み込ませてない...
```http
POST /logs-write/_doc
{ "message": "started", "level": "info", "ts": "2025-09-10T00:00:00Z" }

GET /logs-*/_search
{ "query": { "range": { "ts": { "gte": "now-1d/d" } } } }
``` -->

<!-- ロールオーバー後は`logs-000002`が自動作成されます。これも私の予測だけで検証不足 -->
7日後に古いインデックスが削除されます。

## 7. まとめ

初心者でもComposeで簡単に検索環境を作れます。
単一登録とBulkで投入を覚え、検索の基本3種を体験しました。
運用では明示的マッピングとISMの導入が有効です。
次はアプリからの接続や、可視化の高度化に挑戦しましょう。
