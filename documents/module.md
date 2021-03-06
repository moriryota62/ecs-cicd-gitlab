- [モジュールの説明](#モジュールの説明)
  - [環境構築モジュールの説明](#環境構築モジュールの説明)
    - [ネットワーク](#ネットワーク)
    - [GitLabサーバ](#gitlabサーバ)
    - [GitLab Runner](#gitlab-runner)
    - [ECSクラスタ](#ecsクラスタ)
  - [サービス構築モジュールの説明](#サービス構築モジュールの説明)
    - [事前準備](#事前準備)
    - [サービスデプロイ](#サービスデプロイ)

# モジュールの説明

大きく`環境構築モジュール群`と`サービス構築モジュール群`に別れます。`環境構築モジュール群`はECSサービスとGitLab CICDを行うための環境を準備するモジュールです。プロジェクトで一度だけ実施するモジュールです。`サービス構築モジュール群`はECSサービスをデプロイする前準備とECSサービスのデプロイおよびCICDパイプラインを構築するモジュールです。サービスごとに実施するモジュールです。

各モジュールは`main`と`module`のディレクトリに別れて構成されます。`main`は各モジュールのパラメータを指定する`{モジュール名}.tf`という名前のtfファイルを格納しています。基本的に利用者はこのmain配下のtfファイル内にあるlocalsの値のみ修正し実行します。`module`は各モジュールが実行するサブモジュール群です。基本的に利用者はmodule配下を気にする必要はありません。（追加の設定など細かなカスタマイズが必要な方や実装が気になる方は見てください。）

また、各モジュールを実行した後に作成されるtfstateはとくにリモートへ保存する設定をしておらず、実行時のカレントディレクトリに保存されます。`.gitignore`を記述しているため、リポジトリにはアップされません。必要に応じてリモートへの保存やロックの仕組みを実装してください。

## 環境構築モジュールの説明

基本となる以下環境をセットアップするterraformモジュールを用意しています。

- ネットワーク
- GitLabサーバ
- GitLab Runner
- ECSクラスタ

### ネットワーク

`ネットワーク`はVPCとパブリックサブネットおよびプライベートサブネットを構築するterraformモジュールです。インターネットゲートウェイやNATゲートウェイ、ECRとS3へのエンドポイントも構築します。このモジュールで作成した`VPCのID`や`サブネットのID`は他のモジュールでも使用します。このモジュールはVPCがない場合などに実行ください。すでにVPCやサブネットがある場合はそれらのIDを他モジュールで使用してください。

### GitLabサーバ

`GitLabサーバ`はセルフホストのGitLabサーバを構築するモジュールです。任意のEC2タイプで構築できます。AMIは指定可能ですが、デフォルトではGitLabより公開されている最新のGitLab CE AMIを使用します。使用にはあらかじめAMIをサブスクライブする必要があります。GitLabサーバにはIPアドレス固定化のためEIPを付与します。また、以下の追加機能を任意で設定できます。追加機能はデフォルトでは`無効`にしています。このモジュールを実行せす、インターネットのSaaS版GitLabを使用しても良いです。その場合、SaaS版GitLabでグループ作成やRunnerトークンの確認を行ってください。

|機能|説明|
|-|-|
|自動起動/停止スケジュール|GitLabサーバは自動で起動/停止するスケジュールを設定します。有効にした場合、デフォルトでは平日の日本時間09-19時の間に起動するように設定します。スケジュールは任意の値に変更可能です。|
|自動バックアップ|GitLabサーバのEBSボリュームのスナップショットを取得します。有効にした場合、デフォルトでは日本時間の0時に1世代のスナップショットを取得します。取得時間、世代数は任意の値に変更可能です。|

### GitLab Runner

`GitLab Runner`はGitLab CICDによるパイプライン処理を実行するGitLab Runnerサーバを構築するモジュールです。AMIは最新のAmazon Linuxを使用します。接続するGitLabサーバと認証用のトークンを設定し、Runnerのセットアップを行います。このセットアップはUserdataにより自動で行います。また、GitLab RunnerサーバにはCICD処理のためS3とECRへの書き込みを許可するIAMロールを割り当てます。また、以下の追加機能を任意で設定できます。追加機能はデフォルトでは`無効`にしています。

|機能|説明|
|-|-|
|自動起動/停止スケジュール|GitLabサーバは自動で起動/停止するスケジュールを設定します。有効にした場合、デフォルトでは平日の日本時間09-19時の間に起動するように設定します。スケジュールは任意の値に変更可能です。|

### ECSクラスタ

`ECSクラスタ`はECSのサービスをまとめるクラスタを構築するモジュールです。また、ECSタスクやCodePipeline、CodeDeployに必要となる共通のIAMポリシーおよびロールを作成します。

## サービス構築モジュールの説明

ECSサービスとサービスのCICDをセットアップするモジュールを用意しています。大きく以下2つあります。

- 事前準備
- サービスデプロイ

### 事前準備

`事前準備`はサービスをデプロイする前のソース置き場を構築するモジュールです。コンテナイメージを格納するECRレポジトリとデプロイ設定を格納するS3バケットを構築します。また、サービスに付与するセキュリティグループも作成します。ECRレポジトリは一日以上経過しているタグのついていないイメージを削除するライフサイクルポリシーも設定します。

### サービスデプロイ

`サービスデプロイ`はサービスのデプロイおよびCICDの設定を行うモジュールです。ECSサービスに紐づくALBもデプロイします。CodePipelineとCodeDeployによるECSサービスのBlue/Greenデプロイを設定します。このモジュールを実行する前にソースモジュールで作成したソース置き場にコンテナイメージおよびデプロイ設定を格納してください。CICDはソース置き場の情報が更新される度に自動で実行されます。
