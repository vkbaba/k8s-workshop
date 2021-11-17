---
weight: 1
title: "基本操作"
---

## コンテナの起動
dockerを使ってアプリケーションをコンテナ内で起動するには、コンテナイメージが必要です。コンテナイメージは、アプリケーションを配布するためのパッケージメカニズムとして機能し、アプリケーションソフトウェア、オペレーティングシステムファイル、ライブラリ、その他アプリケーションの実行に必要なソフトウェアが含まれています。

アプリケーション用のあらかじめ構築されたコンテナイメージや、独自のコンテナイメージを構築するためのベースとなるコンテナイメージは、イメージレジストリを使用して配布されます。

イメージレジストリは、Docker HubとQuay.ioの2つの主要なホスト型サービスが存在します。また、GitHubやGitLabのようなGitリポジトリのホスティングサービスでも、コンテナイメージのホスティングと配布をサポートしています。

docker runを使ってイメージレジストリから既存のコンテナイメージをプルダウンして実行するには以下を実行します。

{{< highlight shell  >}}
docker run docker.io/busybox:latest date
{{< /highlight>}}

アウトプットは以下のようになります。
{{< highlight shell >}}
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
e2334dd9fee4: Pull complete
Digest: sha256:a8cf7ff6367c2afa2a90acd081b484cbded349a7076e7bdf37a05279f276bc12
Status: Downloaded newer image for busybox:latest
Mon Apr 20 00:11:37 UTC 2020
{{< /highlight>}}

このコマンドは、イメージをローカル環境にプルダウンするために行われた手順の詳細をログに記録します。この処理が完了すると、コンテナイメージからコンテナが起動され、コマンドラインで指定されたdateコマンドがコンテナ内で実行されます。dateコマンドはすぐに終了するので、コンテナはそのままシャットダウンされます。

今回使用したコンテナイメージは、busyboxというものです。これは、非常にミニマルなUnixライクのコンテナイメージです。最新バージョンのコンテナイメージを使用し、Docker Hubのイメージレジストリ（docker.io）から取得するように指定しました。

このコマンドをもう1回実行してみてください。
{{< highlight shell >}}
docker run docker.io/busybox:latest date
{{< /highlight>}}

dateコマンドがすぐに実行され、コンテナイメージを最初にプルダウンする必要があるという詳細情報は記録されないことがわかります。これは、コンテナイメージがローカル環境にキャッシュされ、次回以降の実行時に使用されるからです。

どのようなコンテナイメージがローカル環境にプルダウンされたかは、次のように実行することで確認できます。

{{< highlight shell >}}
docker images
{{< /highlight>}}

アウトプットは以下のようになります。

{{< highlight shell >}}
REPOSITORY  TAG     IMAGE ID      CREATED     SIZE
busybox     latest  be5888e67be6  5 days ago  1.22MB
{{< /highlight>}}

必要であれば、docker runはコンテナイメージを最初に必要とされるときにプルダウンします。実行される前にイメージをプルダウンしたい場合は、docker pullコマンドを使用することができます。

{{< highlight shell >}}
docker pull docker.io/busybox:latest
{{< /highlight>}}

## ターミナルの操作
先ほどはBusybox コンテナイメージから起動したコンテナ内で date コマンドを実行しました。コンテナイメージに含まれている任意のアプリケーションを実行することができます。

コンテナを作成して、その中でコマンドを実行するために対話したい場合は、対話型シェルを実行します。

{{< highlight shell >}}
docker run -it busybox sh
{{< /highlight>}}

対話型端末を必要とするコマンドを実行する場合は、docker runに-itオプションを指定する必要があります。これにより、ターミナルが確保され、stdinが入力可能な状態に保たれます。
★ 補足

また、今回はコンテナイメージの名前をbusyboxと略しています。バージョンタグが指定されていない場合は latest が使用されます。イメージレジストリのホストが指定されていない場合は、dockerのグローバル設定で定義されているデフォルトのイメージレジストリが検索されます。すでにDocker Hubからbusybox:latestのイメージを取り出しているので、それがマッチします。

コンテナ内で実行されているプロセスの一覧を見るには、次のように実行します。
{{< highlight shell >}}
ps
{{< /highlight>}}

以下のような出力が表示されるはずです。
{{< highlight shell >}}
pid user time command
    1 root 0:00 sh
    8 root 0:00 ps
{{< /highlight  >}}

コンテナのコンテキスト内で起動したプロセスのみが表示されます。下層のコンテナホスト（例えばコンテナを実行している仮想マシン）で実行されているプロセスは表示されません。

