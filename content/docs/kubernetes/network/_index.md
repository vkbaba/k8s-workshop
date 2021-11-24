---
weight: 4
title: "ネットワーク"
---
## 名前空間
ネットワークの前に簡単にNamespace (名前空間) に関して説明します。名前空間はKubernetes におけるリソースを適切に分離する仕組みのことです。例えば、1つのKubernetes クラスタを複数で利用する際に、名前空間を分けない場合はアプリケーションの名前が被ってはいけませんが、名前空間を分けることで、このような重複を気にする必要がなくなります。他にも、名前空間の間の通信を制御したり、機能を制限したりと、Kubernetes におけるマルチテナントの提供形態の1 つと言い換えることもできます。

例えば、これまでのハンズオンの中でkubectl を使った操作の時には名前空間を意識していませんでしたが、実際はハンズオンのセッションごとに個別に割り当てられた名前空間の中で作業をしていました。

名前空間を指定しない場合：
```shell
kubectl get pod
```
名前空間を指定する場合：
```shell
kubectl get pod -n ${NS}
```
{{<hint info>}}
通常、何も設定しない場合のデフォルトの名前空間の名前は"default"となります。
{{</hint>}}

このハンズオンの仕組みとして、ユーザーのセッションごとに個別に名前空間が割り当てられ、みなさんはその名前空間の中で作業しています。
```shell
kubectl get namespaces
```
    NAME                   STATUS   AGE
    default                Active   68d
    eduk8s                 Active   68d
    eduk8s-ingress         Active   68d
    eduk8s-labs-ui         Active   68d
    eduk8s-labs-w01        Active   68d
    eduk8s-labs-w01-s283   Active   172m
    eduk8s-labs-w01-s284   Active   109m
    eduk8s-labs-w01-s285   Active   46m
    ...

ただし、他のユーザーに割り当てられた名前空間やKubernetes が内部で使用している名前空間へのアクセスは制限されています。名前空間ごとに適切にアクセスコントロールができていると言えますね。

```shell
kubectl get pod -n kube-system
```

    Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:eduk8s-labs-w01:eduk8s-labs-w01-s283" cannot list resource "pods" in API group "" in the namespace "kube-system"


## Kubernetes のネットワークの基本
Kubernetes ではPod 単位でIP アドレスがアサインされます。各ポッドに割り当てられているIPアドレスは、次のように実行することで確認できます。

```shell
kubectl get pods -l app=blog -o wide
```
    NAME                    READY   STATUS    RESTARTS   AGE   IP               NODE                                       NOMINATED NODE   READINESS GATES
    blog-65c8484545-br9mf   1/1     Running   0          28m   192.168.12.67    ip-10-0-3-148.us-east-2.compute.internal   <none>           <none>
    blog-65c8484545-qbvvp   1/1     Running   0          49m   192.168.12.111   ip-10-0-3-148.us-east-2.compute.internal   <none>           <none>

これらのIPアドレスは、Kubernetesクラスタ内でのみアクセス可能です。直接、外部からアクセスすることはできません。

Pod は複数のコンテナを含む場合は、そのコンテナ間の通信はlocalhost で実行され、待ち受けるポートは重複してはいけません。例えば、1つのPod の中に80番ポートで待ち受けるnginx コンテナを2つ以上含めることはできません。

