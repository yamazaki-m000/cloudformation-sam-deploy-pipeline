# README

##メモ

Datadogトークンを環境変数でCodebuildに入れ込むための変更を加えたブランチ
現状Datadog用lambdaの1リポジトリだけ

sh make_pipeline.sh <環境名> <Githubトークン> <Datadogトークン>

updateするときはmake_pipeline.shの中のcreateをupdateへ全置換すればよい

## 概要

SAMをデプロイする為のパイプラインを作成するCloudFormationです｡

- common.yml
    - パイプライン共通のリソースを作成
    - システムで1度だけ実施、CMにて実施済
- pipeline.yml
    - SAMアップロード、およびデプロイのパイプラインを作成
    - 1つのアプリケーションに対して2回実施
    - CFn実行時にはアプリケーション名とブランチ名（master(開発用ブランチ)、production(検証/本番用ブランチ)）を指定
    

の2種類のテンプレートファイルを使用します｡

## 前提

- AWS CLIが実行できる環境であること
- CLIを実行するIAMユーザーにAdmin権限が付与されていること
- `pipeline.yml`実行時には事前にアプリケーションソース用のリポジトリが作成されていること

## common.yml

### 作成対象リソース

| 論理 ID                         | タイプ              |
| ------------------------------- | ------------------- |
| ArtifactStore                   | AWS::S3::Bucket     |
| CodeBuildServiceRole            | AWS::IAM::Role      |
| CodeBuildServiceRolePolicies    | AWS::IAM::Policy    |
| CodePipelineServiceRole         | AWS::IAM::Role      |
| CodePipelineServiceRolePolicies | AWS::IAM::Policy    |
| LogBucket                       | AWS::S3::Bucket     |
| LogGroup                        | AWS::Logs::LogGroup |

### 実行パラメータ

なし


### 出力

| キー                       | エクスポート名                                |
| -------------------------- | --------------------------------------------- |
| ArtifactStoreName          | sam-deploy-artifactstore-name            |
| CodeBuildServiceRoleName   | sam-deploy-codebuild-service-role-name   |
| CodePipelineServiceRoleArn | sam-deploy-codepipeline-service-role-arn |
| LogBucketName              | sam-deploy-logbucket-name                |
| LogGroupName               | sam-deploy-loggroup-name                 |

### CLI

Create Stack

```bash
STACK_NAME=sam-deploy-pipeline
STAGE=[環境名] // dev,stg,prd

aws cloudformation create-stack --stack-name ${STACK_NAME} \
    --template-body=file://common.yml \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters ParameterKey=Env,ParameterValue=${STAGE}

aws cloudformation wait stack-create-complete --stack-name ${STACK_NAME}
```

Update Stack

```bash
STACK_NAME=sam-deploy-pipeline
STAGE=[環境名] // dev,stg,prd

aws cloudformation update-stack --stack-name ${STACK_NAME} \
    --template-body=file://common.yml \
    --capabilities CAPABILITY_NAMED_IAM \
    --parameters ParameterKey=Env,ParameterValue=${STAGE}

aws cloudformation wait stack-update-complete --stack-name ${STACK_NAME}
```

Delete Stack

```bash
STACK_NAME=sam-deploy-pipeline
aws cloudformation delete-stack --stack-name ${STACK_NAME}
aws cloudformation wait stack-delete-complete --stack-name ${STACK_NAME}
```

## pipeline.yml

### 作成対象リソース

| 論理 ID             | タイプ                      |
| ------------------- | --------------------------- |
| PipelineWebhook     | AWS::CodePipeline::Webhook  |
| CodePipeline        | AWS::CodePipeline::Pipeline |
| CodeBuildProject    | AWS::CodeBuild::Project     |
| GitHubWebhookSecret | AWS::SecretsManager::Secret |

### 実行パラメータ

