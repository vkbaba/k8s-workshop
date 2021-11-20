---
weight: 1
title: "基本操作"
---

## クラスタへのアクセス
これから行うハンズオンでは、kubectl CLIを使用してKubernetesとやり取りします。このCLI は、ワークショップ環境の「ターミナル」タブからアクセスできるインタラクティブなターミナルセッションで提供されます。ご自身のコンピュータに何かをインストールする必要はありません。ここでは、Webブラウザを使ってすべての操作を行うことになります。使用するKubernetesクラスタにはすでに接続されているので、ログインする必要はありません。

ワークショップ環境では、KubernetesクラスタをWebベースで確認することもできます。これは、ワークショップ環境の「コンソール」タブから利用できます。これは、ハンズオンで行うことの結果を視覚的に確認するために含まれていますが、ハンズオンはこれに依存しません。

ハンズオンを続ける前に、kubectlコマンドが実行され、ワークショップ環境も機能していることを確認します。これを行うには、次のように実行します。
```shell
kubectl version
```

実行すると、次のような出力が表示されます。

    Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.0", GitCommit:"70132b0f130acc0bed193d9
    ba59dd186f0e634cf", GitTreeState:"clean", BuildDate:"2019-12-07T21:20:10Z", GoVersion:"go1.13.4", Compiler:"
    gc", Platform:"linux/amd64"}
    Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.0", GitCommit:"70132b0f130acc0bed193d9
    ba59dd186f0e634cf", GitTreeState:"clean", BuildDate:"2019-12-07T21:12:17Z", GoVersion:"go1.13.4", Compiler:"
    gc", Platform:"linux/amd64"}


使用しているKubernetesのバージョンは、ここで示したバージョンと異なる場合があります。

## アプリケーションのデプロイ
さて、Kubernetesクラスタへのアクセスが正常に行われていることを確認したら、早速、ブログサイトを実装したフロントエンドのWebアプリケーションと、ブログ記事を保存するためのPostgreSQLデータベースで構成される完全なアプリケーションをデプロイしてみましょう。

これは、すでに構成ができていれば、Kubernetesに完全なアプリケーションをいかに早くデプロイできるかを示すためです。アプリケーション全体がデプロイされたら、フロントエンドのWebアプリケーションコンポーネントを削除し、段階的にデプロイし直すことで、どのように組み合わされているのか、どのようにKubernetesを使用しているのかを確認することができます。

デプロイしたいアプリケーションの最初の部分は、PostgreSQLデータベースです。これをデプロイするためのリソースファイル一式は、databaseディレクトリにあります。
```shell
ls -las database/
```
このディレクトリ内の各ファイルには、アプリケーションコンポーネントの配置を構成するための異なるリソース定義が含まれています。

各ファイルの定義を調べるよりも、まずはディレクトリ内のリソースをKubernetes に処理させてみましょう。これを行うには、次のように実行します。
```shell
kubectl apply -f database/ --dry-run
```
これで出力されるはずです。

    secret/blog-credentials created (dry run)
    service/blog-db created (dry run)
    persistentvolumeclaim/blog-database created (dry run)
    deployment.apps/blog-db created (dry run)

今回のkubectl applyコマンドは、設定ファイルやディレクトリに含まれるファイル群からリソースを作成するためのものです。ここでは--dry-runオプションを使い、クラスタ内のオブジェクトを一切作成せずに、どのようなオブジェクトを作成するかを指示しています。また、--dry-runオプションはリソースの定義を検証し、エラーがある場合は警告します。

あるコマンドが何をするのか、どんなオプションを受け付けるのかが不明な場合は、--helpオプションを付けて実行することができます。
```shell
kubectl apply --help
```