Pod 間の通信は、CNI (Container Network Interface) プラグインが担います。つまり、CNI は、このようにPod によしなにIP アドレスを振って、Pod 同士が通信できるよう取り計らう役割を持っています。Kubernetes クラスタを作成する場合CNI プラグインのインストールが必要なため、今回のハンズオン環境では確認することはできませんが、この環境にも当然CNI はインストールされています。CNI の例としては、[Calico](https://www.tigera.io/project-calico/) や[Antrea](https://antrea.io/)
が挙げられます。

## Service リソース
次に、Pod と外部との通信ですが、これはKubernetes のリソースであるService リソースを介して行われます。先ほど説明した通り、Pod のIPアドレスは、Kubernetesクラスタ内でのみアクセス可能ですので、直接外部ネットワークからアクセスすることは基本的にできません。

加えて、Pod は短命です。Replicaset の役割はPod をレプリカの数だけ稼働させることであり、停止したPod を再起動させるわけではなく、再作成します。この場合はPod の名前やIPアドレスも変更されます。すなわち、通信先のコンポーネントのIP アドレスは変わるものとしてアプリケーションを設計しなければなりません。

アプリケーションに安定したIPアドレスとホスト名を割り当て、かつ外部との接続を実現するために、Kubernetes にはService リソースと呼ばれるリソースがあります。
{{<hint info>}}
Service という名前がややこしいですが、Pod やDeployment などと同様、Kubernetes にService という名前のリソースがあります。
{{</hint>}}
    
さっそくService リソースのマニフェストを確認します。
```shell
cat ~/frontend/service.yaml
```

    apiVersion: v1
    kind: Service
    metadata:
    name: blog
    labels:
        app: blog
    spec:
    type: ClusterIP
    selector:
        app: blog
    ports:
    - name: 8080-tcp
        port: 8080
        protocol: TCP
        targetPort: 8080

これをkubectl apply すると、blog という名前のService が作成されたことになります。このマニフェストは「app:blog というラベルが付いたアプリケーションのためにポート8080 を公開し、Pod が待ち受けているポート8080 にマッピングします」ということを意味しています。

既にこのService は作成されているため、Service の詳細を確認してみましょう。

```shell
kubectl get service --selector app=blog -o wide 
```
これにより、以下のような出力が表示されます。

    NAME   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE   SELECTOR
    blog   ClusterIP   10.108.113.113   <none>        8080/TCP   28m   app=blog

SELECTOR に注目しましょう。これはマニフェスト中のspec.selector の値です。

    selector:
        app: blog

これは、どのPod をService のエンドポイントとして登録するかを定義していて、これはラベルによって制御されます。すなわち下記のコマンドの実行で得られるPod をService のエンドポイントとして登録し、Service のIP アドレスでアクセスできるようにします。

```shell
kubectl get pods -l app=blog -o name
```
{{<hint info>}}
ラベル管理を適切にしないと、意図しないPod がService のバックエンドに配置される場合があります。
{{</hint>}}

なお、サービスに対して登録されているポッドのIPアドレスは、次のように実行することで確認できます。
```shell
kubectl get endpoints blog
```
Service のIPアドレスはそのService が存続する限り変更されることはありませんが、それでもそのIP は使用すべきではありません。代わりに、Service に対応するホスト名を使用するべきです。ホスト名は自動的にKubernetesクラスタの内部DNSに登録され、クラスタ内のどのアプリケーションでも使用できます。ホスト名の命名規則については下記ドキュメントを参照してください。

参考：[https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

注意点として、今回作成したService に対して、Kubernetesクラスタの外から直接そのIP アドレスやホスト名にアクセスすることはできません。これは、今回作成したService がClusterIP と呼ばれるタイプであり、Pod のIP アドレスと同様、ClusterIP はクラスタ内で使用される安定したIP アドレスやホスト名の提供を目的としているためです。

{{<details "補足" >}}
実はこの環境ではmy-svc.my-namespace.svc.cluster-domain.example （もしくはIP アドレス）のような形でターミナルからアクセスできます。本環境の場合は以下のようになります。
```shell
curl http://blog.${NS}.svc.cluster.local:8080
```
ただし、ターミナルの外、すなわちブラウザの別タブなどからアクセスすることはできません。これは、今回のハンズオン環境で提供されるターミナルの実体がまさにKubernetes のPod であり、そこからcurl を実行するということは、クラスタ内での通信に他ならないからです。
{{</details>}}

## Ingress によるService の公開
Service をKubernetes クラスタの外からアクセスできるように公開するためには、Ingressリソースオブジェクトを作成する必要があります。

今回使用するIngressリソースオブジェクトの定義を見るには、runを使用します。

```shell
cat ~/frontend/ingress.yaml
```

    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
    name: blog
    labels:
        app: blog
    spec:
    rules:
    - host: blog-eduk8s-labs-w01-s284.tdc-reg-prod-d66138e.tanzu-labs.esp.vmware.com
        http:
        paths:
        - path: "/"
            backend:
            serviceName: blog
            servicePort: 8080

この例では、blog-eduk8s-lab-w01-s284.tdc-reg-prod-d66138e.tanzu-lab.esp.vmware.comというホスト名で受信したHTTPリクエストは、blogという名前を持つService（つまりアプリケーション）に向けられるべきだというルールになっています。

外部のユーザーがWebブラウザからホスト名にアクセスする場合、HTTPトラフィックに標準のポート80を使用しますが、ルーターはそのトラフィックをサービスのポート8080に渡します。

Ingress も既に作成しているので、以下のコマンドで確認することができます。
```shell
kubectl get ingress -l app=blog
```

払い出されたURL を使ってWebブラウザからフロントエンドのWebアプリケーションにアクセスできます。ターミナルではなく、ブラウザの別タブから払い出されたURL にアクセスしてください。
```shell
NS=$(kubectl config view -o jsonpath='{.contexts[].context.namespace}')
echo $NS
```
```shell
http://blog-${NS}.tdc-reg-prod-d66138e.tanzu-labs.esp.vmware.com
```

{{<hint info>}}
1つ目のコマンドはこの環境で払い出されるURL 名の一部（名前空間）を取得しています。URL 中の${NS} は取得した名前空間で置き換えてください。例えば下記のようになります。

http://blog-eduk8s-labs-w01-s287.tdc-reg-prod-d66138e.tanzu-labs.esp.vmware.com

{{</hint>}}

{{<hint warning>}}
この時点では 500 Internal Server Error が表示されることに注意してください。
{{</hint>}}

このリンクをクリックして、フロントエンドのWebアプリケーションにアクセスします。利用できないと表示された場合は、利用できるようになるまでページを更新してください。これは、Ingress ルーティングの再設定に時間がかかるためです。

これは、このホストのトラフィックをKubernetesクラスタのルータに向けるために、外部のドメインネームサーバ（DNS）でワイルドカードCNAMEがすでに事前に設定されているために機能することに注意してください。これが自分のKubernetesクラスタであれば、使用したいホストのDNSに適切なCNAMEを設定する必要があります。