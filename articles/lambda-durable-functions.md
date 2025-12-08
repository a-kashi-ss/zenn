---
title: "AWS Lambda durable functions (Step/Wait/Parallel/Map/Retry)"
emoji: "🍀"
type: "tech"
topics: ["aws", "lambda", "serverless", "tech"]
published: true
published_at: 2025-12-22 05:00
publication_name: "secondselection"
---

## 1. はじめに

今月リリースされた「AWS Lambda durable functions」（以下、durable functions）を早速使ってみました。
「15分の実行時間制限が解除された」という点がまず注目されていますが、実際どのような機能なのか、簡易なサンプルコードを作り動作検証を行いましたので、本記事にまとめます。

### 対象読者

* durable functionsの基本を理解したい方
* Step FunctionsやLambdaのワークフローを改善／見直しを考えている方

## 2. durable functionsの特徴

### これまでのLambdaの特徴

Lambdaは基本的に **「短時間」** で **「ステートレス（状態を持たない）」** な処理を行うために設計されており、15分の実行時間制限がありました。

これまで、長時間実行や複数ステップにまたがる処理を実装するためには、Lambdaを複数利用できる **AWS Step Functions** で実装されていた方も多いのではないでしょうか。

### durable functionsの主な特徴

* **コードベースでワークフローを記述可能**
* **自動チェックポイント + 再開（リプレイ）**
  * 状態管理やリトライを自動化することで、運用の簡素化を図れます。
* **最大 1 年の待機**が可能
  * 長時間のワークフローに対応しています。
* **待機中はコンピュート料金が発生しない**
  * 後述する `wait()` による待機中には、コンピュート料金が発生しないというメリットがあります。

### 制約・注意点

* サポートランタイムやリージョンが限られている（※記事執筆時点）
* 「1回の関数呼び出し」の上限実行時間は通常のLambdaと同様。時間処理は `step` や `wait` を利用して分割する必要あり。

### テスト実行までの流れ

1. リージョンを米国東部（オハイオ）に変更（※記事執筆時点ではリージョンに制限があったため）
2. ランタイムを選択する
3. Durable executionを有効化する
4. 設定を保存（Save）する
5. Lambda関数を実装する（検証シナリオ参照）
6. 一般設定のtimeoutを確認する（Step別の最大稼働時間を設定）
7. 永続実行のtimeoutを設定（Lambda関数のトータル時間を設定）
8. テスト実行時の呼び出しタイプを「非同期（Asynchronous）」を選択

#### 検証シナリオ

Durable Functionsの主要な制御フローとライフサイクルについて、以下を確認しました。

1. **ワークフロー制御（Step実行）**
    * **Parallel**: 複数の処理を「並列」実行し、時間を短縮する。
    * **Map**: リスト要素を「分散」させ、動的にデータを処理する。
    * **Retry**: エラー発生時に自動で再試行する（指数バックオフ等）。
2. **状態の保存と中断（Wait）**
    * 処理を中断し、指定時間待機する（この間、課金は発生しない）。
3. **プロセスの再開（Resume）**
    * 待機時間が経過した後、新しい呼び出しとして処理が再開される挙動を確認する。

:::message

既存の通常Lambdaから切り替えるは不可。
新規作成時のみ設定可能です。

:::

## 3. 【Step/Wait】サーバーレスで「待つ」を実現する

基礎となる`step` と `wait`について触れていきます。
通常のLambda内で待機をする場合課金が発生し続けますが、durable functionsの **Waitステート** を使えば、**待機時間は課金対象になりません**。
（標準ワークフローの場合、状態遷移に対して課金されます）。

* 確認事項

15分の制限について下記パターンを実施し、待機時間を挟むことで制限を超えて動作することを確認しました。

```text
- Stepで2分 → Wait14分 = 合計16分
  - 問題なく実行できることを確認
- (Stepで1分 → Wait7分) × 2 = 合計16分
  - 問題なく実行できることを確認
  - 実行中にエラーが解消すれば自動で再実行される
- Stepを8分、連続してStepを8分 = 合計16分
  - タイムアウトが発生する（1ステップあたりの制限）
```

* **サンプルコード(Step/Wait編)**

```python
from aws_durable_execution_sdk_python.config import Duration
from aws_durable_execution_sdk_python.context import DurableContext, StepContext, durable_step
from aws_durable_execution_sdk_python.execution import durable_execution
import time

@durable_step
def my_step(step_context: StepContext, my_arg: int) -> str:
    step_context.logger.info("Hello from my_step")
    time.sleep(60)
    return f"from my_step: {my_arg}"

@durable_execution
def lambda_handler(event, context) -> dict:
    steps_config = [
        (123, "1st"),
        (456, "2nd")
    ]
    
    msg = "" 

    for arg_value, label in steps_config:
        msg = context.step(my_step(arg_value))        
        context.wait(Duration.from_seconds(420))

    return {
        "statusCode": 200,
        "body": msg,
    }

```

