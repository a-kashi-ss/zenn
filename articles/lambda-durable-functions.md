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
  * 後述する `wait()` による待機中には、
    コンピュート料金が発生しないというメリットがあります。

### 制約・注意点

* サポートランタイムやリージョンが限られている（※記事執筆時点）
* 「1回の関数呼び出し」の上限実行時間は通常のLambdaと同様。
   時間処理は `step` や `wait` を利用して分割する必要あり。

### テスト実行までの流れ

1. リージョンを米国東部（オハイオ）に変更（※記事執筆時点ではリージョンに制限あり）
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
    * **Parallel**: 複数の処理を「並列」実行し、時間を短縮する
    * **Map**: リスト要素を「分散」させ、動的にデータを処理する
    * **Retry**: エラー発生時に自動で再試行する（指数バックオフ等）
2. **状態の保存と中断（Wait）**
    * 処理を中断し、指定時間待機する（この間、課金は発生しない）
3. **プロセスの再開（Resume）**
    * 待機時間が経過した後、新しい呼び出しとして処理が再開される挙動を確認する

:::message alert

既存の通常Lambdaから切り替えるは不可。
新規作成時のみ設定可能です。

:::

## 3. 【step/wait】サーバーレスで「待つ」を実現する

まずは基礎となる`step` と `wait`をまず抑えていきましょう。

* step：Lambda関数の中で実行したい具体的な「仕事の単位」を定義
* wait：長時間の一時停止とコスト効率

:::details 📝 waitステート時の料金について。

通常のLambda内で待機をする場合課金が発生し続けますが、
durable functionsの **Waitステート** を使えば、**待機時間は課金対象になりません**。([参照記事](https://docs.aws.amazon.com/lambda/latest/dg/durable-examples.html))

:::
> **【検証内容】**
>
> 15分の制限について下記4パターンを実施。
> 待機時間を挟むことで制限を超えて動作することを確認。
> 一方で待機時間を挟まない場合、タイムアウトが発生することを確認。

| 処理内容                  | 合計時間 | 結果                        |
| --------------------- | ---- | ------------------------- |
| 1: Step 2分 → Wait 14分    | 16分  | ✅ 問題なく実行できる               |
| 2: Step 1分 → Wait 7分 を2セット | 16分  | ✅ 問題なく実行できる  |
| 3: Step 8分 → Step 8分（連続） | 16分  | ❌ タイムアウト発生（1 Step あたりの制限） |

* **サンプルコード(Step/Wait編)**

```python
from aws_durable_execution_sdk_python.config import Duration
from aws_durable_execution_sdk_python.context import StepContext, durable_step
from aws_durable_execution_sdk_python.execution import durable_execution
import time

@durable_step
def my_step(step_context: StepContext, my_arg: int) -> str:
    step_context.logger.info("Hello from my_step")
    time.sleep(60)
    return f"from my_step: {my_arg}"

@durable_execution
def lambda_handler(event, context) -> dict:
    steps_config = [123, 456]
    msg = "" 
    for arg in steps_config:
        msg = context.step(my_step(arg))        
        context.wait(Duration.from_seconds(420))
    return {
        "statusCode": 200,
        "body": msg,
    }
```

![画像](/images/lambda-durable-functions/test_basic.drawio.png)

## 4. 【Parallel】複数の処理を「並列」に走らせる

Parallelは「あらかじめ決まった数」の処理を並列化して実行します。

> **【検証内容】**
>
> 処理時間が異なる3つの処理を行いました（5秒／3秒／2秒）。
> 3つの処理が並列に開始され、最も長い処理（5秒）に合わせて終了することを確認しました。

* **サンプルコード(Parallel編)**

```python
from aws_durable_execution_sdk_python import durable_execution, DurableContext
import time

@durable_execution
def lambda_handler(event: dict, context: DurableContext) -> dict:

    def call_step1(ctx: DurableContext):
        return ctx.step(lambda _: (time.sleep(5), {"id": "1"})[1], name="my_step1")

    def call_step2(ctx: DurableContext):
        return ctx.step(lambda _: (time.sleep(3), {"id": "2"})[1], name="my_step2")

    def call_step3(ctx: DurableContext):
        return ctx.step(lambda _: (time.sleep(2), {"id": "3"})[1], name="my_step3")

    batch = context.parallel([call_step1, call_step2, call_step3], name="parallel_demo")
    return {
        "my_step1": batch.all[0].result,
        "my_step2": batch.all[1].result,
        "my_step3": batch.all[2].result,
    }
```

![画像](/images/lambda-durable-functions/test_parallel.drawio.png)

## 5. 【Map】動的なリストを「分散」処理する

Mapは「配列（リスト）のデータ数」に応じて動的に処理を並列化します。

> **【検証内容】**
>
> データ数3のリストの処理が並列で実施されることを確認。

* **サンプルコード(Map編)**

```python
from aws_durable_execution_sdk_python import (
    DurableContext,
    durable_execution,
    BatchResult,
)

def square(context: DurableContext, item: int, index: int, items: list[int]) -> int:
    return item * item

@durable_execution
def lambda_handler(event: dict, context: DurableContext) -> BatchResult[int]:
    items = [1, 2, 3]
    result = context.map(items, square)
    return result
```

![画像](/images/lambda-durable-functions/test_map.drawio.png)

## 6. 【Retry】エラーに強いワークフローを作る

エラーが起きたら、少し待ってから **自動でリトライ** させることが可能です。

今回サンプルコードを書くなかで、Exponential Backoff（指数バックオフ）という考え方を学びました。復旧の可能性を高めるために「失敗したら1秒後に再試行、次は2秒後、その次は4秒後…」というように、間隔を空けながらリトライする処理を実装しました。

> **【検証内容】**
>
> 3回までリトライを許容。2回エラー発生後、3回目に処理が成功することを確認。

* **サンプルコード(Retry編)**

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
    if attempt_count == 1 or attempt_count == 2:
        raise Exception("Fail on first attempt for retry test")
    return f"from my_step: {my_arg}"

def retry_strategy(error, attempt_count: int):
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

![画像](/images/lambda-durable-functions/test_map.drawio.png)

## 7. おわりに

Step Functionsとの使い分けについて様々な考察がありますが、下記のサイトでは今後AIとの共存のうえでは、コードベースで定義できる本機能が優位になる可能性があると述べられていました。
@[card](https://www.youtube.com/watch?v=XJ80NBOwsow)

様々AIとの共存に関するニュースが取り上げられていますので、動向を見ながら引き続き技術習得を進めていきます。

## 8. 参考

@[card](https://zenn.dev/aws_japan/articles/lambda-durable-functions)
@[card](https://github.com/aws/aws-durable-execution-sdk-python)
@[card](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
@[card](https://dev.classmethod.jp/articles/aws-lambda-durable-functions-awsreinvent/)