| パラメータ       | デフォルト値（設定値）                  |備考|
| ---------------- | ---------------------------- |--|
| CommonStackName  | sam-deploy-pipeline　　　|common.yml実行時のStackNameを指定|
| ApplicationName  | ${ApplicationName}-${Branch} |アプリケーション名とブランチ名を結合した値を指定|
| GitHubOwner      | PIONEER-ISPC	              ||
| GitHubRepository | ${ApplicationName}           |アプリケーション名を指定|
| GitHubToken      | ${GitHubToken}               |Githubのtoken https://ispc.backlog.com/view/PIONEER_EKS-59|
| Branch           | master                       |トリガーとなるブランチ、開発用、検証/本番用で当該ブランチを分ける|
| ComputeType      | BUILD_GENERAL1_SMALL         |CodeBuildで利用するDockerのサイズ|
| Image            | aws/codebuild/standard:2.0 |CodeBuildで利用するDockerImage|


### 出力

なし

### CLI

Create Stack

```bash
APPLICATION_NAME={{YOUR_APPLICATION_NAME}}
STACK_NAME=sam-deploy-pipeline-${APPLICATION_NAME}
COMMON_STACK_NAME=sam-deploy-pipeline
GITHUB_OWNER={{YOUR_GITHUB_OWNER}}
GITHUB_TOKEN={{YOUR_GITHUB_TOKEN}}
GITHUB_REPOSITORY={{YOUR_GITHUB_REPOSITORY}}
GITHUB_BRANCH={{YOUR_GITHUB_BRANCH}}
aws cloudformation create-stack --stack-name ${STACK_NAME} \
    --template-body=file://pipeline.yml \
    --parameters \
        ParameterKey=CommonStackName,ParameterValue=${COMMON_STACK_NAME} \
        ParameterKey=ApplicationName,ParameterValue=${APPLICATION_NAME} \
        ParameterKey=GitHubOwner,ParameterValue=${GITHUB_OWNER} \
        ParameterKey=GitHubToken,ParameterValue=${GITHUB_TOKEN} \
        ParameterKey=GitHubRepository,ParameterValue=${GITHUB_REPOSITORY} \
        ParameterKey=Branch,ParameterValue=${GITHUB_BRANCH}
aws cloudformation wait stack-create-complete --stack-name ${STACK_NAME}
```

Update Stack

```bash
APPLICATION_NAME={{YOUR_APPLICATION_NAME}}
STACK_NAME=sam-deploy-pipeline-${APPLICATION_NAME}
COMMON_STACK_NAME=sam-deploy-pipeline
GITHUB_OWNER={{YOUR_GITHUB_OWNER}}
GITHUB_TOKEN={{YOUR_GITHUB_TOKEN}}
GITHUB_REPOSITORY={{YOUR_GITHUB_REPOSITORY}}
GITHUB_BRANCH={{YOUR_GITHUB_BRANCH}}

aws cloudformation update-stack --stack-name ${STACK_NAME} \
    --template-body=file://pipeline.yml \
    --parameters \
        ParameterKey=CommonStackName,ParameterValue=${COMMON_STACK_NAME} \
        ParameterKey=ApplicationName,ParameterValue=${APPLICATION_NAME} \
        ParameterKey=GitHubOwner,ParameterValue=${GITHUB_OWNER} \
        ParameterKey=GitHubToken,ParameterValue=${GITHUB_TOKEN} \
        ParameterKey=GitHubRepository,ParameterValue=${GITHUB_REPOSITORY} \
        ParameterKey=Branch,ParameterValue=${GITHUB_BRANCH}

aws cloudformation wait stack-update-complete --stack-name ${STACK_NAME}
```

Delete Stack

```bash
APPLICATION_NAME={{YOUR_APPLICATION_NAME}}
STACK_NAME=sam-deploy-pipeline-${APPLICATION_NAME}

aws cloudformation delete-stack --stack-name ${STACK_NAME}

aws cloudformation wait stack-delete-complete --stack-name ${STACK_NAME}
```
