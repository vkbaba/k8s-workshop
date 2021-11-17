---
weight: 2
title: "Web サーバーの公開"
---

## ネットワークサービス
これまでに実行したすべてのコンテナは、実行元のターミナルに接続されたままでした。コンテナからの出力はターミナルに表示され、コンテナを停止したときに初めてコマンドプロンプトが戻ってきます。

コンテナ内で実行される長時間稼働のネットワークサービスは、ターミナルから切り離してバックグラウンドプロセスとして実行する必要があります。

busyboxイメージを使用してWebサーバーを実行するには、次のように実行します。

{{< highlight shell  >}}
docker run --rm -d --name httpd -p 8080:80 busybox httpd -f -vv
{{< /highlight>}}

docker runに-dオプションを付けると、コンテナがターミナルから切り離されてバックグラウンドで実行されます。

このコンテナをより簡単に識別して操作できるように、--nameオプションを使ってhttpdという名前をつけます。

Webサーバはネットワークサービスなので、公開するネットワークポートを指定する必要があります。これには-pオプションを使用します。

最後に、コンテナ内で実行するコマンドとして、httpd -f -vvを使用します。

httpdの-fオプションは、ウェブサーバがコンテナのコンテキスト内でフォアグラウンドプロセスとして実行されるようにします。これが行われないと、コンテナはすぐに終了してしまいます。vv は詳細なロギングを可能にします。

コンテナが稼働していることを確認するには、次のように実行します。

{{< highlight shell  >}}
docker ps
{{< /highlight >}}

コンテナからの出力を尾行するには、次のように実行します。

{{< highlight shell  >}}
docker logs -f httpd
{{< /highlight >}}

コンテナIDの代わりに、コンテナに割り当てたhttpd名を使用します。f オプションは、ログファイルを継続的にテーリングすることを意味します。

初期状態ではログが出力されていませんが、次のように実行して、Webサーバに対してWebリクエストを行ってください。

{{< highlight shell  >}}
curl localhost:8080
{{< /highlight >}}

を実行すると、リクエストの詳細がログに記録されるはずです。

このケースでは、提供すべきファイルを提供していないため、Webサーバからエラーが発生しています。

コンテナがターミナルから切り離されたので、コンテナを停止するには以下を実行する必要があります。

{{< highlight shell  >}}
docker stop --time 2 httpd
{{< /highlight >}}

## 演習
busybox ではなくnginx イメージを使って、コンテナでWeb サーバーをバックグラウンドで実行してみてください。また、curl で接続して、応答が返ってくることを確認しましょう。次に、nginx のログを出力してみて、実際にアクセスがあったことを確かめてみてください。最後に、実行したコンテナを停止し、削除をしてください。

{{<details "解答" >}}
初めてnginx を実行する際はイメージキャッシュが保存されていないため、自動的にレジストリからダウンロードされます。
```shell
docker run --rm -d --name nginx -p 8080:80 nginx
```
割り当てたポートに対してcurl を実行してみましょう。
```shell
curl localhost:8080
```
以下のように、"Welcome to nginx!" と表示されたらOK です。
```shell
[~/exercises] $ curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
アクセスログも見てみましょう。
```shell
docker logs -f nginx
```
自身のIP アドレス（ここでは172.17.0.1）からHTTP GET リクエストが実行されていることが分かりますね。
```shell
172.17.0.1 - - [17/Nov/2021:12:17:15 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.66.0" "-"
```
最後にコンテナを削除します。起動時に--rm オプションを付けたので、docker stop で自動的に削除されます。
```shell
docker stop nginx
```
{{</details >}}