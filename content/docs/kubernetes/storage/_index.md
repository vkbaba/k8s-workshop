---
weight: 5
title: "ストレージとデータベース"
---

## ストレージの基本
繰り返しになりますが、Pod は短命です。Pod は簡単に削除されるため、データを残したい場合は何かしらの形で外部に保存する仕組みが必要です。その1 つがPersistent Volume （PV）になります。

PV はその名の通り永続ストレージであり、PV の中にデータを保存することができます。PV をPod に割り当てるにはいくつか方法がありますが、代表的なものとしてPersistent Volume Claim (PVC) を使う方法があります。PVC はPV を要求するために使われるリソースであり、そのマニフェストの中には例えば要求するストレージのサイズなどが定義されます。Pod がデータを保存するためにPV を使いたい場合は、直接PV をアタッチするのではなく、PVC をアタッチします。

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
今回使用しているフロントエンドのWebアプリケーションでは、複数のノードにまたがる複数のPod から画像を保存するPV にアクセスさせることを可能にしたいため、厳密にいえばReadWriteMany のPV を使うべきですが、環境に寄ってはReadWriteMany が使えない場合があるため、特定のノードに固定してデプロイされるようにしています(マニフェストではaffinity として設定されています)。一方、データベースは1 つのインスタンスしか持たないので、アクセスモードがReadWriteOnceのストレージで十分です。
{{</details>}}


## データベースとの接続

フロントエンドのWebアプリケーションが動作しており、公共のインターネットからアクセスできます。この時点では、ファイルベースのSQLiteデータベースを使用しています。このデータベースは各インスタンスに対してローカルであり、すべてのインスタンス間で共有されていないため（スケールアップしたアプリケーションには不向き）、ポッドが再起動するたびにデータへの変更が失われます。

データの永続性を確保し、アプリケーションのすべてのインスタンスで同じデータベースを使用するために、フロントエンドのWebアプリケーションでは、すでに稼働している別のPostgresqlデータベースを使用するように設定します。

データベースのリソースを表示するには、以下を実行します。

kubectl get deployment,service,pvc,secret -l app=blog-db -o name

    deployment.apps/blog-db
    service/blog-db
    persistentvolumeclaim/blog-database
    secret/blog-credentials

これで、デプロイメントとサービスの目的が理解できたはずです。persistentvolumeclaim は、データベースが使用する永続的なストレージを要求するために使用されます。secret は、データベースの認証情報を保持するために使用されます。

データベースをフロントエンドのウェブアプリケーションにリンクさせるためには、フロントエンドのウェブアプリケーションにデータベースのホスト名とログイン認証情報を伝える必要があります。そのためには、フロントエンド Web アプリケーションの配置に環境変数の設定を追加する必要があります。

フロントエンドWebアプリケーションにどのような環境変数がすでに設定されているかは、次のように実行すればわかります。

kubectl set env deployment/blog --list
これで表示されるはずです。

# デプロイメントブログ、コンテナブログ
BLOG_SITE_NAME=EduK8Sブログ
フロントエンドWebアプリケーションの実装方法では、別のデータベースのために以下の環境変数を期待しています。

DATABASE_HOST - データベースのホスト名です。
DATABASE_USER - データベースにログインするユーザーです。
DATABASE_PASSWORD - データベースにログインするユーザーのパスワードです。
DATABASE_NAME - データベースの名前です。
環境変数を設定するために、kubectlにはkubectl set envというコマンドが用意されています。本番の設定を更新するのではなく、ローカルに設定を保持するのですが、その設定に必要な変更を行うために使用します。

データベースホストについては、ホスト名はデータベースサービスオブジェクトの名前、つまりblog-dbとなります。

このセットでデプロイメント構成がどうなるかを確認するには、次のように実行します。

kubectl set env deployment/blog DATABASE_HOST=blog-db --dry-run -o yaml
出力結果を見ると、既存のBLOG_SITE_NAME環境変数に加えて、DATABASE_HOST環境変数が追加されていることがわかります。これは、spec.template.spec.container.envの設定の下にあります。