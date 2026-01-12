---
title: AWS CDKを書き始める前に決めておきたいこと
tags:
  - AWS
  - IaC
  - CDK
  - AWSCDK
private: false
updated_at: '2026-01-12T10:36:57+09:00'
id: d5fd96603b7586b6ec08
organization_url_name: ap-com
slide: false
ignorePublish: false
---
こんにちくわ。  
転職してからずっとAWS CDKを書いている[@sori883](https://x.com/sori883)です。  

# はじめに
近年では、IaCの用いたインフラ構築が当たり前になっている来ていますよね。  
しかし、私が知る限りではCDKに明るい意思決定者がいない、レビューするベテラン層もCDKに詳しくなかったりプログラムを書いたことがないということが多々ありました。  
これによって、「実は実装難易度が高かった」「再デプロイ時、リソース再作成になってしまった」といった困難が作業者へ直接降りかかるなんでことにもなりかねません。  

今リアルタイムで困難を背負っているので、これからAWS CDKを始めるプロジェクトの困難を少しでも取り除ければいいなと思って共有します。  

# テンプレートを作成して土台を標準化する
AWS CDKは様々な属性の方が作成に当たるため、一番最初にテンプレート的な土台を定め、コーディング規則や設定ファイルの作成方法を標準化しましょう。  
標準化することで、メンバー間の記述が統一されレビューや引き継ぎ負荷が軽減されます。  

以下のリンクに私が普段検証に使っているAWS CDKテンプレートを記載します。  
どのプロジェクトにも使えそうな機能を簡単に盛り込んでいます。  

- ESLint
  - 命名規則や型定義を統一
- Prettier
  - フォーマット（インデント、インポート順）の統一
- 設定パラメータの定義

https://github.com/sori883/aws-cdk-template  

# 必要な環境をきめておく
開発環境、ステージング環境、本番環境などの複数環境デプロイするプロジェクトがほとんどです。  
CDKを記述する前に必要な環境とどのようにデプロイするのか前もって決めておきましょう。  

- 各環境のデプロイ先
  - 同じアカウント
    - リソース名が重複しないように作る
    - リソース数のクオートを考慮して作る
  - 異なるアカウント
    - 環境間のリソース受け渡しを明確にして作る
    - 環境間の依存関係を明確にして作る


## 環境ごとにパラメータを分けておく
性能パラメータといった環境ごとに異なるパラメーターは設定で分けたい！となるので、あらかじめ分けて設定しておくと後から楽です。  
例えば以下のようにパラメーターファイルを定義します。  

```ts
// parameter.ts

export const parameter = (envName: EnvNameType) => ({
  prefix: envName,
  // 環境差分パラメータを展開する
  diffEnv: envDiffParameter(envName),
});

// 環境差分パラメータを以下のように定義しておく
const envDiffParameter = (envName: EnvNameType) => {
  const params = {
    prd: {
      ec2: {
        instanceType: ec2.InstanceType.of(
          ec2.InstanceClass.T3,
          ec2.InstanceSize.MICRO
        ),
      },
    },
    stg: {
      ec2: {
        instanceType: ec2.InstanceType.of(
          ec2.InstanceClass.T3,
          ec2.InstanceSize.MICRO
        ),
      },
    },
    dev: {
      ec2: {
        instanceType: ec2.InstanceType.of(
          ec2.InstanceClass.T3,
          ec2.InstanceSize.MICRO
        ),
      },
    },
  };
  return params[envName];
};
```

パラメーターファイルはCDKデプロイコマンドの`context`で分岐出来るようにして、`bin/aws-cdk-template.ts`から、各スタックに注入していきます。  

```bash
cdk deploy --context env=dev or stg or prd
```

以下は実装例です。  

```ts
// パラメーターファイルを読み込む
import { parameter as p } from '../parameter';

// CDK実行時のコンテキストを取得する
// 例: cdk deploy --context env=devであるならばdev
const env = validateEnvName(app.node.tryGetContext('env'));
// パラメーターを取得する
const parameter = p(env);

new InfraStack(app, 'Infra', {
  stackName: `${parameter.prefix}-infra`,
  env: { account: parameter.dotEnv.ACCOUNT_ID, region: parameter.region },
  parameter, // スタックに渡す
});


// スタックのPropsにパラメーターを追加
interface StackProps extends cdk.StackProps {
  readonly parameter: ParameterType;
}

// スタック
export class InfraStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: StackProps) {
    super(scope, id, props);
    // パラメータ取得する
    const { parameter } = props;

    // EC2を作成
    new ec2.Instance(this, 'Instance', {
          vpc,
          vpcSubnets: { subnets: [subnet] },
          instanceType: parameter.instanceType, // インスタンスタイプにパラメータを使用する
          machineImage,
          securityGroup,
          keyPair,
          role,
        });
  }
}
```

## デプロイする担当者を決めておく
デプロイする人によって情報の取り扱いが変わります。開発者または運用者なのかを決めておきましょう。  
また、ビジネス制約からデプロイではなく納品し、お客様が既存環境にデプロイする場合もあるかと思います。  
要求を事前に確認しておきましょう。  

- 機密情報の取り扱い
  - SecretsManagerから参照する
- 既存リソースの取り扱い
  - SSMパラメーターストアから参照する


# 適切にスタックを分割する
スタック分割していますか？、まとめていますか？  
人によって意見が分かれるので、一概には言えませんがプロジェクトで結論を出しておきましょう。  

## スタックは原則分割しない
まず、スタックは原則分割せず構築することがベストです。  
これはスタックを分割することで、ほぼ確実にスタック間に依存関係が生じるためCDK更新時に問題が生じるためです。  

> デプロイ要件に応じて、アプリケーションのStageを複数のStackに分割する
一般的には、できるだけ多くのリソースを同じ Stack に入れておくのが最も簡単です。あらかじめ分離したいと分かっている場合を除いて、同じStackに入れます。

[https://aws.amazon.com/jp/blogs/news/best-practices-for-developing-cloud-applications-with-aws-cdk/](https://aws.amazon.com/jp/blogs/news/best-practices-for-developing-cloud-applications-with-aws-cdk/)  

以下の図では、親子の依存関係および循環依存が発生してしまっている図です。  
- Appスタック→Networkスタック
  - EC2がVPCやSubnetを参照しているため、親子の依存関係が発生
  - Appスタックを削除しないとNetworkスタックを更新出来ない状況が発生
- Appスタック←→Dataスタック
  - セキュリティグループの循環依存が発生
    - EC2のセキュリティグループがRDSのセキュリティグループを参照
    - RDSのセキュリティグループがEC2のセキュリティグループを参照
  -  デプロイ自体出来ない可能性が発生する


![AWS_CDK_スタック依存.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/1c032632-dd0f-476b-93a1-6aa8e31dc316.jpeg)  

運用フェーズ以降、気軽にスタックを操作　出来ない場合は致命的になる場合があります。  
分割する必要性が思いつかないスタックはなるべく1つにまとめましょう。  

## スタックを分割する状況
依存関係のデメリットがあっても以下の場合はスタック分けを考えます。  

- マルチリージョン、マルチアカウントを使用する場合
  - マルチリージョンや、マルチアカウントをサポートしていないため
- デプロイ時にCDK以外が介入する場合
  - ECRへ個別にイメージをデプロイする必要がある
  - S3へ個別に設定ファイルを設置する必要がある

ただ、分割する場合は依存関係を考慮します。    
例え下記はライフサイクルを意識して依存関係を整理したパターンです。  

- Networkスタックは不変である可能性が高いのでベースにする
  - Appスタック、DataスタックはNetworkスタックだけに依存させる
- Appスタック、Dataスタック間の依存を取り除く
  - SSMパラメータストア
  - SecretsManager

![AWS_CDK_スタック分割.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/806f19e6-051c-449a-af7c-bab5c6fcf9dc.jpeg)  

# リソース名は原則設定しない
リソース名は設定しないことがベターです。  
もし命名規則を作るので、リソース名を設定してくださいと言われた場合は強い心で断りましょう。  

>自動で生成されるリソース名を使用し、物理的な名前を使用しない

[https://aws.amazon.com/jp/blogs/news/best-practices-for-developing-cloud-applications-with-aws-cdk/](https://aws.amazon.com/jp/blogs/news/best-practices-for-developing-cloud-applications-with-aws-cdk/)  

## どうしてもリソース名を設定する場合
それでもリソース名を設定する動機は、命名規則が与えられ覆せない場合かと思います。  
もしリソース名を命名規則に適用する可能性がある場合は下記に注意しましょう。  

***命名規則にリソースIDを使用しない***  
命名規則にリソースIDを含めると、循環依存が発生します。  
- リソースID発行のためにリソース構築しないといけない←→リソースを構築するために命名規則が必要

具体例を出すと、APIGatewayの実行ロググループ名はデフォルトでリソースIDの含まれたリソースが生成されますが、アクセスロググループ名もそれに合わせて命名したり...。  

***早い段階で命名規則を確定させる***  
後から命名規則を適用するとリソース削除が作り直しになり、意図しない影響に繋がる場合があります。    
1回目のデプロイが完了する前には命名規則を確定させておきましょう。  

- 一部リソース（RDSインスタンス..）は再作成が必要
- AWS CDKの論理IDが代わり別リソースとしてデプロイされる

補足として、AWS CDKはリソース名ではなく、論理ID（以下の`MyFunction`部分）で各リソースを一意に判断しています。  
以下はLambda関数をロググループを指定せずに作成した場合のリソース名と論理IDです。  
- Lambda関数名
  - リソース名：my-custom-function-name(自動生成) 
  - CDK論理ID：MyFunction
- ロググループ名:
  - リソース名：/aws/lambda/my-custom-function-name(自動生成)
  - CDK論理ID：自動生成

```ts
// Lambda作成
new lambda.Function(this, 'MyFunction', {
      runtime: lambda.Runtime.NODEJS_20_X,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda'),
      timeout: cdk.Duration.seconds(30),
      memorySize: 256,
    });
```

以下はLambda関数とロググループを別々に作成した場合のリソース名と論理IDです。  
Lambda関数作成時に自動で作られたロググループとは別の論理IDになるため、エラーの原因になったり、管理外のロググループが生まれる原因になります。  

- Lambda関数名
  - リソース名：my-custom-function-name(自動生成) 
  - CDK論理ID：MyFunction
- ロググループ名:
  - リソース名：hogehoge-log-gloup
  - CDK論理ID：MyLambdaLogGroup

```ts
// ロググループ作成
const logGroup = new logs.LogGroup(this, 'MyLambdaLogGroup', {
  logGroupName: 'hogehoge-log-gloup',
});

// Lambda作成
const fn = new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
  logGroup, // ロググループを設定
});
```

# 終わりに
引き継ぎしたAWS CDKを機能追加しながら地道にリファクタリングしているのですが、現在進行形で格闘中です。  
色々決まりきっているので今更直せなない部分があったり本記事のような部分で悩んだり、改めてAWS CDKに向けの設計って重要かもと思いました。  
とはいいつつ、AWS CDKのベストプラクティスがプロジェクトのベストプラクティスではないので、周りの関係者各位とコミュニケーションを取って出来る限り良いものを作っていきましょう。  

# 参考
[https://aws.amazon.com/jp/blogs/news/best-practices-for-developing-cloud-applications-with-aws-cdk/](https://aws.amazon.com/jp/blogs/news/best-practices-for-developing-cloud-applications-with-aws-cdk/)
[https://tmokmss.hatenablog.com/entry/20221121/1669032738](https://tmokmss.hatenablog.com/entry/20221121/1669032738) 
[https://engineering.mobalab.net/2024/10/22/my-best-practices-for-aws-cdk](https://engineering.mobalab.net/2024/10/22/my-best-practices-for-aws-cdk)  
[https://speakerdeck.com/gotok365/aws-cdk-reusability](https://speakerdeck.com/gotok365/aws-cdk-reusability)  
[https://zenn.dev/yamaren/articles/87c7d2f0e817d9](https://zenn.dev/yamaren/articles/87c7d2f0e817d9)  
