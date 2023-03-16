---
title: "Self-hosted Agentを使ってAzure PipelineからプライベートEC2経由でECRにイメージを登録する"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Merkdown"]
published: true
---
※Zenn初投稿になります

# はじめに
Azure Pipelineを利用してコードのビルド・デプロイを行う場合、1つ以上のエージェントが必要になります。

[MSドキュメント](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser)にはエージェントが以下の3種類あると記載されています。

- Microsoft によってホストされるエージェント
- セルフホステッド エージェント(以下SHAgent)
- Azure Virtual Machine Scale Set エージェント(セルフホステッド エージェントベース)

それらのうち、SHAgentを利用するとAWSのプライベートサブネット上にあるEC2をAzure Pipelineのビルド・デプロイの実行先として登録することができます。

今回は、Azure PipelineからAWSのEC2経由で制限されたECRリポジトリに対して、コンテナイメージをPushする構成を検証してみました。

<br>

# 参考リンク
今回検証する上では以下の記事がかなり参考になりました。

* [DevOps の Self-hosted エージェントを構築して使ってみよう！](https://jpdscore.github.io/blog/azuredevops/try-self-hosted-agent/)
* [Azure DevOps Pipelines — Build and Push a Docker image to AWS ECR](https://cj-hewett.medium.com/azure-devops-pipelines-build-and-push-a-docker-image-to-aws-ecr-bc0d35f8f126)

<br>

# 前提条件
以下の作業は事前に完了している前提とします。

- PAT作成済み(Agent登録時に利用)
- Azure DevOps Organization作成
- Azure DevOps プロジェクト作成
- Azure Reposリポジトリ作成
- Azure Pipeline用コード作成([GitHub](https://github.com/handy-dd18/AzurePipeline-SHAgent-EC2-ECR))
- EC2(Amazon Linux 2)・NATGW構築
- EC2へのgit/dockerのインストール
- ECRリポジトリ作成
- ECRポリシー設定

<br>

# 構成図
[MSドキュメント](https://learn.microsoft.com/ja-jp/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=browser#communication-with-tfs)にも記載がある通り、Azure PipelineとSHAgent間の通信はSHAgentからポーリング接続を行うようになっているため、EC2にはAzure PipelineへのOutbound通信が許可されている必要があります。
そのため、以下のようにEC2の前段にNATGatewayを配置し、インターネットに出れる構成にします。
![](/images/azure-pipeline-shagent-ec2-ecr/azure-pipeline-ec2-ecr.png)

<br>

# 実施内容
## 概要
作業内容としては以下の流れで実施します。

1. エージェントプール作成
2. SHAgentインストールコマンド確認
3. EC2ログイン、SHAgentインストール・実行
4. エージェント登録確認
5. パイプライン作成
6. パイプライン実行
7. イメージPush確認(ECRリポジトリ)

<br>

## エージェントプール作成
まずはAzure DevOps Organizationsのコンソールにログインして、左下の[Organization settings]から設定画面に遷移します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_1.png)

<br>

左側メニューのPipelinesに[Agent pools]項目があるので、そこからエージェントプール一覧を表示し、右上の[Add pool]ボタンから新しいエージェントプールの作成を行います。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_2.png)

<br>

エージェントプール作成画面では、プールタイプを[Self-hosted]、プール名にわかりやすい名前を入力して、作成ボタンを押下すると、新しいエージェントプールが作られます。
permissinosはpipelinesへの権限のみチェックを付けておきます。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_3.png)

<br>

作成したエージェントプールをクリックしてAgentsタブを押下すると、プールに登録されているAgentが確認できます。
※事前検証で設定済みだったので、画面は登録後のものです
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_4.png)

<br>

## SHAgentインストールコマンド確認
作成したエージェントプール画面の右上の[New agent]ボタンを押下します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_5.png)

<br>

agentをインストールするコマンドが表示されるため、インストール先のOS等によって適切なものを確認します。
今回はAmazon Linuxにインストールするので、Linuxのコマンドを確認してメモしておきます。
agentファイルだけダウンロードリンクになっているので、コマンド実行前にEC2側でダウンロードが必要になります。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_6.png)

<br>

実際に実行したコマンドは以下になります。
※config.shだけ対話入力が必要なので注意が必要です。

```bash
$ whoami; date
$ cd ~
$ wget https://vstsagentpackage.azureedge.net/agent/2.217.2/vsts-agent-linux-x64-2.217.2.tar.gz -P ~/Downloads/
$ ls -l ~/Downloads/
$ mkdir myagent && cd myagent
$ tar zxvf ~/Downloads/vsts-agent-linux-x64-2.217.2.tar.gz
$ ./config.sh
$ ./run.sh
```

<br>

## EC2ログイン、SHAgentインストール・実行
Session Manager経由でAWSマネジメントコンソールからEC2にログインして、agentインストールコマンドを順番に実行していきます。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_7.png)

<br>

config.shを実行すると対話形式で7回入力する必要があります。
入力内容は以下になります。

- 1. 入力 (Y/N) Team Explorer Everywhereの使用許諾契約を今すぐ受け入れるか？　→　Y
- 2. サーバーURLを入力　→　https://dev.azure.com/xxxxxxxx
- 3. 認証方法を入力　→　PAT
- 4. PAT情報を入力　→　xxxxxxxxxxxxxxxxxxxxxxxxxxxx　※後述
- 5. 追加するエージェントプールを入力　→　[先ほど作成したエージェントプール名]
- 6. 追加するエージェント名を入力　→　linux-agent
- 7. 作業ディレクトリを入力(デフォルトは_work)　→　_work ※未入力でEnterでも可

