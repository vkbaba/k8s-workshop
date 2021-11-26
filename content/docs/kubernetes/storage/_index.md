---
weight: 4
title: "ストレージとデータベース"
---
## 本セクションで学習すること
- PV とPVC によるデータの永続化
- データベースとの接続のための環境変数とSecret リソース
  
## ストレージの基本
繰り返しになりますが、Pod は短命です。Pod は簡単に削除されるため、データを残したい場合は何かしらの形で外部に保存する仕組みが必要です。その1 つがPersistent Volume （PV）になります。

PV はその名の通り永続ストレージであり、PV の中にデータを保存することができます。PV をPod に割り当てるにはいくつか方法がありますが、代表的なものとしてPersistent Volume Claim (PVC) を使う方法があります。PVC はPV を要求するために使われるリソースであり、そのマニフェストの中には例えば要求するストレージのサイズなどが定義されます。Pod がデータを保存するためにPV を使いたい場合は、直接PV をアタッチするのではなく、PVC をアタッチします。

{{< figure src="k_pvc.png" width="100%" >}}
{{<hint info>}}
物理ストレージをKubernetes 用に抽象化したものがPVで、それをPod にアタッチするためにPVC が使われます。
{{</hint>}}

PVC のマニフェストを見てみましょう。

```shell
cat ~/database/persistentvolumeclaim.yaml
```

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: blog-media
      labels:
        app: blog
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi

ここでは、ストレージのサイズは少なくとも1Giで、アクセスモードはReadWriteOnce である必要があります。アクセスモードとは、払い出すPV がどのようにRead/Write されうるかを示しており、以下の4 つがあります。

- ReadWriteOnce(RWO) - 単一のノードからRead/Write 可能としてボリュームをマウントします。
- ReadOnlyMany (ROX) - 複数のノードによって読み取り専用としてボリュームをマウントします。
- ReadWriteMany (RWX) - 複数のノードによってRead/Write 可能としてボリュームをマウントします。
- ReadWriteOncePod (RWOP) - 単一のPod からRead/Write 可能としてボリュームをマウントします。

なお、使用するKubernetes クラスタによってはRWX が使えないなどの制約があります。

{{<hint info>}}
RWO, ROX, RWX のアクセス元はいずれも"ノード"であることに注意してください。すなわち、RWO を選択したとしても、複数のPod が同一ノード上で稼働していれば、そのPV にはアクセスが可能です。
{{</hint>}}

既に作成したデータベースのマニフェストを見てみましょう。

```shell
cat ~/database/deployment.yaml
```

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: blog-db
      labels:
        app: blog-db
    spec:
      ...
      template:
        ...
        spec:
          containers:
          ...
            volumeMounts:
            - name: data
              mountPath: "/var/lib/pgsql/data"
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: blog-database

spec.template.spec.volumes に使用するPVC のリソースオブジェクト名が記載され、その請求されるPV がdata と名付けられ、さらにそれがspec.template.spec.containers[*].volumeMounts にてコンテナのパス/var/lib/pgsql/data にマウントされていることが分かります。
{{<hint info>}}
フロントのWeb サーバーにもPV がマウントされていますが、これはブログの記事ではなく画像を保存するためのものです。
{{</hint>}}

{{<details "補足" >}}
今回使用しているフロントエンドのWebアプリケーションでは、複数のノードにまたがる複数のPod から画像を保存するPV にアクセスさせることを可能にしたいため（そうしないとアクセスするWeb アプリケーションPod によって表示される画像が異なる可能性がある）、厳密にいえばReadWriteMany のPV を使うべきですが、環境によってはReadWriteMany が使えない場合があるため、特定のノードに固定してデプロイされるようにしています(マニフェストではaffinity として設定されています)。一方、データベースは1 つのインスタンスしか持たないので、アクセスモードがReadWriteOnce のストレージで十分です。
{{</details>}}

## データベースとの接続

フロントエンドのWebアプリケーションが動作しており、インターネットからアクセスできます。データの永続性を確保し、アプリケーションのすべてのインスタンスで同じデータベースを使用するために、フロントエンドのWebアプリケーションでは、すでに稼働しているPostgresql データベースを使用するように設定されています。

データベースのリソースを表示するには、以下を実行します。
```shell
kubectl get deployment,service,pvc,secret -l app=blog-db -o name
```
    deployment.apps/blog-db
    service/blog-db
    persistentvolumeclaim/blog-database
    secret/blog-credentials

Deployment とService、PVC の目的は先に説明した通りです。 Secret はデータベースの認証情報を保持するために使用されます。

データベースをフロントエンドのWeb アプリケーションにリンクさせるためには、Web アプリケーションにデータベースのホスト名とログイン認証情報を伝える必要があります。いくつか方法がありますが、まずは環境変数でパラメータを渡す方法を見てみます。

## 環境変数
フロントエンドWebアプリケーションにどのような環境変数がすでに設定されているかは、次のように実行すればわかります。
```shell
kubectl set env deployment/blog --list
```

以下のように表示されるはずです。

    #Deployment blog, container blog
    BLOG_SITE_NAME=EduK8S Blog
    DATABASE_HOST=blog-db
    #DATABASE_USER from secret blog-credentials, key database-user
    #DATABASE_PASSWORD from secret blog-credentials, key database-password
    #DATABASE_NAME from secret blog-credentials, key database-name

Pod 中のすべての環境変数を確認したい場合は以下のように実行することもできます。
```shell
DB=`kubectl get pod -l app=blog-db -o template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'. | head -1` 
kubectl exec $DB -- env
```

Web アプリケーションのDeployment のマニフェストを確認すると、環境変数が定義されていることが分かります。
```shell
cat ~/frontent/deployment.yaml
```
    spec:
      containers:
      - env:
        - name: BLOG_SITE_NAME
          value: EduK8S Blog
        - name: DATABASE_HOST
          value: blog-db

この場合は、ブログ名や接続するService 名を記載しています。my-svc.my-namespace.svc.cluster-domain.example のように厳密に記載することもできますが、このように省略することもできます。

## Secret

データベースの認証情報も同様にマニフェストに直接環境変数を書き込んで追加することができますが、Deployment のマニフェストとは別に管理するために、Secret を使うことが一般的です。このWeb アプリケーションでは、blog-credentialsというシークレットを見ることができます。

```shell
kubectl get secret/blog-credentials -o yaml
```

出力の中では、data セクションに値が入っているのがわかります。

    data:
      database-name: YmxvZw==
      database-password: dG9wLXNlY3JldA==
      database-user: YmxvZw==
    
表示されているのは、base64 エンコーディングで難読化されているため、実際の値ではありません。なお、デコードすると以下のようになります。
```shell
echo dG9wLXNlY3JldA== | base64 -d
```
    top-secret

このSecret をファイルとしてコンテナに渡すこともできますが、この例ではSecret の値をPod の環境変数として注入していることに注意してください。

やや込み入った内容ですが、どのデータベースにどのようにアクセスすべきか、環境変数やSecret といった方法を使って、フロントエンドのWeb アプリケーションに伝えることができたということを押さえれば十分です。