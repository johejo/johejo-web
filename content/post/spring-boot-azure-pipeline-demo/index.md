---
title: "Spring Boot Azure Pipelines Demo"
date: 2019-01-08T16:28:08+09:00
draft: true
---

## Spring BootのサンプルアプリをAzure PipelinesでビルドしてAzure Container RegistryにPushする

Azure PipelinesとはAzure DevOpsの機能（コンポーネント？）の1つでいわゆるCI/CDツールです。

https://azure.microsoft.com/ja-jp/services/devops/pipelines/

Azureとの連携がとても簡単でした。

軽く試してみたのでその記録を残しておきたいと思います。


### 準備

#### Spring Bootのプロジェクトの用意

Spring Initializrを使います。

https://start.spring.io/

今回はGradle Projectを選択し、DependenciesにはWebを指定しました。

Generate Projectをクリックして、ダウンロードしたzipファイルを適当なところに展開します。

Initial Commitを行ったらそのままリモートリポジトリにpushします。

今回はリモートリポジトリのホスティング先にはGitHubを使いました。

#### Azure Container Registryの準備

Azureのドキュメントがとてもわかりやすいのでここでは割愛します。

https://docs.microsoft.com/ja-jp/azure/container-registry/container-registry-get-started-azure-cli

#### Azure DevOpsの準備

GitHubのMarketplaceでGitHubアカウントと連携させます。なお連携にはMicrosoftアカウントが必要になります。

https://github.com/marketplace/azure-pipelines

途中でリポジトリのアクセス権限を聞かれたら先ほどのdemoアプリのリポジトリを選択します。

Azure DevOpsのトップページでCreate projectをクリックしてプロジェクトを作ります。

#### Service connectionの設定

Azure DevOpsのプロジェクトをAzure上のリソースと連携させるためにService connectionを作成します。

![](image0.png)

今回はAzure Resource Managerを選択して先ほどの作成したAzure Container Registryが入っているリソースグループを指定します。

#### Azure Pipelinesの準備

プロジェクトができたらPipelinesのところでNewをクリックしてパイプラインを作ります。ここでは既存のパイプラインのインポートもできるみたいです。

New Pipelinesではソースリポジトリを選んで進んでいくとChoose a templeteでGradleが出てくるので選びます。

するとエディターが開くのでazure-pipelines.ymlをここで編集できます。

```
trigger:
  - master

variables:
  Azure.ServiceConnectionId: 'FOO' # ここで作成したService ConnectionのConnection nameを指定
  ACR.Name: 'BAR' # ACRの名前

  ACR.RepositoryName: '$(ACR.Name)'
  ACR.ImageName: '$(ACR.Name):master'
  ACR.FullName: '$(ACR.Name).azurecr.io'

jobs:

  - job: BuildImage
    displayName: Build

    pool:
      vmImage: 'Ubuntu-16.04'

    steps:
      - task: Gradle@2
        displayName: 'Build module'
        inputs:
          workingDirectory: ''
          gradleWrapperFile: 'gradlew'
          gradleOptions: '-Xmx3072m'
          javaHomeOption: 'JDKVersion'
          jdkVersionOption: '1.11'
          jdkArchitectureOption: 'x64'
          publishJUnitResults: false
          testResultsFiles: '**/TEST-*.xml'
          tasks: 'build'

      - task: Docker@1
        displayName: 'Build an image'
        inputs:
          azureSubscriptionEndpoint: '$(Azure.ServiceConnectionId)'
          azureContainerRegistry: '$(ACR.FullName)'
          imageName: '$(ACR.ImageName)'
          command: build

      - task: Docker@1
        displayName: 'Push an image'
        inputs:
          azureSubscriptionEndpoint: '$(Azure.ServiceConnectionId)'
          azureContainerRegistry: '$(ACR.FullName)'
          imageName: '$(ACR.ImageName)'
          command: push
```
