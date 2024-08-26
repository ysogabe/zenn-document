---
title: "DifyをAzureですこしセキュアに利用する"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dify", "azure", "kubernetes"]
published: false
---

# この記事は

最近、流行っている[Dify][def]のCommutiy版をAzure上に「すこし」本番運用を考慮した形でデプロイしたので、自身のメモをかねた記事です。

# 構成

## 基本方針

Difyは結構たくさんのDocker Containerが[動作](https://github.com/langgenius/dify/blob/main/docker/docker-compose.yaml)します。
また、PostgreSQL、Redis、VectorDBなどが必要となります。

構築あたり、以下の方針としました

1. VMは極力使わない
  コンテナがたくさん立ち上がるなら、K8Sだよね？程度の思いです。AzureにはContainer AppsというサーバレスK8Sサービス（？）があり、まぁこっちを選んだ方が楽なのは知っていますが。まぁ、私の趣味です（今、開発・検証環境のAKSをもっており、同居させた方が楽だった、、、が本音ですが）

  - コンテナはAzure Kubernetes Service（AKS）を利用する
  - DBなどは、できるだけAzureのPaaSを利用する

2. 外部と直接通信する箇所を限定する
  L7ロードバランサー以外は直接外部から接続出来ないように、仮想ネットワークで分離する方針としました。

  - Ingress以外はPrivateなVnetで通信する
  - PaaSもVnetに対応したサービスを選定する

3. セキュリティ情報を安全に管理する
  Kubernetesのsecretだと、キー情報をテキストに書き留める必要がありそうなのと、運用担当がポータル画面から管理できるように、キーコンテナで管理する方針としました。


概要図すると、こんなかんじです。

![](/images/dify-on-azure/system_design.drawio.png)


# 構成の説明

## Kubernetes サービス

[Docker Compose](https://github.com/langgenius/dify/blob/main/docker/docker-compose.yaml)をmanifestに書き換えています。

- env周りはConfigMapに
- シークレットはAzure KeyVaultから取得する

また、FireCrawlやLangFuseなど他の外部サービスを気軽に追加できます。カスタムツールもすぐに追加可能なので、社内情報にアクセスするツールの追加がはかどります

## データベース

AzureのPostgreSQLサービスには以下の二つがあります。Difyでは、チャット履歴などほぼ全てのデータをデータベスに記録するので、拡張性が優れている（と思われる）Azure Cosmos DB for PostgreSQLを採用しました。Azure Cosmos DB for PostgreSQLでDifyが動作するのか、後述のVectorDBとして動作するのかという検証の意味もあります。

- Azure Database for PostgreSQL
- Azure Cosmos DB for PostgreSQL


## VectorDB

Difyが対応しているVectorDBのうち、Azureのサービスで使えそうなのはpgvectorだったので、Azure Cosmos DB for PostgreSQLを採用しました。。Azure Cosmos DB for PostgreSQLでDifyが動作するのか、という検証の意味合いが強いです。

（DifyのVectorDBが直接CosmosDBかAI Searchに対応してくれればなぁと言う思いもあったりします。こそこそ書いたりしてますが、、、いつになったら動くことらや）

## secret情報

[Azureの公式ドキュメント](https://learn.microsoft.com/ja-jp/azure/aks/csi-secrets-store-driver)にもありますが、KubernetesのSecrets Store CSI Driverを用いキー情報を管理・運用することが出来ます。

ついでにといってはなんですが、[SSL証明書もKeyValut](https://learn.microsoft.com/ja-jp/azure/application-gateway/key-vault-certs)で管理しています。

管理ポータルからシークレット情報が管理できるの運用コスト（と言うか、運用者の教育にかかるコスト）を削減し、誤操作による停止のリスクを減らすことができるので、おすすめです。概要図では省略していますが、Kubernetesの管理用APIは外部公開していません。なので、Kubernetesへの操作は踏み台サーバーから実行する必要があり、結構手間（実際の作業と手順書作成）だったのが楽になりました。

## ネットワーク分離

Ingress Controllerには[Application Gateway Ingress Controller](https://github.com/Azure/application-gateway-kubernetes-ingress)を利用し、ApplicationGateway以外は仮想ネットワーク以外からの通信を行わないようにしています。

AzureのPaaS製品には、[プライベートエンドポイント](https://learn.microsoft.com/ja-jp/azure/private-link/private-endpoint-overview)という、PaaSサービスを仮想ネットワークに取り込む仕組みが用意されています。以下のサービスで利用しています。

- Azure Cosmos DB for PostgreSQL
- Redis
- Storage（Blob）
- OpenAI
- KeyVault

## Storage

Azureなので、普通にBlob Storageです。

# はまったポイント

## Azure Cosmos DB for PostgreSQL

作成したデータベースインスタンスが小さすぎたと言う問題もありますが、[PgBouncer](https://learn.microsoft.com/ja-jp/azure/cosmos-db/postgresql/concepts-connection-pool)を利用しないと、pgvector側で接続が枯渇する問題が発生しました。とくに大きなPDFを一気に登録しようとするとエラーが頻発します。

## Azure Cache for Redis

普通にRedisをデプロイするとSSL通信が必須となっています。REDIS_USE_SSL: 'true' と REDIS_PORT: '6380'を忘れずに

あと、「CELERY_BROKER_URL: ${CELERY_BROKER_URL:-redis://:difyai123456@redis:6379/1}」は以下の通り変更して、KeyVaultから取得しています。

- プロトコルが reiss（最後にs）
- パスワードの部分に、ポータルから取得したトークン
- ポートを6380

```
rediss://:<ポータルから取得したトークン：Base64です>@＜デプロイしたホスト名＞.redis.cache.windows.net:6380/1
```

# 良かったポイント

## 

## 


[def]: https://github.com/langgenius/dify