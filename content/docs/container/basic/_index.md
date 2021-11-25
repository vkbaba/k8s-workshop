---
weight: 1
title: "基本操作"
---
## 本セクションで学習すること
- コンテナの概要
- コンテナの起動、停止、削除といった基本的な操作
- コンテナの中に入った操作
- コンテナのログ出力

## コンテナの起動
dockerを使ってアプリケーションをコンテナ内で起動するには、コンテナイメージが必要です。コンテナイメージは、アプリケーションを配布するためのパッケージメカニズムとして機能し、アプリケーションソフトウェア、オペレーティングシステムファイル、ライブラリ、その他アプリケーションの実行に必要なソフトウェアが含まれています。

{{< figure src="c_image.png" width="75%">}}

{{<hint info>}}
コンテナイメージは仮想アプライアンスに近いですが、OS カーネルを含まないため非常に軽量です。
{{</hint>}}

アプリケーション用のあらかじめ構築されたコンテナイメージや、独自のコンテナイメージを構築するためのベースとなるコンテナイメージは、イメージレジストリを使用して配布されます。

イメージレジストリは、[Docker Hub](https://hub.docker.com/) のようなホスト型サービスが存在します。また、[Github](https://github.com/features/packages) や[GitLab](https://docs.gitlab.com/ee/user/packages/container_registry/)のようなGit リポジトリのホスティングサービスでも、コンテナイメージのホスティングと配布をサポートしています。

さっそくコンテナをデプロイしてみましょう。docker runを使ってイメージレジストリから既存のコンテナイメージをプルダウンして実行するには以下を実行します。

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

<!-- {{<details エラーが出た場合>}}
```shell
docker run public.ecr.aws/runecast/busybox:latest
```
{{</details >}} -->

このコマンドは、イメージをローカル環境にプルダウンするために行われた手順の詳細をログに記録します。この処理が完了すると、コンテナイメージからコンテナが起動され、コマンドラインで指定されたdateコマンドがコンテナ内で実行されます。date コマンドはすぐに終了するので、コンテナはそのままシャットダウンされます。

今回使用したコンテナイメージは、busybox というものです。これは、非常にミニマルなUnix ライクのコンテナイメージで、よく利用されるコマンドを詰め込んでいる便利なイメージです。最新バージョンのコンテナイメージを使用し(:latest)、Docker Hub のイメージレジストリ(docker.io)から取得するように指定しました。

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

必要であれば、docker run はコンテナイメージを最初に必要とされるときにプルダウンします。実行される前にイメージをプルダウンしたい場合は、docker pullコマンドを使用することができます。

```shell
docker pull docker.io/busybox:latest
```

## ターミナルの操作
先ほどはbusybox コンテナイメージから起動したコンテナ内で date コマンドを実行しました。コンテナイメージに含まれている任意のアプリケーションを実行することができます。

コンテナを作成して、その中でコマンドを実行するために対話したい場合は、対話型シェルを実行します。

```shell
docker run -it busybox sh
```

対話型端末を必要とするコマンドを実行する場合は、docker runに-itオプションを指定する必要があります。これにより、ターミナルが確保され、stdin が入力可能な状態に保たれます。つまり、手元のターミナルで入力したコマンドをコンテナの中で実行し、その結果をターミナルで受け取れるようになります。

{{< hint info >}}
「コンテナの中に入って何か操作をしたい場合 = -it」 と最初は覚えてしまいましょう。
{{< /hint >}}

また、今回はコンテナイメージの名前をbusybox と略しています。バージョンタグが指定されていない場合は latest が使用されます。イメージレジストリのホストが指定されていない場合は、docker のグローバル設定で定義されているデフォルトのイメージレジストリが検索されます。ここではDocker Hub からイメージを取得しました。

コンテナ内で実行されているプロセスの一覧を見るには、次のように実行します。
```shell
ps
```
以下のような出力が表示されるはずです。

    pid user time command
        1 root 0:00 sh
        8 root 0:00 ps

コンテナのコンテキスト内で起動したプロセスのみが表示されます。下層のコンテナホスト(例えばコンテナを実行している仮想マシン)で実行されているプロセスは表示されません。

実行中のコンテナの一覧を表示するには、コンテナホスト上で実行します。**ターミナル2** でdocker ps コマンドを試してみてください(ターミナル1 はコンテナの中に入ったままです)。  

```shell
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

停止したコンテナにはいくつかの用途がありますが、スペースを消費しますので、停止したコンテナを削除することが重要です。そうしないと、最終的にはディスクスペースが足りなくなってしまいます。

停止したコンテナを1つだけ削除するには、コンテナID を引数にしてdocker rmを使用します。ここでは、docker ps -ql でコンテナID を取得しています。

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
{{< /hint>}}

```shell
docker ps -a
```



## 演習
1. コンテナはOS を含まないと説明しましたが、コンテナイメージには下記のようにubuntu やcentos があります。この理由は何でしょうか？
   1. https://hub.docker.com/_/ubuntu 
   2. https://hub.docker.com/_/centos
2. 本セクションの一番最初のコマンドは下記の通りですが、date コマンドを入力しない場合はどうなるでしょうか？この理由は何でしょうか？また、docker exec を使わず、どうすればbusybox を一定時間起動しておくことができるか考えてみてください。
```shell
docker run docker.io/busybox:latest date
```
{{<details "解答" >}}
1. 正しくはOS "カーネル" を含まない、です。ubuntu やcentos をコンテナで実行したとしても、それらはホストOS を共有しているため、cat /proc/version で調べられるカーネルのバージョンは同一になります。これらの違いは、Linux を使いやすくするために提供されるソフトウェア群、例えばapt (ubuntu) とyum (centos) などの違いに現れます。
2. まずはdate コマンドなしで実行してみましょう。
```shell
docker run docker.io/busybox:latest
```
何も出力されず、docker ps で調べても起動中のコンテナはありません。
```shell
docker ps
```
    CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

```shell
docker ps -a
```
出力は以下のようになります。この例ですとコンテナ自体は作成されましたが、5 秒後にExited(終了)していることが分かります。

    CONTAINER ID   IMAGE            COMMAND                  CREATED          STATUS                      PORTS     NAMES
    37341b4975aa   busybox:latest   "sh"                     6 seconds ago    Exited (0) 5 seconds ago              hopeful_zhukovsky

これは、コンテナの中で実行されたメインプロセスが実行され、終了したためです。言い換えると、date コマンドを入力した場合は、date コマンドを実行することが唯一の役割であり、日付を表示するという役割を終えたためにコンテナは停止しました。何も実行しない場合は、出力の通りsh コマンドを実行し（何も起こらなかったように見えますが）、コンテナは停止します。

ではbusybox を長時間起動するにはどうしたらよいでしょうか？docker exec でコンテナの中に入ることで、コンテナがすぐに終了することを防ぐこともできますが、コンテナから抜けた場合は停止してしまいます。

そこで、これを実現する典型的な1 つの方法として、sleep コマンドを使います。
```shell
docker run docker.io/busybox:latest sleep 10
```
これは、10 秒待ってからコンテナが停止します。ただ、上記の方法だと実行中そのターミナルでは操作できないため、\-d オプションをつけることでバックグラウンドでの実行が可能です。
```shell
docker run -d docker.io/busybox:latest sleep 60
```
docker ps で稼働中のコンテナが見えますね。
```shell
docker ps 
```

    CONTAINER ID   IMAGE            COMMAND      CREATED          STATUS          PORTS     NAMES
    5c517a3f2493   busybox:latest   "sleep 60"   20 seconds ago   Up 19 seconds             nervous_vaughan

sleep が実行されているコンテナにexec することもできますので、何らかの検証やトラブルシューティングの際に活用できる場合があります。

{{</details >}}
