---
title: "AWSサーバーレス入門｜StepFunctionsの基本からIaCで最小構成でデプロイまで"
emoji: "📶"
type: "tech"
topics: ["aws","stepfunctions","serverlessframework","iac","初心者"]
published: true
published_at: 2025-09-29 06:00
publication_name: "secondselection"
---

## 1. はじめに

今回はAWSのサーバーレスアーキテクチャをGUIから始めてIaCで構築しました。
Step Functionsをはじめて使用して学んだ基本から、Serverless Frameworkでのデプロイ手順までまとめました。
この記事が私と同じようなAWS初心者の方にとって有益な情報となれば幸いです。

**💡 対象読者**
・AWSの基本（LambdaやS3, DynamoDB, ポリシーなど）を把握しており、Step Functionsを体験したい人。
・サーバーレスアーキテクチャに興味のある方。

**💡 この記事を通して得られること**
・Step Functionsのステートマシン設計の基本（Task, Choice, Wait, Parallelなど）の理解。
・Serverless Frameworkを使って、最小構成でStep Functionsをデプロイする。

### 今回作成したワークフロー

#### 《StepFunctions》

⽇本語のテキストを英訳化・⾳声化するサーバーレスなワークフローを作成しました。
![画像](/images/begginer-aws-sfn/handson_sfn.drawio.png)

#### 《Serverless Framework》

Lambda関数を1つ呼び出して終了する直列処理(左図)と、入力値によって分岐する並行処理(右図)を作成しました。

![画像](/images/begginer-aws-sfn/serverlessframework.drawio.png)

## 2. AWS Step Functionsについて

### 2-1. AWS Step Functionsとは

> ・ サーバーのプロビジョニング/管理なしで、ワークフローを定義実⾏できるサービス。
> ・ ステートマシーンという単位で各種状態を定義し、ワークフローを作成できる。
>
> (参照元:[AWS Hands-on for Begginers](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-StepFunctions-2022-reg-event.html?trk=aws_introduction_page))

AWSのサーバレスサービスの1つで、複数のサービスを組み合わせて、処理の順序・分岐・並列処理・待機などを含むワークフローを定義できます。
コンソール上での操作方法は、視覚的にノードをブロックのようにつなぎ合わせ、ステートマシン（ワークフローの定義）を作成していきます。

- **ステート**
  - ワークフローを構成する個々のステップ
  - 各ステートの役割

    | ステート      | 役割                   |
    | ------------ | ----------------------- |
    | **Task**     | 他のAWSサービスを呼び出して処理する |
    | **Choice**   | 条件に応じて処理を分岐させる        |
    | **Parallel** | 複数の処理を同時並行で実行する      |
    | **Wait**     | 指定した時間まで待つ       |
    | **Pass**     | そのまま次に渡す  |

- **ステートマシン**
  - ワークフロー全体のことで、「**Amazon States Language（以下ASL）**」というJSONベースの言語で定義します。

### 2-2. 基本

アクション・フロー、変数の使用方法について説明します。

#### 《アクション・フロー》

- **アクション**  
  各AWSサービスのAPIを呼び出す機能です。  
  例えば「DynamoDBにデータを書き込む」「Lambda関数を実行する」といった処理を選択できます。

- **フロー**  
  ワークフロー全体の流れを制御する機能です。  
  例えば「条件分岐（Choice）」「並列処理（Parallel）」「待機（Wait）」など、アクションをどのように実行・接続するかを定義します。

使用するアクションやフローが決まれば、ドラッグ&ドロップ操作でワークフローを組み立てます。

![画像](/images/begginer-aws-sfn/handson_sfn_basic1.drawio.png)

#### 《変数》

ステートマシンのアクションのパラメーターが可変値である場合、記載ルールは以下の通りです。

- 参照する入力値を、先頭に `$.`をつける。例:`$.user.id`
- パラメータのキー名に `. $`を付ける。例： `"UserId.$"`: "$.user.id"

