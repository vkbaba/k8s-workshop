---
weight: 6
title: "まとめ"
---

## まとめ
本ハンズオンでは以下のことを学習しました。
### 基本操作
- Kubernetes の概要
- kubectl を使った基本的な操作
- アプリケーション（ブログサイト）のデプロイ
### リソースの詳細
- Deployment、Replicaset、Pod の概要
- Pod のスケールアウト
### ネットワーク
- Kubernetes の名前空間
- Pod 間通信とCNI
- Servie リソース
- Ingress リソース
- （補足）Kubernetes におけるLoadBalancer
### ストレージとデータベース
- PV とPVC によるデータの永続化
- データベースとの接続のための環境変数とSecret リソース
### トラブルシューティング
- Kubernetes の場合のコンテナのログ取得
- Kubernetes の場合のコンテナへのアクセス


ラボを終了する場合は、ブラウザのタブを閉じるか、Terminate ボタンをクリックしてください。
{{< figure src="k_terminate.png" width="75%">}}

自分のPC でKubernetesを試してみたい方は、ぜひTanzu Community Edition を使ってみてください。

https://tanzucommunityedition.io/
