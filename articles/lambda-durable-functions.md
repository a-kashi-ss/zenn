---
title: "AWS Lambda durable functionsを試してみた (Pythonでparallel/map/retryなど)"
emoji: "🔄"
type: "tech"
topics: ["aws", "lambda", "serverless", "tech", "stepfunctions"]
published: true
published_at: 2025-12-22 05:00
publication_name: "secondselection"
---

## 1. はじめに

2025年12月リリースされた「**AWS Lambda durable functions**」（以下、durable functions）を早速使ってみました。
本機能は、**Lambda関数内に複数ステップのワークフローを記述しながら、長時間の処理や待機を可能にする耐久実行**を実現する機能です。

この記事では、従来のLambdaの**15分の実行時間制限をどのような条件であれば回避できるか**を、簡易なサンプルコードを交えて実際に確認した内容をまとめます。

> 【公式ドキュメント】
> [**AWS Lambda announces durable functions for multi-step applications and AI workflows**](https://aws.amazon.com/jp/about-aws/whats-new/2025/12/lambda-durable-multi-step-applications-ai-workflows/)  Posted on: Dec 2, 2025
> Lambda durable functions are generally available in US East (Ohio) with support for Python (versions 3.13 and 3.14) and Node.js (versions 22 and 24) runtimes. For the latest region availability, visit the AWS Capabilities by Region [page](https://builder.aws.com/build/capabilities).

:::message

### 対象読者

* durable functionsの基本を理解したい方
* LambdaやStep Functionsのワークフローの改善／見直しを考えている方

:::

## 2. durable functionsの特徴

### 2-1. これまでのLambdaの特徴

Lambdaは基本的に **「短時間」** で **「ステートレス（状態を持たない）」** な処理を行うために設計されており、15分の実行時間制限がありました。

これまで、長時間実行や複数ステップにまたがる処理を実装するためには、Lambdaを複数利用できるStep Functionsで実装されている方も多いのではないでしょうか。

### 2-2. durable functionsの主な特徴

1つの関数内で分割された `step` という処理は、各stepが **15 分以内** であれば、従来のLambdaの15分制約を実質的に回避できます。また`wait`で処理を一旦停止したあと再開できます。  

:::message

#### おさえておきたいポイント

* **単一ステップで15分を超える処理はできない**  
  * ただし複数のstepをチェックポイントとして進めることで実行時間の延長が可能。

* **wait を使うことで処理を中断・再開できる**
  * `wait` によって待機ポイントを作成し、再開時には直前のチェックポイントから継続。

:::

### 2-3. 制約・注意点

* サポートランタイムやリージョンが限られている
   ※ 記事執筆時点の情報です

## 3. 今回試したことの概要

### 3-1. 検証シナリオ

今回Durable Functionsの主要な制御フローとライフサイクルについて、
以下の5つを確認しました。

* **step**: ワークフロー制御
* **wait**： 状態の保存と中断
* **parallel**: 複数の処理を「並列」実行し、時間を短縮
* **map**: リスト要素を「分散」させ、動的にデータを処理
* **retry**: エラー発生時に自動で再試行

### 3-2. 操作手順の基本

今回実行した操作の大まかな手順を以下にまとめます。

1. `リージョンを米国東部（オハイオ）`に変更する
2. ランタイムを選択する
3. `Durable execution`を有効化する
4. `関数を作成`ボタンをクリックする
5. Lambda関数を記述し、`Deploy`ボタンをクリック
6. `一般設定`のタイムアウトを設定する（**Step別の最大稼働時間**を設定）
7. `永続実行`の実行タイムアウトを設定する（**Lambda関数のトータルの実行時間**を設定）
8. テスト実行時の呼び出しタイプを`非同期`を選択後、`テスト`ボタンを押下する

![画像](/images/lambda-durable-functions/procedure.drawio.png)

:::message alert

既存の通常Lambdaから切り替えは不可。
新規作成時のみ設定可能です。
(上記の3の手順を忘れて保存した場合、再作成になるのでご注意ください)

:::

## 4. 【step/wait】サーバーレスで「待つ」を実現する

まずは基礎となる`step` と `wait`をまず抑えていきましょう。

* step：Lambda関数の中で実行したい具体的な「仕事の単位」を定義する
* wait：長時間の一時停止とコスト効率化を行う

:::details 📝 waitステート時の料金について。

通常のLambda内で待機をする場合課金が発生し続けますが、
durable functionsの **Waitステート** を使えば、**待機時間は課金対象になりません**。([参照記事](https://docs.aws.amazon.com/lambda/latest/dg/durable-examples.html))

:::
> **【検証内容】**
>
> **15分の制限**について下記4パターンを実施し確認。
> **待機時間(wait)を挟むことで15分を超えて動作する**ことを確認。
> 一方で待機時間を挟まない場合には、タイムアウトが発生することを確認。

| 処理内容                  | 合計時間 | 結果                        |
| --------------------- | ---- | ------------------------- |
| 1: step 2分 → wait 14分    | 16分  | ✅ 問題なく実行できる               |
| 2: step 8分 → wait 1分 を2セット | 18分  | ✅ 問題なく実行できる  |
| 3: step 8分 → step 8分（連続） | 16分  | ❌ タイムアウト発生（1 Step あたりの制限） |

* **サンプルコード(Step/Wait編)**

上記処理内容の`2: step 8分 → wait 1分を2セット`のコードです。

```python
from aws_durable_execution_sdk_python.config import Duration
from aws_durable_execution_sdk_python.context import StepContext, durable_step
from aws_durable_execution_sdk_python.execution import durable_execution
import time

@durable_step
def my_step(step_context: StepContext, my_arg: int) -> str:
    step_context.logger.info("Hello from my_step")
    time.sleep(480)
    return f"from my_step: {my_arg}"

@durable_execution
def lambda_handler(event, context) -> dict:
    steps_config = [123, 456]
    msg = "" 
    for arg in steps_config:
        msg = context.step(my_step(arg))        
        context.wait(Duration.from_seconds(60))
    return {
        "statusCode": 200,
        "body": msg,
    }
```

![画像](/images/lambda-durable-functions/test_basic.drawio.png)

## 5. 【parallel】複数の処理を「並列」に走らせる

parallelは「あらかじめ決まった数」の処理を並列化して実行します。

> **【検証内容】**
>
> 処理時間が異なる3つの処理を行いました（5秒／3秒／2秒）。
> 3つの処理が並列に開始され、最も長い処理（5秒）に合わせて終了することを確認しました。

* **サンプルコード(parallel編)**

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

:::message

並列実行を行う件数が増えたらどうなるか、確認してみました。
parallelで実行するstep数を**100件**にして実行した結果、**9秒**で処理ができました！

（上記のサンプルコードをもとに、実行時間1～5秒を20セットで実行した結果です）

| 項目       | 内容                          |
|------------|----------------------------------|
| 開始日     | 2025年12月11日 14:57:16.539 (UTC+09:00) |
| 終了日     | 2025年12月11日 14:57:25.853 (UTC+09:00) |
| 所要時間   | 9秒314ms                        |

:::

## 6. 【map】動的なリストを「分散」処理する

mapは「リストのデータ数」に応じて動的に処理を並列化します。

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

:::message

リストの要素数が増えたらどうなるか、確認してみました。
リストを**100個**にして実行した結果、**4秒**で処理ができました！

（上記のサンプルコードを`items = list(range(1, 101))`に更新して実行した結果です）

| 項目       | 内容                          |
|------------|----------------------------------|
| 開始日     | 2025年12月11日 14:28:54.187 (UTC+09:00) |
| 終了日     | 2025年12月11日 14:28:58.774 (UTC+09:00) |
| 所要時間   | 4秒587ms                        |

:::

※ 注意:上記画像の通りすべて成功と表示されていますが、なぜか詳細ステータスにはfailedと表示されます。同内容でNode.jsで書き直して実行したところ、successとなることは確認しました。

## 7. 【retry】エラーに強いワークフローを作る

エラーが起きたら、少し待ったあとに **自動でリトライ** させることが可能です。

:::details 📝 tips : 指数バックオフについて。

今回サンプルコードを書くなかで、**指数バックオフ**という考え方を学びました。

これは、処理が失敗した場合にすぐに再試行するのではなく、
「1秒後 → 2秒後 → 6秒後 → …」
といった形で、試行間隔を徐々に伸ばしながらリトライする手法です。

システムの一時的な障害が解消されるまで適度に待ちながら再試行することで、
復旧の成功率を高めることができるようなので、取り入れてみました。

:::

> **【検証内容】**
>
> 3回までリトライを許容。2回エラー発生後、3回目に処理が成功することを確認。

* **サンプルコード(retry編)**

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

## 8. おわりに

AWS re:Invent 2025において、Step Functionsとの使い分けについて触れられたシーンがありました。今後AIの技術が高度化していくなかでは、**コードベースで定義できる本機能が優位になる可能性がある**と述べられていました。(詳細が気になる方は下記ご確認ください)
@[card](https://www.youtube.com/watch?v=XJ80NBOwsow)

AIと共存できるようにこのような動向も追いかけながら、IaC含めた技術習得も引き続き進めていきます。

## 9. 参考

@[card](https://zenn.dev/aws_japan/articles/lambda-durable-functions)
@[card](https://github.com/aws/aws-durable-execution-sdk-python)
@[card](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
@[card](https://dev.classmethod.jp/articles/aws-lambda-durable-functions-awsreinvent/)
