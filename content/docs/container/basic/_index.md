---
weight: 1
title: "基本操作"
---

## コンテナの起動
dockerを使ってアプリケーションをコンテナ内で起動するには、コンテナイメージが必要です。コンテナイメージは、アプリケーションを配布するためのパッケージメカニズムとして機能し、アプリケーションソフトウェア、オペレーティングシステムファイル、ライブラリ、その他アプリケーションの実行に必要なソフトウェアが含まれています。

{{< figure src="c_image.png" width="75%">}}

アプリケーション用のあらかじめ構築されたコンテナイメージや、独自のコンテナイメージを構築するためのベースとなるコンテナイメージは、イメージレジストリを使用して配布されます。

イメージレジストリは、[Docker Hub](https://hub.docker.com/) のようなホスト型サービスが存在します。また、[Github](https://github.com/features/packages) や[GitLab](https://docs.gitlab.com/ee/user/packages/container_registry/)のようなGitリポジトリのホスティングサービスでも、コンテナイメージのホスティングと配布をサポートしています。

docker runを使ってイメージレジストリから既存のコンテナイメージをプルダウンして実行するには以下を実行します。

```shell
docker run docker.io/busybox:latest date
```

アウトプットは以下のようになります。

    Unable to find image 'busybox:latest' locally
    latest: Pulling from library/busybox
    e2334dd9fee4: Pull complete
    Digest: sha256:a8cf7ff6367c2afa2a90acd081b484cbded349a7076e7bdf37a05279f276bc12
    Status: Downloaded newer image for busybox:latest
    Mon Apr 20 00:11:37 UTC 2020


このコマンドは、イメージをローカル環境にプルダウンするために行われた手順の詳細をログに記録します。この処理が完了すると、コンテナイメージからコンテナが起動され、コマンドラインで指定されたdateコマンドがコンテナ内で実行されます。dateコマンドはすぐに終了するので、コンテナはそのままシャットダウンされます。

今回使用したコンテナイメージは、busyboxというものです。これは、非常にミニマルなUnixライクのコンテナイメージで、よく利用されるコマンドを詰め込んでいる便利なイメージです。最新バージョンのコンテナイメージを使用し（:latest）、Docker Hubのイメージレジストリ（docker.io）から取得するように指定しました。

このコマンドをもう1回実行してみてください。
```shell
docker run docker.io/busybox:latest date
```

dateコマンドがすぐに実行され、コンテナイメージを最初にプルダウンする必要があるという詳細情報は記録されないことがわかります。これは、コンテナイメージがローカル環境にキャッシュされ、次回以降の実行時に使用されるからです。

どのようなコンテナイメージがローカル環境にプルダウンされたかは、次のように実行することで確認できます。

```shell
docker images
```

アウトプットは以下のようになります。

    REPOSITORY  TAG     IMAGE ID      CREATED     SIZE
    busybox     latest  be5888e67be6  5 days ago  1.22MB

必要であれば、docker runはコンテナイメージを最初に必要とされるときにプルダウンします。実行される前にイメージをプルダウンしたい場合は、docker pullコマンドを使用することができます。

```shell
docker pull docker.io/busybox:latest
```

## ターミナルの操作
先ほどはbusybox コンテナイメージから起動したコンテナ内で date コマンドを実行しました。コンテナイメージに含まれている任意のアプリケーションを実行することができます。

コンテナを作成して、その中でコマンドを実行するために対話したい場合は、対話型シェルを実行します。

```shell
docker run -it busybox sh
```

対話型端末を必要とするコマンドを実行する場合は、docker runに-itオプションを指定する必要があります。これにより、ターミナルが確保され、stdinが入力可能な状態に保たれます。

参考 : [https://ohbarye.hatenablog.jp/entry/2019/05/05/learn-tty-with-docker](https://ohbarye.hatenablog.jp/entry/2019/05/05/learn-tty-with-docker)

{{< hint info >}}
「コンテナの中に入って何か操作をしたい場合 = -it」 と最初は覚えてしまいましょう。
{{< /hint >}}


また、今回はコンテナイメージの名前をbusyboxと略しています。バージョンタグが指定されていない場合は latest が使用されます。イメージレジストリのホストが指定されていない場合は、dockerのグローバル設定で定義されているデフォルトのイメージレジストリが検索されます。すでにDocker Hubからbusybox:latestのイメージを取り出しているので、それがマッチします。

コンテナ内で実行されているプロセスの一覧を見るには、次のように実行します。
```shell
ps
```
以下のような出力が表示されるはずです。

    pid user time command
        1 root 0:00 sh
        8 root 0:00 ps

コンテナのコンテキスト内で起動したプロセスのみが表示されます。下層のコンテナホスト（例えばコンテナを実行している仮想マシン）で実行されているプロセスは表示されません。

実行中のコンテナの一覧を表示するには、コンテナホスト上で実行します。ターミナル2 でdocker ps コマンドを試してみてください（ターミナル1 はコンテナの中に入ったままです）。  

```shell
exit
docker ps
```
上記で起動したコンテナが対話型シェルで実行されているのが確認できるはずです。

    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS             NAMES
    ae576793e3e9        busybox             "sh"                32 seconds ago      Up 31 seconds                         recursing_leavitt

コンテナホストから既存のコンテナにアクセスし、その中でコマンドを実行するには、docker execコマンドを使用します。docker runと同様に、インタラクティブなターミナルを必要とするコマンドを実行する場合は、-itオプションを使用します。

docker execを実行する際には、アクセスしたいコンテナのIDを指定する必要があります。最後に起動したコンテナのコンテナIDを表示するには、次のコマンドを実行します。
```shell
docker ps -ql
```
では、起動中のコンテナの中にexec コマンドで入ってみましょう。
```shell
docker exec -it `docker ps -ql` sh
```

再度 ps を実行します。
```shell
ps
```
これで、2つのシェルプロセスが実行されていることが確認できます。

1つ目の対話型ターミナルで次のように実行して終了します。
```shell
exit
```
このプロセスはコンテナ内で実行されているメインプロセスであり、プロセスIDは1であるため、コンテナがシャットダウンされ、2つ目の対話型端末のセッションも閉じられます。

## コンテナの停止
コンテナがシャットダウンされると、アプリケーションプロセスはなくなりますが、コンテナの状態のコピーは保持されます。停止したコンテナを含むすべてのコンテナの一覧は、次のように実行することで確認できます。

```shell
docker ps -a
```
いくつものコンテナを起動しているので、停止しているコンテナが複数表示されているはずです。


    ONTAINER ID        IMAGE               COMMAND             CREATED              STATUS
    PORTS               NAMES
    ae576793e3e9        busybox             "sh"                About a minute ago   Exited (0) 16 seconds ago
                        recursing_leavitt
    e894e7d5790f        busybox:latest      "date"              3 minutes ago        Exited (0) 3 minutes ago
                        heuristic_bassi
    9d1706a4b51f        busybox:latest      "date"              3 minutes ago        Exited (0) 3 minutes ago
                        inspiring_banzai

コンテナからの出力はログファイルにも取り込まれます。コンテナIDを引数にしてdocker logsコマンドを実行すると、実行中または停止中のコンテナのログファイルを見ることができます。

```shell
docker logs `docker ps -ql`
```
コンテナのログファイルに加えて、コンテナ内からファイルシステムに加えられた変更のコピーも保存されるため、停止中のコンテナからファイルをコピーしたり、停止中のコンテナから新しいコンテナイメージを作成したりすることもできます。
{{<details 補足>}}
どこにあるか
{{</details>}}

停止したコンテナにはいくつかの用途がありますが、スペースを消費しますので、停止したコンテナを削除することが重要です。そうしないと、最終的にはディスクスペースが足りなくなってしまいます。

停止したコンテナを1つだけ削除するには、コンテナIDを引数にしてdocker rmを使用します。

```shell
docker rm `docker ps -ql`
```

これで、停止していた2つのコンテナだけが残るはずです。

```shell
docker ps -a
```
停止しているコンテナをすべて削除するには、次のように実行します。

```shell
docker rm $(docker ps -aq)
```
docker ps -aは停止中のコンテナだけでなく、稼働中のコンテナも反応しますが、docker rmコマンドは停止中のコンテナのみを削除します。

これでコンテナは残っていないはずです。

停止したコンテナをシャットダウンした後に操作する必要がないことがわかっている場合は、docker runの実行時に\-\-rmオプションを使用できます。

```shell
docker run --rm busybox date
```
このオプションを使用すると、停止したコンテナは自動的に削除されます。
{{< hint info >}}
検証目的においては常に\-\-rm オプションをつけるよう意識しておくとディスクスペースの節約につながります。
{{< hint>}}
```shell
docker ps -a
```