# fargate-cloudformation

AWS ECS/Fargate を中心としたスタックを構築してくれる CloudFormation
WEB と API の二つのコンテナを作成。

## 構成と cloudformation 実行手順

`templates` - cloudformation で読み込むテンプレートファイルを配置

## テンプレートと読み込み順

マネジメントコンソールから、下記ファイルを上から順番に読み込んでいく。
依存関係があるので、順番が前後した場合取り込めない。

### 共通ファイル

`common.yaml` - 環境問わず共通に、作成したいリソースを記載。Role,Policy

### 環境(dev/stage/prd)ごとに作成

環境ごとに下記リソースを作成していく。

① `network.yaml` - 最下層レイヤー。ネットワークレイヤーのリソースを記述する。ネットワークレイヤーは最も依存性の低いリソースであり、外部のリソースを必要としない。VPC, Subnet, Internet Gateway など。

② `security.yaml` - 第二レイヤー。ネットワークリソースのうち、ネットワークレイヤーに依存するセキュリティ関連のリソースを記述する。IAM は特に他のリソースに依存しないが、個々のレイヤーに記述する。ただし、後述する第三レイヤーに依存するポリシーや IAM はその後に作成する。Security Group, IAM など。

③ `application.yaml` - 第三レイヤー。システムが直接利用するアプリケーションを記述する。ECR,ECS,RDS,ELB,S3,CloudFront,Cognito など。

④ `others/*.yaml` - 上記ファイル以外に、依存関係上最後に作成したいリソースは、適宜このディレクトリ内で管理する。

### ERC/ECS のデプロイ実行

上記 ③ で ECS だけ作成中のままになるため、初回のみデプロイが必要。
ECR にイメージをプッシュして、ECS のサービスを更新する。
※1 時間以内くらいにやらないと ③ の作成がタイムアウトする。
以降は github action で CD を設定しておき、ブランチの更新をトリガーにそちらで ECR/ECS は更新されていく。

#### WEB

作成するフロントエンドの WEB リソースのデプロイを行う。
詳細は割愛。

#### API

1. RDS にデータベース、ユーザ生成

- KDL の VPN を接続必要！
- 以下のコマンドで mysql サーバにアクセス

```
mysql -h <mysql-host> -u admin -p
// stageの場合
mysql -h <xxx.xxx.ap-northeast-1.rds.amazonaws.com> -u admin -p

※adminユーザのパスワードはCloudformationのapplication statckから設定する
```

- データベースとユーザの生成

```
// database生成
CREATE DATABASE <database-name>;

// ユーザの生成
CREATE USER '<apiuser-name>'@'<mysql-host>' IDENTIFIED BY '<apiuser-password>';

// ユーザに上記のデータベースだけ接続できるように権限付与
GRANT ALL PRIVILEGES ON `<database-name>`.* TO '<apiuser-name>'@'<mysql-host>';

FLUSH PRIVILEGES;
```

2. DB マイグレーション
   作成 API の方針に準拠し、マイグレーションを実行する。

3. API Server デプロイ  
   作成 API の初回デプロイを行う。
   詳細は割愛。

## 実行中のコンテナにログインできるように設定

1．ローカルにプラグインインストール

- AWS CLI v2
- Session Manager プラグイン  
  https://dev.classmethod.jp/articles/ecs-exec/  
  URL アクセス ➝ 前提条件 ➝ クライアント側

２．下記のコマンドでコンテナにログインする

```
aws ecs execute-command \
    --cluster <clusterName> \
    --task <taskId> \
    --container <containerName> \
    --interactive \
    --command "ps aux"

command の結果とともに下記の文章が出たら成功！
`The Session Manager plugin was installed successfully. Use the AWS CLI to start a session.`
```