ドライランデプロイメントを行うことで、作成されるリソースを確認することができました。実際にデータベースコンポーネントをデプロイするために、今度は次のように実行します。
```shell
kubectl apply -f database/
```
ドライランの時と同様に、kubectl applyはリソースをリストアップしますが、今回は実際にリソースが作成されます。

    secret/blog-credentials created
    service/blog-db created
    persistentvolumeclaim/blog-database created
    deployment.apps/blog-db created

このリストの中で重要なリソースはdeployment です。deployment では、アプリケーションにデプロイするコンテナイメージの名前、起動するインスタンスの数、デプロイメントの管理方法などを指定します。

デプロイの進捗を監視し、完了を知るには、次のコマンドを実行します。
```shell
kubectl rollout status deployment/blog-db
```
引数には、リソースのタイプとこのインスタンスの名前を含む、リソースのフルネームを指定します。この場合、インスタンスはblog-dbという名前でした。

データベースがデプロイされたので、今度はフロントエンドのWebアプリケーションを次のように実行してデプロイします。
```shell
kubectl apply -f frontend/ 
```
出力は以下のようになるはずです。

    persistentvolumeclaim/blog-media created
    deployment.apps/blog created
    service/blog created
    ingress.extensions/blog created

以下を実行して監視し、デプロイが完了するのを待ちます。
```shell
kubectl rollout status deployment/blog
```
## デプロイしたアプリケーションへのアクセス
フロントエンドのWebアプリケーション用にingressオブジェクトが作成されていることに注目してください。このオブジェクトは、アクセス可能なURLを払い出し、Webアプリケーションへのアクセスを設定します。

この例では、WebアプリケーションにアクセスするためのURLは次のようになります。
```shell
NS=$(kubectl config view -o jsonpath='{.contexts[].context.namespace}')
http://${NS}.tdc-reg-prod-d66138e.tanzu-labs.esp.vmware.com
```
{{hint info}}
1つ目のコマンドはこの環境で払い出されるURL 名の一部を取得しています。
{{/hint}}

このリンクをクリックして、フロントエンドのWebアプリケーションにアクセスします。利用できないと表示された場合は、利用できるようになるまでページを更新してください。これは、Ingress の設定に時間がかかる場合があるためです。

まだ、ブログ記事は表示されません。データベースの設定と入力については後ほど説明します。

KubernetesクラスタのWebコンソールを使用すると、どのリソースが作成されているか、またそれらのリソース間の関係をブラウザで視覚的に表示することができますが、ほとんどの開発者はkubectlを使用してコマンドラインからKubernetesクラスタを操作します。

現在の名前空間にある、すでに作成されたすべてのデプロイメントの一覧を表示するには、次のように実行します。
```shell
kubectl get deployment
```
出力は以下のようになるはずです。

    NAME      READY   UP-TO-DATE   AVAILABLE   AGE
    blog      2/2     2            2           5m
    blog-db   1/1     1            1           5m

特定のリソースに絞り込みたい場合は、コマンドにそのリソース名を加えることができます。
```shell
kubectl get deployment/blog
```
これで、1つのリソースだけの出力が得られるはずです。もしくは、以下のように実行することもできます。
```shell
kubectl get deployment blog
```
出力の最初の部分は以下のようになるはずです。

    Name:                   blog
    Namespace:              lab-k8s-fundamentals-user1
    CreationTimestamp:      Tue, 04 Feb 2020 03:06:56 +0000
    Labels:                 app=blog
    Annotations:            deployment.kubernetes.io/revision: 1
    Selector:               app=blog
    Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
    StrategyType:           RollingUpdate
    MinReadySeconds:        0
    RollingUpdateStrategy:  25% max unavailable, 25% max surge
    Pod Template:
        ....

これは比較的読みやすい形式にしたものであり、生のリソース定義を見るには、kubectl getの-o yaml表示出力オプションを使います。
```shell
kubectl get deployment/blog -o yaml
```
また、YAMLではなくJSONで作業をしたい場合には以下を実行します。
```shell
kubectl get deployment/blog -o json
```
