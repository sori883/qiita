---
title: AWS Lambda durable functionsでCloudWatch Logsのログをエクスポートしてみる
tags:
  - AWS
  - lambda
  - DurableFunctions
  - CDK
private: false
updated_at: '2026-02-28T23:14:17+09:00'
id: f30867e121c02e578b56
organization_url_name: ap-com
slide: false
ignorePublish: false
---

こんにちは。  
AWS CDKに囚われている[@sori883](https://x.com/sori883)です。

最近Lambda durable functionsを使ってCloudWatch Logsのログエクスポートを実装したので、その備忘録を残します。

# AWS Lambda durable functionsの概要

durable functionsは通常のLambdaにチェックポイント、リプレイを追加し、
通常のLambda関数単体では難しかった待機、中断、再開をネイティブサポートしてくれる機能です。

- `@durable_execution`を設定することで、durable実行が出来る
- `context.step`、`@durable_step`を設定することでstepが定義できる
  - step完了時は自動でチェックポイントが保存される
- `context.wait`で処理を待機出来る
  - wait完了後は自動でチェックポイントが保存される

```py
from aws_durable_execution_sdk_python import durable_execution, durable_step, DurableContext
from aws_durable_execution_sdk_python.config import Duration

@durable_step
def step_1(step_context):
    return "Step 1 done"

@durable_step
def step_2(step_context):
    return "Step 2 done"

@durable_execution
def lambda_handler(event: dict, context: DurableContext):
    result1 = context.step(step_1(), name="step-1") # step1
    context.wait(Duration.from_seconds(10)) # 10秒待機
    result2 = context.step(step_2(), name="step-2") # step2
    return {"step1": result1, "step2": result2}
```

今までは15分以上実行するためにStep Functionsと組み合わせる一工夫が必要でしたが、durable functionsを使えばLambda関数単体で実装できるようになります。  

# durable functionsでログエクスポートする

今までLambda関数とStep Functionsで実装していましたが、durable functionsで実装してみます。

## レポジトリ

durable functions、S3、EventBridgeをCDKで実装したレポジトリです。  
cdk deployだけで使えるはず。  

https://github.com/sori883/aws-log-export

## durable functionsの実装

durable functionsで実装するログエクスポートフローは下記です。

1. エクスポート対象のロググループを取得
2. CreateExportTaskを実行しログをS3にエクスポート
3. CreateExportTaskのステータスを確認しながら完了するまで待機

これをそのままdurable functionsのstepとして定義します。

### Step1. エクスポート対象のロググループを取得

特定のタグが付与されたロググループを取得するStepを定義します。

```py
@durable_step
def get_target_log_groups(step_context):
    # 特定タグが付与されたロググループを取得する
    log_groups = []
    paginator = tagging_client.get_paginator("get_resources")

    for page in paginator.paginate(
        TagFilters=[{"Key": TARGET_TAG_KEY, "Values": [TARGET_TAG_VALUE]}],
        ResourceTypeFilters=["logs:log-group"],
    ):
        for resource in page.get("ResourceTagMappingList", []):
            arn = resource["ResourceARN"]
            log_group_name = arn.split(":log-group:")[-1]
            if log_group_name.endswith(":*"):
                log_group_name = log_group_name[:-2]
            log_groups.append(log_group_name)

    return log_groups
```

### Step2. CreateExportTaskを実行しログをS3にエクスポート

次にCreateExportTaskを実行しロググループをS3niエクスポートするStepを定義します。  

```py
@durable_step
def create_export_task(step_context, log_group_name, from_time, to_time, date_str):
    # CreateExportTaskを呼び出してログをエクスポート
    prefix = f"logs/{log_group_name.replace('/', '-').lstrip('-')}/{date_str}"
    response = logs_client.create_export_task(
        logGroupName=log_group_name,
        fromTime=from_time,
        to=to_time,
        destination=EXPORT_BUCKET,
        destinationPrefix=prefix,
    )
    return response["taskId"]
```


### Step3. CreateExportTaskの完了ステータスを確認

最後にCreateExportTaskのステータスを確認するStepを定義します。  

```py
@durable_step
def check_export_status(step_context, task_id):
    # エクスポートタスクのステータスを確認する
    response = logs_client.describe_export_tasks(taskId=task_id)
    tasks = response.get("exportTasks", [])

    if not tasks:
        raise Exception(f"Export task not found: {task_id}")

    current_status = tasks[0]["status"]["code"]

    if current_status in ["FAILED", "CANCELLED"]:
        message = tasks[0]["status"].get("message", "Unknown error")
        raise Exception(f"Export task {current_status}: {message}")

    return current_status
```

### ハンドラー関数

`@durable_execution`を付与したハンドラー関数で処理フローを組んでいきます。  
- `context.step`で定義したstepを使用する
- `context.wait`で待機する

```py
@durable_execution
def handler(event: dict, context: DurableContext):
    # Step1. エクスポート対象のロググループを取得
    log_groups = context.step(get_target_log_groups(), name="get-target-log-groups")

    results = []

    for log_group_name in log_groups:
      # Step2. CreateExportTaskを実行しログをS3にエクスポート
        task_id = context.step(
            create_export_task(log_group_name, from_time, to_time, date_str),
            name=f"create-export-{log_group_name}",
        )
        
        status = "PENDING"
        # ステータスがCOMPLETEDになるまで繰り返す
        while status != "COMPLETED":
            # 120秒待機
            context.wait(Duration.from_seconds(120))
            # Step3. CreateExportTaskの完了ステータスを確認
            status = context.step(
                check_export_status(task_id),
                name=f"check-{task_id}",
            )

        results.append({
            "logGroupName": log_group_name,
            "taskId": task_id,
            "status": "COMPLETED",
        })

    return {
        "status": "COMPLETED",
        "exportedCount": len(results),
        "results": results,
    }
```

## 実行結果

ログが少なすぎて4分で終わっていますが、お仕事環境でやった時は3時間くらい動いていることを確認してます。  
また、永続オペレーションの実行ログではdurable functionsで定義した各stepがWaitを挟んで意図したフローで実行されていることが伺えます。  

![durable functionsの実行結果.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/34a33914-413a-4cb5-9c51-f3d7f8c87656.jpeg)  
