---
title: 未知の環境にAWS CDKをデプロイすることになったら
tags:
  - AWS
  - IaC
  - CDK
  - AWSCDK
  - AI
private: false
updated_at: ''
id: null
organization_url_name: ap-com
slide: false
ignorePublish: false
---
こんにちは。  
AWS CDKを書き続けている[@sori883](https://x.com/sori883)です。  

# はじめに
未知のAWS環境に`cdk deploy`することになった経験はありますか？  
私は何度かあります。  

ドキュメントが整備されておらずよく分からない、でもスケジュールがあるから来週には改修対応してデプロイしなければ行けないという事象は、この業界で時々起こる厄災だと思ってます。  
この記事では未知のAWS環境へデプロイする時に普段していることをまとめます。  

なお、この記事では未知のAWS環境を以下のとおり定義します。  
- 設計書がない、または最新でない
- 構成図がない、または最新でない
- 実機の状態が分からない
- 他に把握している人がいない

# AWS環境を紐解く

## CloudFomationスタックからAWS環境を紐解く
まずはCloudFormationスタックを紐解いていきます。  
スタックにはデプロイ時のテンプレートを参照出来る機能があり、「AWS マネージドコンソール > CloudFormationサービス > 対象スタック > テンプレート」から参照出来ます。  

![cloudformation-copy-template.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/6f22176c-7d7e-4966-a4c1-e1b8597e351d.jpeg)    

このテンプレートや、実リソースを参照しながら、関係者たちとコミュニケーションが取れる資料を最優先で作成します。  
具体的には以下の資料があると技術者、非技術者関係なく会話が楽になります。  

- 構成図
  - 視覚的に分かりやすい
  - リソース間の通信経路が把握できる
- 各リソースのユースケース
  - ECSはバックエンド処理
  - Lambdaはフロントエンド処理

また、解析や資料作成は生成AIにおまかせすると一瞬で終わります。  
以下は私が[過去に投稿した記事](https://qiita.com/sori883/items/a8f022877911966f0300)のスタックをAIに解析させ資料化したものです。  



【指示したプロンプト】  

```
下記はCloudFormationのテンプレートです。
構成図を作成し、構成図.mdに出力してください。
```

<details><summary>【出力結果】（クリックで展開）</summary>

```
                                    ┌─────────────────────────────────────────────────────────────────────────┐
                                    │                              AWS Cloud                                   │
                                    │                          (ap-northeast-1)                                │
┌──────────┐                        │                                                                          │
│          │                        │  ┌─────────────────────────────────────────────────────────────────┐   │
│  Users   │───────────────────────▶│  │                        CloudFront                                │   │
│          │                        │  │                    (Lambda@Edge連携)                             │   │
└──────────┘                        │  └────────────────────────────┬──────────────────────────────────────┘   │
                                    │                               │                                          │
                                    │                               ▼                                          │
                                    │  ┌─────────────────────────────────────────────────────────────────┐   │
                                    │  │              Lambda Function URL (Frontend)                      │   │
                                    │  │                   (Container Image)                              │   │
                                    │  └────────────────────────────┬──────────────────────────────────────┘   │
                                    │                               │                                          │
                                    │  ┌────────────────────────────┼──────────────────────────────────────┐   │
                                    │  │                    VPC (10.0.0.0/16)                              │   │
                                    │  │                                                                    │   │
                                    │  │  ┌─────────────────────────────────────────────────────────────┐ │   │
                                    │  │  │               Public Subnets (ap-northeast-1a/1c)            │ │   │
                                    │  │  │                    10.0.0.0/24, 10.0.1.0/24                  │ │   │
                                    │  │  │                                                               │ │   │
                                    │  │  │  ┌─────────────────┐        ┌────────────────────────────┐  │ │   │
                                    │  │  │  │ Internet Gateway│        │   Bastion (EC2 t3.micro)   │  │ │   │
                                    │  │  │  └─────────────────┘        │       (SSM接続用)           │  │ │   │
                                    │  │  │                             └────────────────────────────┘  │ │   │
                                    │  │  └─────────────────────────────────────────────────────────────┘ │   │
                                    │  │                                                                    │   │
                                    │  │  ┌─────────────────────────────────────────────────────────────┐ │   │
                                    │  │  │              Private Subnets (ap-northeast-1a/1c)           │ │   │
                                    │  │  │                    10.0.2.0/24, 10.0.3.0/24                 │ │   │
                                    │  │  │                                                               │ │   │
                                    │  │  │  ┌─────────────────────────────────────────────────────┐    │ │   │
                                    │  │  │  │              Internal ALB (Application LB)           │    │ │   │
                                    │  │  │  └────────────────────────┬────────────────────────────┘    │ │   │
                                    │  │  │                           │                                   │ │   │
                                    │  │  │                           ▼                                   │ │   │
                                    │  │  │  ┌─────────────────────────────────────────────────────┐    │ │   │
                                    │  │  │  │           ECS Fargate - Agent Service               │    │ │   │
                                    │  │  │  │    (Claude 3 Haiku via Bedrock, Port 8080)          │    │ │   │
                                    │  │  │  │         CPU: 512, Memory: 1024MB                    │    │ │   │
                                    │  │  │  └────────────────────────┬────────────────────────────┘    │ │   │
                                    │  │  │                           │                                   │ │   │
                                    │  │  │  ┌─────────────────────────────────────────────────────┐    │ │   │
                                    │  │  │  │          ECS Fargate - Memory Service               │    │ │   │
                                    │  │  │  │        (SQSからメッセージ処理, 長期記憶生成)          │    │ │   │
                                    │  │  │  │         CPU: 512, Memory: 1024MB                    │    │ │   │
                                    │  │  │  └─────────────────────────────────────────────────────┘    │ │   │
                                    │  │  │                                                               │ │   │
                                    │  │  │  ┌─────────────────────────────────────────────────────┐    │ │   │
                                    │  │  │  │           Neptune (db.t3.medium)                    │    │ │   │
                                    │  │  │  │          (グラフDB - エンティティ関係管理)            │    │ │   │
                                    │  │  │  │              Engine: Neptune 1.3.1.0                │    │ │   │
                                    │  │  │  └─────────────────────────────────────────────────────┘    │ │   │
                                    │  │  └─────────────────────────────────────────────────────────────┘ │   │
                                    │  │                                                                    │   │
                                    │  │  ┌─────────────────────────────────────────────────────────────┐ │   │
                                    │  │  │         Private Endpoint Subnets (ap-northeast-1a/1c)       │ │   │
                                    │  │  │                    10.0.4.0/24, 10.0.5.0/24                 │ │   │
                                    │  │  │                                                               │ │   │
                                    │  │  │  ┌───────────────────────────────────────────────────────┐  │ │   │
                                    │  │  │  │                  VPC Endpoints                        │  │ │   │
                                    │  │  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │  │ │   │
                                    │  │  │  │  │ ECR API  │ │ ECR DKR  │ │ Bedrock  │ │   Logs   │ │  │ │   │
                                    │  │  │  │  │          │ │          │ │ Runtime  │ │          │ │  │ │   │
                                    │  │  │  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │  │ │   │
                                    │  │  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │  │ │   │
                                    │  │  │  │  │  Lambda  │ │   SQS    │ │ DynamoDB │ │    S3    │ │  │ │   │
                                    │  │  │  │  │          │ │          │ │ (Gateway)│ │ (Gateway)│ │  │ │   │
                                    │  │  │  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ │  │ │   │
                                    │  │  │  └───────────────────────────────────────────────────────┘  │ │   │
                                    │  │  │                                                               │ │   │
                                    │  │  │  ┌─────────────────────────────────────────────────────┐    │ │   │
                                    │  │  │  │        OpenSearch (t3.small.search)                 │    │ │   │
                                    │  │  │  │    Domain: conversation-memory                       │    │ │   │
                                    │  │  │  │    (ベクトル検索 - 類似会話検索)                      │    │ │   │
                                    │  │  │  └─────────────────────────────────────────────────────┘    │ │   │
                                    │  │  └─────────────────────────────────────────────────────────────┘ │   │
                                    │  └────────────────────────────────────────────────────────────────────┘   │
                                    │                                                                          │
                                    │  ┌──────────────────────────────────────────────────────────────────┐   │
                                    │  │                      Serverless Services                          │   │
                                    │  │                                                                    │   │
                                    │  │  ┌────────────────────┐  ┌────────────────────────────────────┐  │   │
                                    │  │  │     DynamoDB       │  │          SQS Queue                 │  │   │
                                    │  │  │   (ChatHistory)    │  │    (ShortMemoryQueue)              │  │   │
                                    │  │  │ ・user_id (PK)     │  │  メッセージ保持: 4日               │  │   │
                                    │  │  │ ・session_id (SK)  │──▶│  可視性タイムアウト: 5分          │  │   │
                                    │  │  │ ・ProcessedIndex   │  │                                    │  │   │
                                    │  │  └────────────────────┘  └────────────────────────────────────┘  │   │
                                    │  │                                   │                               │   │
                                    │  │  ┌────────────────────────────────┼───────────────────────────┐  │   │
                                    │  │  │     Lambda (ShortMemoryScan)   │                           │  │   │
                                    │  │  │  ・EventBridge: 5分間隔        │                           │  │   │
                                    │  │  │  ・非アクティブセッションをスキャン                        │  │   │
                                    │  │  │  ・SQSにメッセージ送信                                    │  │   │
                                    │  │  └────────────────────────────────────────────────────────────┘  │   │
                                    │  └──────────────────────────────────────────────────────────────────┘   │
                                    │                                                                          │
                                    │  ┌──────────────────────────────────────────────────────────────────┐   │
                                    │  │                       Amazon Bedrock                              │   │
                                    │  │  ┌────────────────────────────┐  ┌────────────────────────────┐  │   │
                                    │  │  │   Claude 3 Haiku           │  │   Titan Embed Text v2      │  │   │
                                    │  │  │ (anthropic.claude-3-haiku- │  │ (amazon.titan-embed-text-  │  │   │
                                    │  │  │  20240307-v1:0)            │  │  v2:0)                     │  │   │
                                    │  │  │  会話応答生成               │  │  テキスト埋め込み生成       │  │   │
                                    │  │  └────────────────────────────┘  └────────────────────────────┘  │   │
                                    │  └──────────────────────────────────────────────────────────────────┘   │
                                    │                                                                          │
                                    └──────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    リクエストフロー                                         │
└────────────────────────────────────────────────────────────────────────────────────────────┘

    User Request
         │
         ▼
    CloudFront ──────▶ Lambda@Edge (us-east-1)
         │                   │
         │                   │ (Origin Request処理)
         │                   ▼
         └────────────▶ Frontend Lambda (Function URL)
                             │
                             │ HTTP Request
                             ▼
                      Internal ALB
                             │
                             ▼
                      Agent ECS Service
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
    DynamoDB            OpenSearch           Neptune
 (短期記憶保存)      (類似会話検索)      (エンティティ検索)
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ▼
                      Amazon Bedrock
                   (Claude 3 Haiku)
                             │
                             ▼
                       Response to User


┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  長期記憶生成フロー                                         │
└────────────────────────────────────────────────────────────────────────────────────────────┘

    EventBridge (5分間隔)
         │
         ▼
    ShortMemoryScan Lambda
         │
         │ (ProcessedIndex使用)
         ▼
    DynamoDB Scan
  (非アクティブセッション検出)
         │
         ▼
    SQS Queue
 (ShortMemoryQueue)
         │
         ▼
    Memory ECS Service
         │
         ├──────────────────────────────────────┐
         │                                      │
         ▼                                      ▼
    Amazon Bedrock                         OpenSearch
 (Titan Embed Text v2)                  (ベクトル保存)
   埋め込み生成                                 │
         │                                      │
         │                                      ▼
         │                                 Neptune
         │                            (エンティティ関係保存)
         │                                      │
         └──────────────────────────────────────┘
```
</details>


私が手動で作ったアーキテクチャは下記です。データフローが正確に描写出来ていることがわかります。  

![手動で作ったアーキテクチャ図.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/75ec9f00-b0ed-4ff7-bc75-6a915d5db873.jpeg)  

## ドリフト検出機能を使い、手動変更リソースを把握する
ドリフト検出とは、CDKをデプロイした後に手動変更されたリソースの検出と変更された設定値の確認が出来る機能です。  
「AWS マネージドコンソール > CloudFormationサービス > 対象スタック > スタックアクション > ドリフト検出」からドリフト検出を実行できます。  
また、検出されたドリフトは「AWS マネージドコンソール > CloudFormationサービス > 対象スタック > 画面左ドリフト」から確認できます。  

以下のように一覧で変更されたリソースが表示されます。  
![ドリフト検出.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/07148832-0fb9-41af-934c-b3cf3554760f.jpeg)  

更に、リソースごとに設定値がどう変更されたかもわかります。  
![ドリフト変更点.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/eb962f53-b0da-44b6-a876-3da90c5001eb.jpeg)

ルートテーブルとかセキュリティグループの変更は直接アプリ影響になる上に気付けないので絶対に実施することをおすすめします。  

# AWS CDKのソースコードを紐解く
次に適用するCDKソースコードを紐解きます。  
以下は「CloudFomationスタックからAWS環境を紐解く」からCloudFrontとFrontend用のLambdaを手動削除からAIに指示したものです。  
それなりに反映されていることがわかります。  

【指示したプロンプト】  

```
/cdkはCDKプロジェクトです。                                                                                                
構成図を作成し、markdown形式で構成図.mdに出力してください。
```
  
<details><summary>【出力結果】（クリックで展開）</summary>
```
┌────────────────────────────────────────────────────────────────────────────────┐
│ AWS Cloud                                                                      │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ VPC (Private Subnets)                                                    │  │
│  │                                                                          │  │
│  │   ┌─────────────────────────────────────────────────────────────────┐    │  │
│  │   │ アプリケーション層                                               │    │  │
│  │   │                                                                 │    │  │
│  │   │   ┌─────────────────────────────────────────────────────────┐   │    │  │
│  │   │   │                    Agent ECS (Fargate)                  │   │    │  │
│  │   │   │                      FastAPI + LangGraph                │   │    │  │
│  │   │   └───────────────────────────┬─────────────────────────────┘   │    │  │
│  │   │                               │                                 │    │  │
│  │   └───────────────────────────────┼─────────────────────────────────┘    │  │
│  │                                   │                                      │  │
│  │                 ┌─────────────────┼─────────────────┐                    │  │
│  │                 │                 │                 │                    │  │
│  │                 ▼                 ▼                 ▼                    │  │
│  │   ┌─────────────────────────┐  ┌───────────────────────────────────┐    │  │
│  │   │ 短期記憶                 │  │ 長期記憶                          │    │  │
│  │   │                         │  │                                   │    │  │
│  │   │  ┌───────────────────┐  │  │  ┌─────────────┐ ┌─────────────┐  │    │  │
│  │   │  │     DynamoDB      │  │  │  │ OpenSearch  │ │   Neptune   │  │    │  │
│  │   │  │   (会話履歴)       │  │  │  │(ベクトル検索)│ │ (グラフDB)  │  │    │  │
│  │   │  └─────────┬─────────┘  │  │  └──────┬──────┘ └──────┬──────┘  │    │  │
│  │   │            │            │  │         │               │         │    │  │
│  │   │            ▼            │  │         │               │         │    │  │
│  │   │  ┌───────────────────┐  │  │         │    ┌──────────┴───────┐ │    │  │
│  │   │  │ Short Memory Scan │  │  │         │    │                  │ │    │  │
│  │   │  │     (Lambda)      │  │  │         └───▶│   Memory ECS     │◀┘    │  │
│  │   │  └─────────┬─────────┘  │  │              │   (バッチ処理)   │      │  │
│  │   │            │            │  │              └────────┬─────────┘      │  │
│  │   │            ▼            │  │                       │                │  │
│  │   │  ┌───────────────────┐  │  │  ┌─────────────────┐  │                │  │
│  │   │  │    SQS Queue      │◀─┼──┼──│   Agent ECS     │  │                │  │
│  │   │  │ (メッセージキュー) │──┼──┼─▶│  (完了通知送信)  │  │                │  │
│  │   │  └───────────────────┘  │  │  └─────────────────┘  │                │  │
│  │   │            │            │  │                       │                │  │
│  │   │            └────────────┼──┼───────────────────────┘                │  │
│  │   │                         │  │                                        │  │
│  │   │                         │  │  ┌─────────────────┐                   │  │
│  │   │                         │  │  │    Bastion      │ (管理用)          │  │
│  │   │                         │  │  └─────────────────┘                   │  │
│  │   └─────────────────────────┘  └────────────────────────────────────────┘  │
│  │                                                                          │  │
│  │   ┌──────────────────────────────────────────────────────────────────┐   │  │
│  │   │ VPC Endpoints                                                    │   │  │
│  │   │   • Bedrock    • DynamoDB    • SQS    • ECR    • CloudWatch Logs │   │  │
│  │   └──────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                         │                                      │
│                                         ▼                                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ AWS マネージドサービス                                                    │  │
│  │   ┌─────────────────────────────────────────────────────────────────┐    │  │
│  │   │              Amazon Bedrock (LLM / Embedding)                   │    │  │
│  │   └─────────────────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```
</details>

# CDK DIFFをして最終チェックをする
`cdk diff`コマンドはデプロイしようとしているCDKと、既にデプロイされているCloudFormationスタックを比較して、差分を確認するコマンドです。  
ただし、ドリフトは加味されないため注意が必要です。  

`cdk diff`コマンドを実行することで、スタック毎に下記のような情報が出力されます。  
今までCloudFormationスタックから得た情報、CDKから得た情報、更にドリフト検出の結果を加味してどのような変更が適用されるか最終チェックします。  

- セキュリティ項目の変更
  - IAMポリシー
  - ECRポリシー
  - セキュリティグループ
- リソース変更
  - 追加
  - 削除
  - 変更

なお、実際の出力例は以下になります。  
<details><summary>出力例（クリックで展開）</summary>

```
Stack EdgeFunction
IAM Statement Changes
┌───┬───────────────────────────────────────────────────────────────────┬────────┬─────────────────────────┬───────────────────────────────────────────────────────────────────┬───────────┐
│   │ Resource                                                          │ Effect │ Action                  │ Principal                                                         │ Condition │
├───┼───────────────────────────────────────────────────────────────────┼────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────┼───────────┤
│ - │ ${Custom::CrossRegionExportWriterCustomResourceProvider/Role.Arn} │ Allow  │ sts:AssumeRole          │ Service:lambda.amazonaws.com                                      │           │
├───┼───────────────────────────────────────────────────────────────────┼────────┼─────────────────────────┼───────────────────────────────────────────────────────────────────┼───────────┤
│ - │ arn:aws:ssm:ap-northeast-1:123456789012:parameter/cdk/exports/*   │ Allow  │ ssm:DeleteParameters    │ AWS:${Custom::CrossRegionExportWriterCustomResourceProvider/Role} │           │
│   │                                                                   │        │ ssm:GetParameters       │                                                                   │           │
│   │                                                                   │        │ ssm:ListTagsForResource │                                                                   │           │
│   │                                                                   │        │ ssm:PutParameter        │                                                                   │           │
└───┴───────────────────────────────────────────────────────────────────┴────────┴─────────────────────────┴───────────────────────────────────────────────────────────────────┴───────────┘
IAM Policy Changes
┌───┬───────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────────┐
│   │ Resource                                                      │ Managed Policy ARN                                                                           │
├───┼───────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ - │ ${Custom::CrossRegionExportWriterCustomResourceProvider/Role} │ {"Fn::Sub":"arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"} │
└───┴───────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────────┘
(NOTE: There may be security-related changes not in this list. See https://github.com/aws/aws-cdk/issues/1299)

Resources
[-] Custom::CrossRegionExportWriter ExportsWriterapnortheast12334E1B8/Resource ExportsWriterapnortheast12334E1B81D43DF3F destroy
[-] AWS::IAM::Role Custom::CrossRegionExportWriterCustomResourceProvider/Role CustomCrossRegionExportWriterCustomResourceProviderRoleC951B1E1 destroy
[-] AWS::Lambda::Function Custom::CrossRegionExportWriterCustomResourceProvider/Handler CustomCrossRegionExportWriterCustomResourceProviderHandlerD8786E8A destroy


start: Building Main Template
success: Built Main Template
start: Publishing Main Template (123456789012-ap-northeast-1-b9399d58)
success: Published Main Template (123456789012-ap-northeast-1-b9399d58)
Hold on while we create a read-only change set to get a diff with accurate replacement information (use --no-change-set to use a less accurate but faster template-only diff)

Could not create a change set, will base the diff on template differences (run again with -v to see the reason)

Stack Main
IAM Statement Changes
┌───┬───────────────────────────────────────────────────────────────────────────┬────────┬────────────────────────────┬───────────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│   │ Resource                                                                  │ Effect │ Action                     │ Principal                                                         │ Condition                                                                                                        │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ - │ ${Custom::CrossRegionExportReaderCustomResourceProvider/Role.Arn}         │ Allow  │ sts:AssumeRole             │ Service:lambda.amazonaws.com                                      │                                                                                                                  │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ - │ ${FrontendLambda/Frontendfunction/FunctionUrl.FunctionArn}                │ Allow  │ lambda:InvokeFunctionUrl   │ Service:cloudfront.amazonaws.com                                  │ "ArnLike": {                                                                                                     │
│   │                                                                           │        │                            │                                                                   │   "AWS:SourceArn": "arn:${AWS::Partition}:cloudfront::${AWS::AccountId}:distribution/${CloudFront/Distribution}" │
│   │                                                                           │        │                            │                                                                   │ }                                                                                                                │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ - │ ${FrontendLambda/Frontendfunction/ServiceRole.Arn}                        │ Allow  │ sts:AssumeRole             │ Service:lambda.amazonaws.com                                      │                                                                                                                  │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ - │ arn:aws:ssm:ap-northeast-1:123456789012:parameter/cdk/exports/Main/*      │ Allow  │ ssm:AddTagsToResource      │ AWS:${Custom::CrossRegionExportReaderCustomResourceProvider/Role} │                                                                                                                  │
│   │                                                                           │        │ ssm:GetParameters          │                                                                   │                                                                                                                  │
│   │                                                                           │        │ ssm:RemoveTagsFromResource │                                                                   │                                                                                                                  │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${FailTest/Handler.Arn}                                                   │ Allow  │ lambda:GetFunction         │ AWS:${FailTest/Provider/framework-onEvent/ServiceRole}            │                                                                                                                  │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${FailTest/Handler.Arn}                                                   │ Allow  │ lambda:InvokeFunction      │ AWS:${FailTest/Provider/framework-onEvent/ServiceRole}            │                                                                                                                  │
│   │ ${FailTest/Handler.Arn}:*                                                 │        │                            │                                                                   │                                                                                                                  │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${FailTest/Handler/ServiceRole.Arn}                                       │ Allow  │ sts:AssumeRole             │ Service:lambda.amazonaws.com                                      │                                                                                                                  │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${FailTest/Provider/framework-onEvent/ServiceRole.Arn}                    │ Allow  │ sts:AssumeRole             │ Service:lambda.amazonaws.com                                      │                                                                                                                  │
├───┼───────────────────────────────────────────────────────────────────────────┼────────┼────────────────────────────┼───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ arn:aws:ssm:ap-northeast-1:123456789012:parameter/test/should-fail-delete │ Allow  │ ssm:GetParameter           │ AWS:${FailTest/Handler/ServiceRole}                               │                                                                                                                  │
└───┴───────────────────────────────────────────────────────────────────────────┴────────┴────────────────────────────┴───────────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
IAM Policy Changes
┌───┬───────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────────┐
│   │ Resource                                                      │ Managed Policy ARN                                                                           │
├───┼───────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ - │ ${Custom::CrossRegionExportReaderCustomResourceProvider/Role} │ {"Fn::Sub":"arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"} │
├───┼───────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ - │ ${FrontendLambda/Frontendfunction/ServiceRole}                │ arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole               │
│ - │ ${FrontendLambda/Frontendfunction/ServiceRole}                │ arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole           │
│ - │ ${FrontendLambda/Frontendfunction/ServiceRole}                │ arn:${AWS::Partition}:iam::aws:policy/AmazonBedrockFullAccess                                │
│ - │ ${FrontendLambda/Frontendfunction/ServiceRole}                │ arn:${AWS::Partition}:iam::aws:policy/BedrockAgentCoreFullAccess                             │
│ - │ ${FrontendLambda/Frontendfunction/ServiceRole}                │ arn:${AWS::Partition}:iam::aws:policy/AWSLambdaInvocation-DynamoDB                           │
├───┼───────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${FailTest/Handler/ServiceRole}                               │ arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole               │
├───┼───────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┤
│ + │ ${FailTest/Provider/framework-onEvent/ServiceRole}            │ arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole               │
└───┴───────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────────┘
Security Group Changes
┌───┬────────────────────────────────────────────┬─────┬────────────┬─────────────────────────────┐
│   │ Group                                      │ Dir │ Protocol   │ Peer                        │
├───┼────────────────────────────────────────────┼─────┼────────────┼─────────────────────────────┤
│ - │ ${FrontendLambda/FrontendLambdaSG.GroupId} │ Out │ Everything │ ${VPC/DefaultVpc.CidrBlock} │
└───┴────────────────────────────────────────────┴─────┴────────────┴─────────────────────────────┘
(NOTE: There may be security-related changes not in this list. See https://github.com/aws/aws-cdk/issues/1299)

Resources
[-] AWS::EC2::SecurityGroup FrontendLambda/FrontendLambdaSG FrontendLambdaFrontendLambdaSGD5A4EA3B destroy
[-] AWS::IAM::Role FrontendLambda/Frontendfunction/ServiceRole FrontendLambdaFrontendfunctionServiceRole25A3B473 destroy
[-] AWS::Lambda::Function FrontendLambda/Frontendfunction FrontendLambdaFrontendfunction7037D040 destroy
[-] AWS::Logs::LogGroup FrontendLambda/Frontendfunction/LogGroup FrontendLambdaFrontendfunctionLogGroup1F8E44BD orphan
[-] AWS::Lambda::Url FrontendLambda/Frontendfunction/FunctionUrl FrontendLambdaFrontendfunctionFunctionUrl71054C68 destroy
[-] AWS::CloudFront::Function CloudFront/Function CloudFrontFunction6A525AF6 destroy
[-] AWS::CloudFront::OriginRequestPolicy CloudFront/ForwardOriginalHost CloudFrontForwardOriginalHost8AB3A5F3 destroy
[-] AWS::CloudFront::OriginAccessControl CloudFront/Distribution/Origin1/FunctionUrlOriginAccessControl CloudFrontDistributionOrigin1FunctionUrlOriginAccessControl99EC7C7E destroy
[-] AWS::Lambda::Permission CloudFront/Distribution/Origin1/InvokeFromApiForMainCloudFrontDistributionOrigin1EAB280BC CloudFrontDistributionOrigin1InvokeFromApiForMainCloudFrontDistributionOrigin1EAB280BC2425DE28 destroy
[-] AWS::CloudFront::Distribution CloudFront/Distribution CloudFrontDistributionEAB06B35 destroy
[-] Custom::CrossRegionExportReader ExportsReader/Resource ExportsReader8B249524 destroy
[-] AWS::IAM::Role Custom::CrossRegionExportReaderCustomResourceProvider/Role CustomCrossRegionExportReaderCustomResourceProviderRole10531BBD destroy
[-] AWS::Lambda::Function Custom::CrossRegionExportReaderCustomResourceProvider/Handler CustomCrossRegionExportReaderCustomResourceProviderHandler46647B68 destroy
[+] AWS::SSM::Parameter FailTest/ShouldFailDeleteParam FailTestShouldFailDeleteParamB78ED8B1
[+] AWS::IAM::Role FailTest/Handler/ServiceRole FailTestHandlerServiceRoleF976D07F
[+] AWS::IAM::Policy FailTest/Handler/ServiceRole/DefaultPolicy FailTestHandlerServiceRoleDefaultPolicy6FED9499
[+] AWS::Lambda::Function FailTest/Handler FailTestHandlerEB2D5BE4
[+] AWS::Logs::LogGroup FailTest/Handler/LogGroup FailTestHandlerLogGroupC82329DB
[+] AWS::IAM::Role FailTest/Provider/framework-onEvent/ServiceRole FailTestProviderframeworkonEventServiceRole74031A02
[+] AWS::IAM::Policy FailTest/Provider/framework-onEvent/ServiceRole/DefaultPolicy FailTestProviderframeworkonEventServiceRoleDefaultPolicy81242EFF
[+] AWS::Lambda::Function FailTest/Provider/framework-onEvent FailTestProviderframeworkonEvent085EE688
[+] AWS::Logs::LogGroup FailTest/Provider/framework-onEvent/LogGroup FailTestProviderframeworkonEventLogGroup70A00805
[+] AWS::CloudFormation::CustomResource FailTest/ResourceV2 FailTestResourceV2704AB875
[~] AWS::Lambda::Function ShortMemoryScan/ShortMemoryScanFunction ShortMemoryScanShortMemoryScanFunction29B169E6
 ├─ [~] Code
 │   └─ [~] .S3Key:
 │       ├─ [-] 60976e2b399247c24af427ed4e4caf46f653bd8eed1ea49385cb533c165170e6.zip
 │       └─ [+] 4f709b02131c14e2231c681a771b1db8d3092a79308e5779092280ffcad5203b.zip
 └─ [~] Metadata
     └─ [~] .aws:asset:path:
         ├─ [-] asset.60976e2b399247c24af427ed4e4caf46f653bd8eed1ea49385cb533c165170e6
         └─ [+] asset.4f709b02131c14e2231c681a771b1db8d3092a79308e5779092280ffcad5203b
```
</details>

# 最後に
ここまでやったら後は覚悟を決めて`cdk deploy`をしましょう。  
私はいつもリアルサマーウォーズです。  