## 4. 【Parallel】複数の処理を「並列」に走らせる

Parallelステートは「あらかじめ決まった数」の処理を並列化して実行します。

このステートの確認として、処理時間が異なる3つの処理を行いました。
3つの処理が並列に開始され、最も長い処理（5秒）に合わせて終了することを確認しました。

> (1)2秒 (2)5秒 (3)2秒

* **サンプルコード(Parallel編)**

```python
import time
from aws_durable_execution_sdk_python import durable_execution, DurableContext

@durable_execution
def lambda_handler(event: dict, context: DurableContext) -> dict:
    # 各 API 呼び出し（模擬）をステップ関数にまとめる
    def call_user(ctx: DurableContext):
        return ctx.step(lambda _: (time.sleep(2), {"user_id": "U001", "name": "Taro", "email": "taro@example.com"})[1],
                        name="user_api")

    def call_orders(ctx: DurableContext):
        return ctx.step(lambda _: (time.sleep(5), {"order_id": "O001", "items": ["item1"], "total": 1000})[1],
                        name="orders_api")

    def call_inventory(ctx: DurableContext):
        return ctx.step(lambda _: (time.sleep(2), {"product_id": "P001", "stock": 50})[1],
                        name="inventory_api")

    batch = context.parallel([call_user, call_orders, call_inventory], name="parallel_demo")

    return {
        "user": batch.all[0].result,
        "orders": batch.all[1].result,
        "inventory": batch.all[2].result,
    }
```

## 5. 【Map】動的なリストを「分散」処理する

Parallelに対し、Mapステートは「配列（リスト）のデータ数」に応じて動的に処理を並列化します。

* **サンプルコード(Map編)**

```js
import { withDurableExecution } from '@aws/durable-execution-sdk-js';

// --- Step 処理 ---
async function processItem(item) {
  return {
    id: item.id,
    result: `processed-${item.id}`,
    timestamp: new Date().toISOString(),
  };
}

export const handler = withDurableExecution(async (event, context) => {

  const items = event.items;

  const results = await context.map(
    items,
    (childCtx, item) =>
      childCtx.step(`process-${item.id}`, async () => processItem(item)),
    { maxConcurrency: 5 }
  );

  return { results };
});
```

## 6. 【Retry】エラーに強いワークフローを作る

エラーが起きたら、少し待ってから **自動でリトライ** させるということが叶います。

今回サンプルコードを書くなかで、Exponential Backoff（指数バックオフ）という考え方を学びました。
復旧の可能性を高めるために「失敗したら1秒後に再試行、次は2秒後、その次は4秒後…」というように、間隔を空けながらリトライする処理を実装しました。

* **サンプルコード(Retry編)**

> 3回までリトライを許容。
> 2回意図してエラーを発生させたあと
> 処理が成功することを確認。

```python
import time
from aws_durable_execution_sdk_python.config import Duration, StepConfig
from aws_durable_execution_sdk_python.context import DurableContext, StepContext, durable_step
from aws_durable_execution_sdk_python.execution import durable_execution
from aws_durable_execution_sdk_python.retries import RetryDecision

attempt_count = 0

@durable_step
def my_step(step_context: StepContext, my_arg: int) -> str:
    global attempt_count
    attempt_count += 1

    step_context.logger.info(f"my_step called {attempt_count} times")

    # 1回目は失敗
    if attempt_count == 1:
        raise Exception("Fail on first attempt for retry test")
        # return f"from my_step: {my_arg}"

    # 2回目も失敗
    if attempt_count == 2:
        raise Exception("Fail on first attempt for retry test2")

    return f"from my_step: {my_arg}"

def retry_strategy(error, attempt_count: int):
    """ RetryDecision を返す """
    if attempt_count >= 3:
        return RetryDecision.no_retry()

    delay_seconds = min(2 ** attempt_count, 300)
    return RetryDecision.retry(Duration(seconds=delay_seconds))

@durable_execution
def lambda_handler(event, context: DurableContext) -> dict:

    msg: str = context.step(
        my_step(123),
        name='call-api',
        config=StepConfig(retry_strategy=retry_strategy),
    )

    return {
        "statusCode": 200,
        "body": msg,
    }
```

## 7. おわりに

Step Functionsとの使い分けについて様々な考察がありますが、下記のサイトでは今後AIとの共存のうえでは、コードベースで定義できる本機能が優位になる可能性があると述べられていました。
@[card](https://www.youtube.com/watch?v=XJ80NBOwsow)

様々AIとの共存に関するニュースが取り上げられていますので、動向を見ながら引き続き技術習得を進めていきます。

## 8. 参考

@[card](https://zenn.dev/aws_japan/articles/lambda-durable-functions)
@[card](https://github.com/aws/aws-durable-execution-sdk-python)
@[card](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
@[card](https://dev.classmethod.jp/articles/aws-lambda-durable-functions-awsreinvent/)