:::message  
**ステートマシンのクエリ言語**

ステートマシンのクエリ言語は、`JSONata`と`JSONPath`の2種類があります。
今回は`JSONPath`を採用して、変数を使用しています。

下記の記事にて、ステートマシンのクエリ言語について紹介されています。
@[card](https://zenn.dev/dannykitadani/articles/b0ea91d3139823)

:::

:::details  📝 DynamoDBで変数を参照した際の具体例をご紹介します。

- DynamoDBのArticleテーブルがあり、キーとしてAriticleIDが文字列`S`(sentenceの頭文字)として存在します。
![画像](/images/begginer-aws-sfn/handson_sfn_dynamodb.drawio.png)

- ステートマシンで入力されたArticleIDの入力値を、そのままDynamoDBのキーとして使うために、APIパラメータのキー部分に`"S.$": "$.ArticleID"`を入力値として埋め込みます。

```json
// DynamoDB GetItemのAPIパラメータ
{
  "TableName": "Article",
  "Key": {
    "ArticleID": {
      "S.$": "$.Article.ID"
    }
  },
  {}
}

```

- コンソール上での実行結果を確認しました。

```json
// 入力
{
 "Article": {
   "ID": "0001"
 }
}

// 出力 (DynamoDBにArticleIDが0001で登録されている情報が出力されます。)
{
  "Item": {
    "ArticleID": {
      "S": "0001"
    },
    "Detail": {
      "S": "AWS Step Functions は、AWS のサービスのオーケストレーション、ビジネスプロセスの自動化、サーバーレスアプリケーションの構築に使用されるローコードの視覚的なワークフローサービスです。ワークフローは、障害、再試行、並列化、サービス統合、可観測性などを管理するため、デベロッパーはより価値の高いビジネスロジックに集中することができます。"
    }
  },
  {省略}
}
```

:::

### 2-3. 作成したワークフロー・手順

ハンズオンの案内に沿って、⽇本語のテキストを英訳化・⾳声化するサーバーレスなワークフローを作成しました。
![画像](/images/begginer-aws-sfn/handson_sfn_workflow.drawio.png)*[参照元:AWS Hands-on for Begginers AWS Step Functions入門](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-StepFunctions-2022-reg-event.html?trk=aws_introduction_page)*

:::details  📝 各サービスの概要は、こちらからご確認ください。

> #### Amazon DynamoDB
>
> - フルマネージドの **NoSQL データベース** サービス  
> - 高い信頼性、性能要件に応じたテーブルごとのキャパシティ設定が可能  
>
> #### Amazon Simple Storage Service（Amazon S3）
>
> - **99.999999999％の耐久性** を持つ **オブジェクトストレージ** サービス
> - 容量無制限、安価なストレージ、様々なAWSサービスと連携
>
> #### Amazon Translate
>
> - 高速かつ高品質な **言語翻訳** サービス  
>
> #### Amazon Polly
>
> - **テキストを音声に変換** するサービス
> - 日本語、英語をはじめ、複数の言語をサポート
>
> ![画像](/images/begginer-aws-sfn/handson_sfn_resorce.drawio.png)*[参照元:AWS Hands-on for Begginers AWS Step Functions入門](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-StepFunctions-2022-reg-event.html?trk=aws_introduction_page)*

:::

:::details  📝 ハンズオンの手順・詳細については、こちらをご確認ください。

```text
(1) ステートマシーンを作成 + 「アクション」を使ってみる
(2) Input の受け取り + Choice ステートを使ってみる
(3) Parallel ステートで処理を並列に実行する
(4) Output の調整 + DB（Amazon DynamoDB）の更新
(5) Parallel ステートのもう一方の実装 - テキストを音声化する
(6) ステートで「タスクが終わるまで待つ」を実装する
(7) 音声ファイルの出力先情報を DynamoDB テーブルに格納する
```

(1)～(5)までの流れについて、下記のサイトで詳しく紹介されています。
(6)(7)については、(5)までの流れを応用すれば、対応可能です。
@[card](https://cloud5.jp/tryawsstepfunctions/)

💡 変数の参照時にエラーが出た場合は、下記の画像を参考にしてステートマシーンのコードを編集して保存してください。

- 1.States内のArgumentsをParametersに変更
- 2.QueryLanguageをJSONataからJSONPathに変更

![画像](/images/begginer-aws-sfn/handson_sfn_setting.drawio.png)

:::

#### 《今回作成したステートマシーンのASL》

上記のハンズオンで構成を作成した結果、下記のステートマシンが完成しました。

![画像](/images/begginer-aws-sfn/handson_sfn.drawio.png)

:::details  📝 ASLのJSONファイルは、こちらをご確認ください。

```json
{
  "Comment": "A description of my state machine",
  "StartAt": "DynamoDB GetItem",
  "States": {
    "DynamoDB GetItem": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:dynamodb:getItem",
      "Parameters": {
        "Key": {
          "ArticleID": {
            "S.$": "$.ArticleID"
          }
        },
        "TableName": "Article"
      },
      "Next": "Choice-Item Is Present"
    },
    "Choice-Item Is Present": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.Item",
          "IsPresent": true,
          "Next": "Parallel"
        }
      ],
      "Default": "Fail"
    },
    "Parallel": {
      "Type": "Parallel",
      "End": true,
      "Branches": [
        {
          "StartAt": "TranslateText",
          "States": {
            "TranslateText": {
              "Type": "Task",
              "Parameters": {
                "SourceLanguageCode": "ja",
                "TargetLanguageCode": "en",
                "Text.$": "$.Item.Detail.S"
              },
              "Resource": "arn:aws:states:::aws-sdk:translate:translateText",
              "Next": "DynamoDB UpdateItem - EnglishVerison",
              "ResultPath": "$.Result"
            },
            "DynamoDB UpdateItem - EnglishVerison": {
              "Type": "Task",
              "Resource": "arn:aws:states:::aws-sdk:dynamodb:updateItem",
              "Parameters": {
                "TableName": "Article",
                "Key": {
                  "ArticleID": {
                    "S.$": "$.Item.ArticleID.S"
                  }
                },
                "UpdateExpression": "SET EnglishVersion = :EnglishVestionRef",
                "ExpressionAttributeValues": {
                  ":EnglishVestionRef": {
                    "S.$": "$.Result.TranslatedText"
                  }
                }
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "StartSpeechSynthesisTask",
          "States": {
            "StartSpeechSynthesisTask": {
              "Type": "Task",
              "Parameters": {
                "OutputFormat": "mp3",
                "OutputS3BucketName": "h4b-stepfunctions-output-20250915",
                "Text.$": "$.Item.Detail.S",
                "VoiceId": "Mizuki"
              },
              "Resource": "arn:aws:states:::aws-sdk:polly:startSpeechSynthesisTask",
              "ResultPath": "$.Result",
              "Next": "GetSpeechSynthesisTask"
            },
            "GetSpeechSynthesisTask": {
              "Type": "Task",
              "Parameters": {
                "TaskId.$": "$.Result.SynthesisTask.TaskId"
              },
              "Resource": "arn:aws:states:::aws-sdk:polly:getSpeechSynthesisTask",
              "ResultPath": "$.Result",
              "Next": "Choice -Task is completed"
            },
            "Choice -Task is completed": {
              "Type": "Choice",
              "Choices": [
                {
                  "Variable": "$.Result.SynthesisTask.TaskStatus",
                  "StringMatches": "completed",
                  "Next": "DynamoDB UpdateItem -mp3 URL"
                }
              ],
              "Default": "Wait"
            },
            "DynamoDB UpdateItem -mp3 URL": {
              "Type": "Task",
              "Resource": "arn:aws:states:::dynamodb:updateItem",
              "Parameters": {
                "TableName": "Article",
                "Key": {
                  "ArticleID": {
                    "S.$": "$.Item.ArticleID.S"
                  }
                },
                "UpdateExpression": "SET S3URL = :S3URLRef",
                "ExpressionAttributeValues": {
                  ":S3URLRef": {
                    "S.$": "$.Result.SynthesisTask.OutputUri"
                  }
                }
              },
              "End": true
            },
            "Wait": {
              "Type": "Wait",
              "Seconds": 5,
              "Next": "GetSpeechSynthesisTask"
            }
          }
        }
      ]
    },
    "Fail": {
      "Type": "Fail"
    }
  },
  "QueryLanguage": "JSONPath",
  "TimeoutSeconds": 30
}

```

:::

## 3. Serverless Frameworkについて

前項では、Step Functionsを用いたワークフローを紹介しました。  
Step FunctionsのワークフローをIaC(Infrastructure as Code)でAWS環境へデプロイできるツールが **Serverless Framework** です。

:::details  📝 IaCとは？

IaCとは**Infrastructure as Code**の略称です。具体的にはインフラ構築の内容をコード化して環境構築を自動化する手法で、主なメリットは以下の通りです。

- 手作業による人為的なミスを削減できる。
- 同じ環境を複製する際の作業時間を短縮できる。
- Webアプリケーション開発と同様にコードのバージョン管理ができる。

:::

### 3-1. Serverless Frameworkとは

Serverless Frameworkは、サーバーレスアプリケーションの開発とデプロイをサポートするオープンソースのフレームワークです。クラウドリソースの構成を含めてコードで管理できます。

### 3-2. 作成したワークフロー

- Step1(左図)
  - Lambda関数を1つ呼び出して終了する単純な構成のTask型ステートマシン
- Step2(右図)
  - 入力値に応じて分岐するChoiceステートを持つステートマシン

![画像](/images/begginer-aws-sfn/serverlessframework.drawio.png)

### 3-3. 手順

- AWS CDKの環境構築が完了していることを前提に、作業の流れを説明します。

:::details 📝 今回使用したツールのバージョンの情報はこちらです。

```bash
 
# Python: 実行環境として利用
$ python3 -V
> Python 3.11.2

# Node.js: Serverless Framework自体や関連パッケージが依存する実行環境
$ node -v
> v20.19.4

# Serverless Framework: AWSリソースをデプロイ・管理するためのフレームワーク
$ serverless -v
> Framework Core: 3.40.0
> Plugin: 7.2.3
> SDK: 4.5.1

# AWS CLI: AWSの各種サービスを操作するための公式コマンドラインツール
$ aws --version
> aws-cli/2.30.0 Python/3.13.7 Linux/5.15.153.1-microsoft-standard-WSL2 exe/x86_64.debian.12

```

:::

#### (1) Serverless Frameworkインストール

```bash
# Serverless Frameworkインストール
$ npm install -g serverless@3
```

:::message

**Serverless Frameworkのバージョンについて**

通常`npm install -g serverless`で、最新版(記事執筆時はv4)をインストールができます。

注意点としては、v4からはアカウント登録が必須で、かつ利用コストが発生します。
（詳細は[こちら](https://www.serverless.com/blog/serverless-framework-v4-a-new-model)をご参照ください）

今回は検証目的のため、旧バージョンであるv3を指定(`serverless`のうしろに`@3`を追加)して、インストールしています。

:::

#### (2) バージョン確認

Serverless Frameworkがインストールされたことを確認します。

```bash
# バージョン確認
$ serverless -v
```

#### (3) サービスを作成

複数のリソースを管理できるサービスを作成します。

- サービス用のディレクトリ名:TestStepFunctions
- サービス名:StepFunctionsService
- ランタイム:python3

```bash
# サービス作成
$ serverless create --template aws-python3 --name StepFunctionsService --path TestStepFunctions
```

コマンドを実行すると、カレントディレクトリにフォルダとファイルが自動で作成されました。

```text
TestStepFunctions
├── .serverless
│   ├── cloudformation-template-create-stack.json
│   ├── cloudformation-template-update-stack.json
│   ├── serverless-state.json
│   └── StepFunctionsService.zip
├── .gitignore
├── handler.py
└── serverless.yml
```

#### (4) サービスの内容を設定

Serverless Frameworkの定義ファイルである`serverless.yml`を更新します。

##### 【step1の場合】

Lambda関数を1つ呼び出して終了する単純な構成のステートマシンを作成します。

:::details  📝 step1のサンプルコード。

```yml
service: StepFunctionsService

plugins:
  - serverless-step-functions

provider:
  name: aws
  runtime: python3.12
  region: ap-northeast-1

functions:
  Function1:
    name: TestFunction
    handler: handler.hello

stepFunctions:
  stateMachines:
    StateMachine1:
      name: TestStateMachine
      definition:
        StartAt: HelloJapan
        States:
          HelloJapan:
            Type: Task
            Resource:
              Fn::GetAtt: [Function1LambdaFunction, Arn]
            End: true

```

:::

##### 【step2の場合】

入力値に応じて分岐するChoiceステートを持つステートマシンを作成します。

:::details  📝 step2のサンプルコード。

```yml

service: StepFunctionsService

plugins:
  - serverless-step-functions

provider:
  name: aws
  runtime: python3.12
  region: ap-northeast-1

functions:
  Function1:
    name: TestFunction
    handler: handler.hello

stepFunctions:
  stateMachines:
    StateMachine1:
      name: TestStateMachine
      definition:
        StartAt: ChoiceState
        States:
          ChoiceState:
            Type: Choice
            Choices:
              - Variable: "$.input"
                StringEquals: "error"
                Next: Fail
            Default: HelloWorld
          HelloWorld:
            Type: Task
            Resource:
              Fn::GetAtt: [Function1LambdaFunction, Arn]
            Next: Succeed
          Fail:
            Type: Fail
          Succeed:
            Type: Succeed
```

:::

#### (5) デプロイ

```bash
# デプロイ
$ sls deploy
```

#### (6) ステートマシンの実行・動作確認

```bash
# ステートマシン呼出し(step1の場合)
$ sls invoke stepf --name StateMachine1

# ステートマシン呼出し(step2のinput内容を確認する場合)
$ sls invoke stepf --name StateMachine1 --data '{"input":"XXXXX"}'
```

- CLIでの確認例(step1の場合)

  ![画像](/images/begginer-aws-sfn/serverlessframework_step1_cli.drawio.png)

- GUIでの確認例(step1の場合)

  ![画像](/images/begginer-aws-sfn/serverlessframework_step1_gui.drawio.png)

## 4. おわりに

仮想サーバーを手動で設定・管理できるサービスからはじめて、今回サーバーレス×IaCに取り組みました。
AWS CDKも少し触ってみたので、今後の意気込みとしては[AWS SAM](https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/what-is-sam.html)も実際に触ってそれぞれのメリット・デメリット・特徴を理解していきたいと考えています。

最後までご覧いただき、ありがとうございました。

## 5. 参考

@[card](https://pages.awscloud.com/JAPAN-event-OE-Hands-on-for-Beginners-StepFunctions-2022-reg-event.html?trk=aws_introduction_page)
@[card](https://pages.awscloud.com/rs/112-TZM-766/images/JAPAN_FIELD_WEBINAR_Hands-on-for-Beginners-Step-Functions_2022.pdf)
@[card](https://dev.classmethod.jp/articles/easy-deploy-of-lambda-with-serverless-framework/)
@[card](https://dev.classmethod.jp/articles/serverless-framework-step-functions/)