![](/images/azure-pipeline-shagent-ec2-ecr/20230316_8.png)

<br>

config.shの設定が完了したら、run.shを実行します。
run.shが実行されるとAzure DevOpsとの通信が確立されて、Azure側のエージェントプールに先ほど登録したエージェント名が表示されます。
※最初のほうにも記載しましたが、SHAgentはポーリング接続でJobの実行を確認するため、常時起動させる場合はバックグラウンドで動くように設定しておく必要があります。

![](/images/azure-pipeline-shagent-ec2-ecr/20230316_10.png)

<br>

### ※PATについて
PATとはPersonal Access Tokenの略で、エージェント登録コマンドを実行しているユーザーが、Azure DevOpsのユーザーであることを保証するためのトークンになります。
PATの作成方法は他の方が記事にされていると思うので、そちらを確認して作成してください。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_9.png)

<br>

## エージェント登録確認
EC2側でリッスン状態になった後、Azure DevOpsコンソールからエージェントプールを見ると、先ほど登録した内容でエージェントが登録されたことが確認できます。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_11.png)

<br>

## パイプライン作成
エージェントの登録が確認出来たら次はAzure Pipelineから、以下の流れでパイプラインの作成を行っていきます。
※途中、既にパイプライン作成済みのAzure Reposのリポジトリを選択している関係で最後のyamlファイル名に[-1]がついてますが、リポジトリに同名ファイルがなければ[-1]無しの[azure-pipelines.yml]というファイル名になると思います。

<br>

Azure Pipelineの画面から[New pipeline]ボタンを押下します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_12.png)

<br>

ソースコードが格納されているGitリポジトリを指定します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_13.png)

<br>

対象のリポジトリ名を選択します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_14.png)

<br>

pipelineでビルドデプロイする環境を選択します。
今回はDockerfileを使用してイメージをECRにPushするので、一番上のDockerを選択します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_15.png)

<br>

Dockerfileはソースコードに含まれているファイルを指定します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_16.png)

<br>

パイプラインの実行はまだしないので、Saveボタンを押下します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_17.png)

<br>

パイプラインを作成するとリポジトリにazure-pipeline.ymlファイルが追加されることになるので、コミット先のブランチを指定して保存します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_18.png)

<br>

パイプライン保存後、自動作成されるazure-piplnies.ymlを以下のように修正します。
※[こちら](https://github.com/handy-dd18/AzurePipeline-SHAgent-EC2-ECR)にソースコード置いてます

```
# Starter pipeline
# Start with a minimal pipeline that you can customize to build and deploy your code.
# Add steps that build, run tests, deploy, and more:
# https://aka.ms/yaml

trigger:
  branches:
    include:
      - main

stages:
- template: templates/build.yml
```

<br>

### ※実行する前に：パイプライン環境変数設定
本記事では、ビルド・デプロイ時に指定するAWSリージョン名とECRリポジトリ名をパイプラインの環境変数に設定するように構成しているので、実行前にコンソールから登録しておきます。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_21.png)
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_22.png)


<br>

### ※実行する前に：ECRにPushするIAMユーザー情報登録
Azire PipelineからECRリポジトリにPushする場合、Push操作を行うIAMユーザーの情報を事前に登録することが可能です。
Azure DevOpsコンソールの[Project Settings]からサービス接続の項目を選択し、[New service connection]ボタンから新しく接続情報を追加します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_23.png)
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_24.png)


<br>

IAMユーザーのアクセスキーとシークレットアクセスキーの入力は必須ですが、オプションとしてAssume Roleにも対応しているようでした。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_25.png)


<br>

サービス名まで入力できたら[Save]ボタンを押下して、サービスを作成します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_26.png)

<br>

templates/build.ymlファイル内で以下のように指定して、登録したIAMユーザーの認証情報をJobパラメータとして渡すことが可能です。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_32.png)

<br>

## パイプライン実行
Azure DevOpsのコンソールから作成したパイプラインを開いて、[Run pipeline]ボタンを押下してパイプラインを実行します。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_19.png)
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_20.png)

<br>

しばらく待っているとJobが正常終了することが確認できました。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_28.png)

<br>

SHAgent側でもJobの実行と結果が出力されていました。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_29.png)

<br>

## イメージPush確認(ECRリポジトリ)
ECRリポジトリを見ると、イメージが登録されていることが確認できました。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_30.png)

<br>

ECRリポジトリは特定のIPかVPCEndpointID以外からのアクセスを拒否するように設定済みのため、SHAgentをインストールしたプライベートサブネット上のEC2からデプロイされたとみて間違いなさそうです。
![](/images/azure-pipeline-shagent-ec2-ecr/20230316_31.png)

<br>

# おわりに
前にGitLab Runnerを構築してECR/ECSにビルド&デプロイする検証([URL](https://qiita.com/handy-dd18/items/f095407d0b49b826d007))をしていたので、Azure Pipelineのyamlファイルのイメージはすんなり理解できました。

普段はAWSメインでAzureを使うことはないのですが、たまたま機会があったので自分の備忘として書いてみました。
Azure側の理解ができていないところもあるので、もし説明や設定内容が間違っていたらご指摘いただけると助かります。

この記事がどなたかの参考になれば幸いです。