---
title: 30分でAmazon Bedrock AgentCore RuntimeにAIエージェントをデプロイする
tags:
  - AWS
  - 初心者
  - UV
  - bedrock
  - AgentCore
private: false
updated_at: '2025-10-24T16:07:47+09:00'
id: 6396332cbff4e302ca13
organization_url_name: ap-com
slide: false
ignorePublish: false
---
こんにちは〜。  
[@sori883](https://x.com/sori883)です！  

Amazon Bedrock AgentCoreのお試しということで、30分でRuntimeにAIエージェントをデプロイしてみようと思います。  
めっちゃめちゃ簡単にデプロイできるので、AgentCore触ったことない方の参考になれば幸いです。  


| ソフトウェア | バージョン |
|---------|-----------|
| macOS | 15.6.1 |
| Python | 3.13.7 |
| uv | 0.9.5 |
| bedrock-agentcore | 1.0.3 |
| bedrock-agentcore-starter-toolkit | 0.1.26 |
| strands-agents | 1.13.0 |

成果物はこちらを参照ください。

https://github.com/sori883/30min-agentcore


# はじめに
AgentCore Runtimeへのデプロイはどんなイメージをお持ちでしょうか？  

私はAWS公式のチュートリアルに掲載されていた画像を見て、「Docker Image作って、ECRにPushしてやっとデプロイ...めんどくさそう...」と思っていました。  
![AWS公式のBedrock AgentCoreデプロイのイメージ](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/e1ca649a-8108-42cd-b922-c546d30cfbe2.jpeg)

実際には、[bedrock-agentcore-starter-toolkit](https://github.com/aws/bedrock-agentcore-starter-toolkit)を使うことで、DockerやECRを時前構築することなくコマンド1発でデプロイできます！  


# Python、uvのインストール
まず、`Python`や`uv`を導入します。  

Macであれば、Homebrewといったパッケージ管理ツールでインストールできます。  

```
brew install python
brew install uv
```

Windowsであれば、[Python公式](https://www.python.org/downloads/windows/)からインストール出来ます。  
また、`uv`は[Installing uv](https://docs.astral.sh/uv/getting-started/installation/#__tabbed_1_2)を参照ください。  

`uv`の使い方はこちらが参考になります。  

https://qiita.com/futakuchi0117/items/9ec8bd84797fed180647

# uvのプロジェクトのセットアップ
インストールが完了したら、`uv`プロジェクトを作成します。  

```
uv init test-app
cd test-app
```

以下のフォルダ、ファイルが作成されていたら成功です。  
```
.git/
.gitignore
.python-version
main.py
pyproject.toml
README.md
```

## 動作確認
uvプロジェクトのセットアップで作成された`main.py`を実行し、動作確認を行います。  

```
uv run main.py
```

`Hello from test-app!`と表示されたら動作確認完了です。  

# デプロイするAIエージェントを作成
本題の前にデプロイするAIエージェントを作成していきます。  
AWS公式のクイックスタートに沿ってやっていきます！  

https://aws.github.io/bedrock-agentcore-starter-toolkit/user-guide/runtime/quickstart.html

## パッケージインストール
まずは、以下のコマンドを実行し、必要なパッケージをインストールします。  
```
uv add bedrock-agentcore-starter-toolkit bedrock-agentcore strands-agents
```

インストールするパッケージの役割は、以下のとおりです。  

- bedrock-agentcore-starter-toolkit
  - AgentCoreへ簡単にAIエージェントをデプロイするパッケージ
- bedrock-agentcore
  -  AIエージェントをAgentCoreに対応させるパッケージ
- strands-agents
  - AWS製のAIエージェント構築フレームワーク


## AIエージェント実装
`main.py`を開き、以下のとおり修正します。  

```py
from bedrock_agentcore import BedrockAgentCoreApp
from strands import Agent

app = BedrockAgentCoreApp()

# AIエージェントの作成
agent = Agent(
    ## システムプロンプト設定
    system_prompt="日本語で回答してください。"
)

@app.entrypoint
def invoke(payload):
    # ユーザープロンプトの取得
    user_message = payload.get("prompt", "こんにちは。元気ですか？")
    # ユーザプロンプトをもとに回答を生成して返却
    result = agent(user_message)
    return {"result": result.message}

if __name__ == "__main__":
    app.run()
```

## ローカルで動作確認
以下のコマンドでエージェントを起動します。  
```
uv run main.py
```

次に**別のターミナル**で、以下のとおりPOSTリクエストをしてみます。  
```
curl -X POST http://localhost:8080/invocations \
  -H "Content-Type: application/json" \
  -d '{"prompt": "これは動作確認です。元気ですか？"}'
```

以下のように、生成AIからの回答があれば動作確認OKです！  
```
{"result": {"role": "assistant", "content": [{"text": "こんにちは！元気です、ありがとうございます。\n\n動作確認ということですが、私は正常に動作しており、日本語での会話も問題なくできます。何かお手伝いできることがあれば、お気軽にお声かけください。\n\nあなたはいかがですか？何かご質問やお話ししたいことがあれば、どうぞお聞かせください。"}]}}%
```

※エラー（`{"error":"Unable to locate credentials"}%`）が表示される場合は、末尾の付録：トラブルシューティングを参照ください。  

#  AgentCore Runtimeへデプロイ
さて、AIエージェントの動作確認も終わったらいよいよデプロイしていきたいと思います。  

## デプロイ設定
まずは、デプロイ先のアカウントや、Runtime名を決めるために、デプロイ設定をします。  

```
uv run agentcore configure -e main.py -r ap-northeast-1
```

色々質問されますが、全て`Enter`で大丈夫です。  
参考として、以下に実行結果を記載しておきます。クリックで適宜展開してください。  

<details><summary>実行結果（クリックで展開）</summary>

```
agentcore configure -e main.py -r ap-northeast-1
Configuring Bedrock AgentCore...
✓ Using file: main.py

🏷️  Inferred agent name: main
Press Enter to use this name, or type a different one (alphanumeric without '-')
Agent name [main]:
✓ Using agent name: main

🔍 Detected dependency file: pyproject.toml
Press Enter to use this file, or type a different path (use Tab for autocomplete):
Path or Press Enter to use detected dependency file: pyproject.toml
✓ Using requirements file: pyproject.toml

🔐 Execution Role
Press Enter to auto-create execution role, or provide execution role ARN/name to use existing
Execution role ARN/name (or press Enter to auto-create):
✓ Will auto-create execution role

🏗️  ECR Repository
Press Enter to auto-create ECR repository, or provide ECR Repository URI to use existing
ECR Repository URI (or press Enter to auto-create):
✓ Will auto-create ECR repository

🔐 Authorization Configuration
By default, Bedrock AgentCore uses IAM authorization.
Configure OAuth authorizer instead? (yes/no) [no]:
✓ Using default IAM authorization

🔒 Request Header Allowlist
Configure which request headers are allowed to pass through to your agent.
Common headers: Authorization, X-Amzn-Bedrock-AgentCore-Runtime-Custom-*
Configure request header allowlist? (yes/no) [no]:
✓ Using default request header configuration
Configuring BedrockAgentCore agent: main




💡 No container engine found (Docker/Finch/Podman not installed)
✓ Default deployment uses CodeBuild (no container engine needed), For local builds, install Docker, Finch, or Podman

Memory Configuration
Tip: Use --disable-memory flag to skip memory entirely

No region configured yet, proceeding with new memory creation
✓ Short-term memory will be enabled (default)
  • Stores conversations within sessions
  • Provides immediate context recall

Optional: Long-term memory
  • Extracts user preferences across sessions
  • Remembers facts and patterns
  • Creates session summaries
  • Note: Takes 120-180 seconds to process

Enable long-term memory? (yes/no) [no]:
✓ Using short-term memory only
Will create new memory with mode: STM_ONLY
Memory configuration: Short-term memory only
Generated .dockerignore
Generated Dockerfile: .bedrock_agentcore/main/Dockerfile
Setting 'main' as default agent
╭───────────────────────────────────────────────── Configuration Success ─────────────────────────────────────────────────╮
│ Agent Details                                                                                                           │
│ Agent Name: main                                                                                                        │
│ Runtime: None                                                                                                           │
│ Region: ap-northeast-1                                                                                                  │
│ Account: 012345678912                                                                                                   │
│                                                                                                                         │
│ Configuration                                                                                                           │
│ Execution Role: Auto-create                                                                                             │
│ ECR Repository: Auto-create                                                                                             │
│ Authorization: IAM (default)                                                                                            │
│                                                                                                                         │
│                                                                                                                         │
│ Memory: Short-term memory (30-day retention)                                                                            │
│                                                                                                                         │
│ 📄 Config saved to: /Users/test-app/.bedrock_agentcore.yaml                                                             │
│                                                                                                                         │
│ Next Steps:                                                                                                             │
│    agentcore launch                                                                                                     │
╰─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

```

</details>

以上でデプロイ設定は完了です。  
また、実行することで以下のファイルやフォルダが生成されます。  
```
.bedrock_agentcore/
.bedrock_agentcore.yaml
.dockerignore
```

`.bedrock_agentcore.yaml`には`AWS アカウントID`や`ローカルの絶対パス`が記載されているので取扱い注意です。  

## デプロイ
デプロイコマンドを実行して完了です！  

```
uv run agentcore launch
```

デプロイ完了すると、作成されたリソース一覧が表示されます。  
```
Deployment completed successfully - Agent: arn:aws:bedrock-agentcore:ap-northeast-1:012345678912:runtime/main-XXXXXXXXX
╭────────────────────────────────────────────────── Deployment Success ───────────────────────────────────────────────────╮
│ Agent Details:                                                                                                          │
│ Agent Name: main                                                                                                        │
│ Agent ARN: arn:aws:bedrock-agentcore:ap-northeast-1:012345678912:runtime/main-XXXXXXXXX                                 │
│ ECR URI: 012345678912.dkr.ecr.ap-northeast-1.amazonaws.com/bedrock-agentcore-main:latest                                │
│ CodeBuild ID: bedrock-agentcore-main-builder:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx                                   │
│                                                                                                                         │
│ 🚀 ARM64 container deployed to Bedrock AgentCore                                                                        │
│                                                                                                                         │
│ Next Steps:                                                                                                             │
│    agentcore status                                                                                                     │
│    agentcore invoke '{"prompt": "Hello"}'                                                                               │
│                                                                                                                         │
│ 📋 CloudWatch Logs:                                                                                                     │
│    /aws/bedrock-agentcore/runtimes/main-XXXXXXXXX-DEFAULT --log-stream-name-prefix "2025/10/24/[runtime-logs]"          │
│    /aws/bedrock-agentcore/runtimes/main-XXXXXXXXX-DEFAULT --log-stream-names "otel-rt-logs"                             │
│                                                                                                                         │
│ 🔍 GenAI Observability Dashboard:                                                                                       │
│    https://console.aws.amazon.com/cloudwatch/home?region=ap-northeast-1#gen-ai-observability/agent-core                 │
│                                                                                                                         │
│ ⏱️  Note: Observability data may take up to 10 minutes to appear after first launch                                     │
│                                                                                                                         │
│ 💡 Tail logs with:                                                                                                      │
│    aws logs tail /aws/bedrock-agentcore/runtimes/main-XXXXXXXXX-DEFAULT --log-stream-name-prefix                        │
│ "2025/10/24/[runtime-logs]" --follow                                                                                    │
│    aws logs tail /aws/bedrock-agentcore/runtimes/main-XXXXXXXXX-DEFAULT --log-stream-name-prefix                        │
│ "2025/10/24/[runtime-logs]" --since 1h                                                                                  │
╰─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```


# デプロイしたAIエージェントを使ってみる
AWSマネージドコンソールに「エージェントサンドボックス」という機能があるので、それを使ってみます。  

**Amazon Bedrock AgentCore > エージェントサンドボックス**を開きます。  

以下の通り入力し、「実行」ボタンを押下します。  
```
{"prompt": "はじめまして。これは動作確認です。"}
```

出力に生成AIからの回答が返ってきているかと思います！  

![AgentCoreデプロイ確認.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/5ac0d0bf-0034-4112-a762-1651445c050e.jpeg)


# お掃除
最後に、デプロイしたリソースを削除します。  
```
uv run agentcore destroy
```

こんな感じの表示が出れば削除完了です！  
```
╭─────────────────────────────────────────── Resources Successfully Destroyed ────────────────────────────────────────────╮
│   ✓ AgentCore agent: arn:aws:bedrock-agentcore:ap-northeast-1:012345678912:runtime/main-0yfUQM2oSs                      │
│   ✓ ECR images: 1 images from bedrock-agentcore-main                                                                    │
│   ✓ CodeBuild project: bedrock-agentcore-main-builder                                                                   │
│   ✓ Memory: main_mem-NbQxFkG8MJ                                                                                         │
│   ✓ Deleted CodeBuild IAM role: AmazonBedrockAgentCoreSDKCodeBuild-ap-northeast-1-XXXXXXXXX                             │
│   ✓ IAM execution role: AmazonBedrockAgentCoreSDKRuntime-ap-northeast-1-XXXXXXXXX                                       │
│   ✓ Agent configuration: main                                                                                           │
│   ✓ Configuration file (no agents remaining)                                                                            │
╰─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

# 付録
## トラブルシューティング
`uv run main.py`などのコマンドが実行できず、以下のメッセージが表示されている場合のトラシューです。  
```
{"error":"Unable to locate credentials"}%   
```

これはAWSクレデンシャルが設定されていないため、発生しています。  
以下のとおりAWSクレデンシャルすることで解消できます。  

```
# AWSクレデンシャルを設定する
aws configure

# プロファイルがdefault以外の場合は以下を実行
export AWS_DEFAULT_PROFILE={プロファイル名}
```

https://dev.classmethod.jp/articles/aws-cli_initial_setting/


## フロントエンドから使ってみる
詳細は解説しませんが、Next.jsからAgentCoreのRuntimeを実行し、ストリーミング表示するサイトを構築しました。  
ソースコードは以下のGitHubを参照ください。  

https://github.com/sori883/study-agentcore/tree/01.deploy-agent-to-runtime

構成図はこんな感じです。  
![構成図.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/315113/1b06672c-0299-4b94-b6e0-0141a5b9432c.jpeg)

# 終わりに
使っていてCDKやCloudFormationじゃないんだ...と思ってました。  
なんとなくAgentCoreは独自の生態系を築いて行きそう。  

あと、uvの勉強もせねば...  
uv venv要らないって初めて知った。  

# 参考
- [amazon-bedrock-agentcore-samples](https://github.com/awslabs/amazon-bedrock-agentcore-samples/blob/main/01-tutorials/01-AgentCore-runtime/README.md)
- [QuickStart: Your First Agent in 5 Minutes!](https://aws.github.io/bedrock-agentcore-starter-toolkit/user-guide/runtime/quickstart.html)
