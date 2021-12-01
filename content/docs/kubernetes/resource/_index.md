---
weight: 2
title: "リソースの詳細"
---
## 本セクションで学習すること
- Deployment、Replicaset、Pod の概要
- Pod のスケールアウト

## Deployment について
前の手順でDeployment を作成しましたが、Deployment は追加のリソースを作成します。調べてみましょう。
```shell
kubectl get all -o name -l app=blog
```
all はリソース名ではなく、Kubernetesのコアのリソースタイプを表示するエイリアスです。

出力は以下のようになるはずです。
```shell
pod/blog-6b8999855c-6jjhj
pod/blog-6b8999855c-zk8pg
deployment.apps/blog
replicaset.apps/blog-6b8999855c
```
Deployment の作成によって、結果的にReplicaset とPod のリソースが追加で作成されています。

これは、Deploymentが、Replicaset を作成するためのテンプレートとして機能するためです。また、Replicaset は、Pod を作成するためのテンプレートとして機能します。Pod は、アプリケーションのインスタンスを表すものです。このケースでは、レプリカの数が2に設定されているため、2つのPod が存在します。

{{< figure src="k_deployment.png" width="75%">}}

Replicaset のリソース定義を表示するには、次のように実行します。
```shell
kubectl get replicaset -l app=blog -o yaml
```

    spec:
    replicas: 2
    selector:
        matchLabels:
        app: blog
        pod-template-hash: "896156207"
    template:
        metadata:
        creationTimestamp: null
        labels:
            app: blog
            pod-template-hash: "896156207"
        spec:
        containers:
        - env:
            - name: BLOG_SITE_NAME
            value: EduK8S Blog
            image: quay.io/eduk8s-labs/app-k8s-fundamentals-frontend:latest
            imagePullPolicy: Always
            name: blog
            ports:
            - containerPort: 8080
            protocol: TCP
            resources: {}
            terminationMessagePath: /dev/termination-log
            terminationMessagePolicy: File
        dnsPolicy: ClusterFirst
        restartPolicy: Always
        schedulerName: default-scheduler
        securityContext: {}
        terminationGracePeriodSeconds: 30


すなわち、Replicase のマニフェストは、Deployment のSpec で提供されたもので構成されています。

Pod のリソース定義を表示するには以下を実行します。
```shell
kubectl get pod -l app=blog -o yaml
```

ここでは、Replicaset のspec.template のフィールドがPod のリソース定義の作成に使われています。

Replicaset とPod の両方で、多くのフィールドが追加されていることがわかります。これは、元のDeployment では指定されていなかった値のデフォルトが記入されているためです。また、リソース定義には、リソースのステータスを追跡するためのフィールドも含まれています。

継承されたデフォルトの場合、リソースの種類によって異なりますが、グローバル構成やネームスペース構成から一部が継承されている場合もあります。例えば、CPUとメモリのリソース制限は、作業しているネームスペースで設定された制限を継承しています。

いずれにしても、これらのデフォルト値が正しくないことが判明し、それらを変更する必要がある場合、または追加の設定を追加する必要がある場合は、その変更をDeployment で行う必要があります。Replicaset やPod を自分で直接編集してはいけません。代わりにDeployment を更新してください。Replicaset とPod のインスタンスは、それに対応して更新されます。

{{<hint info>}}
Kubernetes の世界ではマニフェストがすべてです。例えばReplicaset のマニフェストを直接編集し、それを管理するDeployment との整合性にズレが生じた場合、整合性を取るようDeployment のマニフェストに従うように再構成されます。
{{</hint>}}

アプリケーションを削除する際には、先ほどフロントエンドのWebアプリケーションで行ったように、Replicaset もPod も明示的に削除する必要はありません。これは、Pod は作成元のReplicaset が所有するものとしてマークアップされ、Replicaset はDeployment が所有するものとしてマークされるためです。Deoloyment を削除すると、Replicaset と、そこから作成されたPod は自動的に削除されます。

{{<hint info>}}
長々と書いていますが、要するにKubernetes でアプリケーションをデプロイしたい場合は基本的にDeployment リソースを使う、とだけ覚えれば十分です。
{{</hint>}}

## スケールアウト
Pod をスケールアウトしてみましょう。先ほど説明した通り、Pod やReplicaset を編集するのではなく、Deployment のマニフェスト中のspec.replicas の値を変更する必要があります。今回は、レプリカの数を3 に増やすために、Deployment のマニフェストを更新し、再びApply してみましょう。

```shell
sed -i -e "s/replicas: 2/replicas: 3/" ~/frontend/deployment.yaml
kubectl apply -f ~/frontend/deployment.yaml
```