実行中のコンテナの一覧を表示するには、コンテナホスト上で実行します。

{{< highlight shell >}}
docker ps
{{< /highlight  >}}

上記で起動したコンテナが対話型シェルで実行されているのが確認できるはずです。

{{< highlight shell >}}
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS
            NAMES
ae576793e3e9        busybox             "sh"                32 seconds ago      Up 31 seconds
            recursing_leavitt
{{< /highlight  >}}

コンテナホストから既存のコンテナにアクセスし、その中でコマンドを実行するには、docker execコマンドを使用します。docker runと同様に、インタラクティブターミナルを必要とするコマンドを実行する場合は、-itオプションを使用します。

docker execを実行する際には、アクセスしたいコンテナのIDを指定する必要があります。最後に起動したコンテナのコンテナIDを表示するには、次のコマンドを実行します。
{{< highlight shell >}}
docker ps -ql
{{< /highlight  >}}

既存のコンテナ内で動作する2つ目の対話型端末を作成するには、次のように実行します。

{{< highlight shell >}}
docker exec -it `docker ps -ql` sh
{{< /highlight  >}}


元の対話型端末で再度 ps を実行します。
{{< highlight shell >}}
ps
{{< /highlight  >}}

これで、2つのシェルプロセスが実行されていることが確認できます。

1つ目の対話型ターミナルで次のように実行して終了します。

exit
このプロセスはコンテナ内で実行されているメインプロセスであり、プロセスIDは1であるため、コンテナがシャットダウンされ、2つ目の対話型端末のセッションも閉じられます。

## コンテナの停止
コンテナがシャットダウンされると、アプリケーションプロセスはなくなりますが、コンテナの状態のコピーは保持されます。停止したコンテナを含むすべてのコンテナの一覧は、次のように実行することで確認できます。

{{< highlight shell >}}
docker ps -a
{{< /highlight  >}}

いくつものコンテナを起動しているので、停止しているコンテナが複数表示されているはずです。

{{< highlight shell >}}
ONTAINER ID        IMAGE               COMMAND             CREATED              STATUS
 PORTS               NAMES
ae576793e3e9        busybox             "sh"                About a minute ago   Exited (0) 16 seconds ago
                     recursing_leavitt
e894e7d5790f        busybox:latest      "date"              3 minutes ago        Exited (0) 3 minutes ago
                     heuristic_bassi
9d1706a4b51f        busybox:latest      "date"              3 minutes ago        Exited (0) 3 minutes ago
                     inspiring_banzai
{{< /highlight  >}}

コンテナを実行するときは、フォアグラウンドで実行し、コンテナからの出力はすべてターミナルに表示されました。

また、コンテナからの出力はログファイルにも取り込まれました。コンテナIDを引数にしてdocker logsコマンドを実行すると、実行中または停止中のコンテナのログファイルを見ることができます。

{{< highlight shell >}}
docker logs `docker ps -ql`
{{< /highlight  >}}

コンテナのログファイルに加えて、コンテナ内からファイルシステムに加えられた変更のコピーも保存されるため、停止中のコンテナからファイルをコピーしたり、停止中のコンテナから新しいコンテナイメージを作成したりすることもできます。

停止したコンテナにはいくつかの用途がありますが、スペースを消費しますので、停止したコンテナを削除することが重要です。そうしないと、最終的にはディスクスペースが足りなくなってしまいます。

停止したコンテナを1つだけ削除するには、コンテナIDを引数にしてdocker rmを使用します。

{{< highlight shell >}}
docker rm `docker ps -ql`
{{< /highlight  >}}


これで、停止していた2つのコンテナだけが残るはずです。

{{< highlight shell >}}
docker ps -a
{{< /highlight  >}}

停止しているコンテナをすべて削除するには、次のように実行します。

{{< highlight shell >}}
docker rm $(docker ps -aq)
{{< /highlight  >}}

docker ps -aは停止中のコンテナだけでなく、稼働中のコンテナも反応しますが、docker rmコマンドは停止中のコンテナのみを削除します。

これでコンテナは残っていないはずです。

docker ps -aを実行します。
停止したコンテナをシャットダウンした後に操作する必要がないことがわかっている場合は、docker runの実行時に--rmオプションを使用できます。

{{< highlight shell >}}
docker run --rm busybox date
{{< /highlight  >}}

このオプションを使用すると、停止したコンテナは自動的に削除されます。

{{< highlight shell >}}
docker ps -a
{{< /highlight  >}}
