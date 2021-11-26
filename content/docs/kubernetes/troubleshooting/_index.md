---
weight: 5
title: "トラブルシューティング"
---
## 本セクションで学習すること
- Kubernetes の場合のコンテナのログ取得
- Kubernetes の場合のコンテナへのアクセス
  
## コンテナのロギング
アプリケーションの各インスタンスは、それぞれのポッドの中で実行されます。

すでに見たように、フロントエンドWeb サーバーのポッドをリストアップするには、次のようにします。
```shell
kubectl get pods -l app=blog -o name
```
アプリケーションの特定のインスタンスのログ出力にアクセスするには、kubectl logsコマンドでポッドの名前を使用できます。今回は複数のポッドがあるので、そのうちの1つの名前だけを取得する必要があります。

```shell
POD=`kubectl get pod -l app=blog -o template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'. | head -1` && echo $POD
```
次にkubectl logsを実行します。
```shell
kubectl logs $POD
```

{{<hint info>}}
ポッド内に複数のコンテナがある場合は、-c または --container オプションを使用してコンテナに名前を付ける必要があります。
また、実行中のアプリケーションの出力を追跡したい場合は、-fまたは--followオプションを使用できます。
{{</hint>}}


## コンテナへのアクセス
コンテナにアクセスしてコマンドを実行するには、kubectl execを使用します。ロギングと同様に、アクセスしたい特定のポッドを指定する必要があり、ポッド内で複数のコンテナが動作している場合は、-cまたは--containerオプションを使用してどのコンテナかを指定します。

```shell
kubectl exec $POD env
```

対話型のターミナルセッションを実行したい場合は、docker と同様、-iまたは--stdinオプションと、-tまたは--ttyオプションを使用する必要があります。

```shell
kubectl exec -it $POD -- bash
```

一通り確認したら、対話型シェルを終了してください。
```shell
exit
```

このあたりはDocker ととても似ていますね。