Pod の状態を確認しましょう。

```shell
kubectl get pods -l app=blog
```
コマンドをすぐに実行した場合、「Pending（保留）」または「ContainerCreating（コンテナ作成中）」状態のPod が表示されることがあります。これは新しいPod が起動しているところです。Running 状態のPodが3つ表示されるまで、kubectl get podsを実行し続けます。

このように、非常に簡単にPod をスケールアウトさせることができました。仮想マシンの場合、通常このように即座にスケールアウトすることはできません。コンテナを利用する1 つのメリットとして、このように拡張性に非常に優れることが挙げられます。

## Pod とコンテナ
Pod がアプリケーションのインスタンスを表すことはすでに述べたとおりです。

より正確に言うと、Pod は、実行中のコンテナのグループを表す抽象的なもので、Pod はコンテナの管理およびスケーリングをするための単なるグルーピングにすぎません。ほとんどの場合、Pod は1つのコンテナのみで構成されます。1つのPod で複数のコンテナを実行するユースケースもありますが、ここでは1 Pod = 1 コンテナと覚えてしまっても差し支えありません。

今回のハンズオンで使用するサンプルアプリケーションを見ると、2つの別々のDeployment があります。1つはフロントエンドのWeb サーバー用、もう1つはデータベース用です。

```shell
kubectl get deployment
```

    NAME      READY   UP-TO-DATE   AVAILABLE   AGE
    blog      2/2     2            2           64s
    blog-db   1/1     1            1           34s

このようにDeployment を分けることで、フロントエンドのWebアプリケーションをデータベースとは別に複数のインスタンスにスケールアウトすることができます。両方のコンポーネントが1 つのDeployment の一部として作成され、Pod の別々のコンテナで実行されていた場合、アプリケーションを複数のインスタンスにスケールアウトすることはできません。これは、データベースはデータと紐づくため（＝ステートフル）、データの整合性などを気にする必要があり、Web サーバーのようなステートレスアプリケーションのようにレプリカ数を簡単に増やすことはできないからです。

### その他のリソース
本ハンズオンでは紹介できませんが、Kubernetes には他にも沢山のリソースがあります。また、自分でリソースを作ることもできます。詳細はKubernetes のドキュメントを参照してください。

参考 : [https://kubernetes.io/docs/concepts/](https://kubernetes.io/docs/concepts/)

## 演習
1. 以下を参考に、nginx のバージョン1.14.2 のコンテナイメージを使ったDeployment を作成してください(作成中エラーが出ますが想定通りです)。その後、作成したDeployment を削除をしてください。
   https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/
2. Deployment で管理されるPod (Deployment > Replicaset > Pod) を手動で削除した場合、何が起こると考えられますか？
 
{{<details "解答" >}}
1. ドキュメント中のマニフェストをそのまま使いましょう。vim 等で作成してもよいですが、URL を直接引数に入れることができます。
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/deployment.yaml
```
デプロイされたDeployment、Replicaset、Pod を見てみましょう。
```shell
kubectl get deployment,replicaset,pod
```
エラーが出てコンテナの作成に失敗していますが、無視し、確認したら削除をします。Pod やReplicaset を個別に削除する必要はありません。
```shell
kubectl delete -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/application/deployment.yaml
```

2. Deployment のマニフェストの中のReplicaset のspec でPod のレプリカ数を定義しているため、Replicaset のマニフェストと整合性を取るよう、Pod が再作成されます。確かめてみましょう。

まずは今稼働しているPod を表示してみましょう。
```shell
kubectl get pods -l app=blog 
```

    NAME                    READY   STATUS    RESTARTS   AGE
    blog-65c8484545-7c8qk   1/1     Running   0          20m
    blog-65c8484545-qbvvp   1/1     Running   0          20m

次に、Pod を手動で削除します。Pod 名は環境によって異なるため、先ほど表示されたもののどちらか一方を使用してください。
```shell
kubectl delete pods blog-65c8484545-7c8qk 
```
すると、以下のように表示されるはずです。

    pod "blog-65c8484545-7c8qk" deleted

しかしながら、再度稼働中のPod を表示してみると、Pod の数に変化がありません。

    NAME                    READY   STATUS    RESTARTS   AGE
    blog-65c8484545-br9mf   1/1     Running   0          8s
    blog-65c8484545-qbvvp   1/1     Running   0          21m

これは、実際にはPod が削除された後、自動的にPod が再作成されているためです。その証拠に、先ほど削除したPod 名(この例ではblog-65c8484545-7c8qk) は存在せず、作成されたばかりの（この例ではAGE=8s）、別名のPod が表示されています。

{{</details>}}


