# 各種バージョン
terraform 0.13.2 以上
aws providor 3.5.0以上

# レポジトリの説明

GitLab + ECS CICDパイプラインを構築するTerraformモジュール群とそのセットアップ方法を格納したレポジトリです。GitLabに作成するレポジトリのサンプルも格納しています。本レポジトリで作成するCICDの全体像は以下の通りです。

AWS構成図

CICDフロー図

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

`ネットワーク`はVPCとパブリックサブネットおよびプライベートサブネットを構築するterraformモジュールです。インターネットゲートウェイやNATゲートウェイも構築します。このモジュールで作成した`VPCのID`や`サブネットのID`は他のモジュールでも使用します。このモジュールはVPCがない場合などに実行ください。すでにVPCやサブネットがある場合はそれならのIDを他モジュールで使用してください。

`GitLabサーバ`はセルフホストのGitLabサーバを構築するモジュールです。任意のEC2タイプで構築できます。AMIは指定可能ですが、デフォルトではGitLabより公開されている最新のGitLab CE AMIを使用します。使用にはあらかじめAMIをサブスクライブする必要があります。GitLabサーバにはIPアドレス固定化のためEIPを付与します。また、以下の追加機能を任意で設定できます。追加機能はデフォルトでは`無効`にしています。このモジュールを実行せす、インターネットのSaaS版GitLabを使用しても良いです。その場合、SaaS版GitLabでグループ作成やRunnerトークンの確認を行ってください。

|機能|説明|
|-|-|
|自動起動/停止スケジュール|GitLabサーバは自動で起動/停止するスケジュールを設定します。有効にした場合、デフォルトでは平日の日本時間09-19時の間に起動するように設定します。スケジュールは任意の値に変更可能です。|
|自動バックアップ|GitLabサーバのEBSボリュームのスナップショットを取得します。有効にした場合、デフォルトでは日本時間の0時に1世代のスナップショットを取得します。取得時間、世代数は任意の値に変更可能です。|

`GitLab Runner`はGitLab CICDによるパイプライン処理を実行するGitLab Runnerサーバを構築するモジュールです。AMIは最新のAmazon Linuxを使用します。接続するGitLabサーバと認証用のトークンを設定し、Runnerのセットアップを行います。このセットアップはUserdataにより自動で行います。また、GitLab RunnerサーバにはCICD処理のためS3とECRへの書き込みを許可するIAMロールを割り当てます。

`ECSクラスタ`はECSのサービスをまとめるクラスタを構築するモジュールです。また、ECSタスクやCodePipeline、CodeDeployに必要となる共通のIAMポリシーおよびロールを作成します。

## サービス構築モジュールの説明

ECSサービスとサービスのCICDをセットアップするモジュールを用意しています。大きく以下2つあります。

- ソース
- サービスデプロイ

`ソース`はサービスをデプロイする前のソース置き場を構築するモジュールです。コンテナイメージを格納するECRレポジトリとデプロイ設定を格納するS3バケットを構築します。

`サービスデプロイ`はサービスのデプロイおよびCICDの設定を行うモジュールです。ECSサービスに紐づくALBもデプロイします。CodePipelineとCodeDeployによるECSサービスのBlue/Greenデプロイを設定します。このモジュールを実行する前にソースモジュールで作成したソース置き場にコンテナイメージおよびデプロイ設定を格納してください。CICDはソース置き場の情報が更新される度に自動で実行されます。

# 使い方

以下の順番で各モジュールを実行します。環境構築の`ネットワーク`および`GitLabサーバ`は自身の環境に合わせて実行要否を判断してください。サービス構築の`GitLab CICDによるソース配置`はTerraformモジュールの実行ではなく、GitLabのレポジトリに対する作業になります。サービス構築はサービスごとに実行してください。

- 環境構築
  - ネットワーク（任意）
  - GitLabサーバ（任意）
  - GitLab Runner
  - ECSクラスタ
- サービス構築
  - ソース
  - GitLab CICDによるソース配置
  - サービスデプロイ

まずは本レポジトリを任意の場所でクローンしてください。なお、以降の手順では任意のディレクトリのパスを`$CLONEDIR`環境変数として進めます。

``` sh
export CLONEDIR=`pwd`
git clone https://github.com/moriryota62/ecs-cicd.git
```

## 環境構築

環境構築はプロジェクトで一度だけ行います。環境の分け方によっては複数実施するかもしれません。`main`ディレクトリをコピーして`環境名`ディレクトリなどの作成がオススメです。以下の手順では`cicd-dev`という環境名を想定して記載します。

``` sh
cd $CLONEDIR/ecs-cicd/
export PJNAME=cicd-dev
cp -r main $PJNAME
```

また、すべてのモジュールで共通して設定する`pj`、`region`、`owner`の値はsedで置換しておくと後の手順が楽です。regionはデフォルトでは'ap-northeast-1'を指定しています。変える必要がなければsedする必要ありません。

**macの場合**

``` sh
cd $PJNAME
find ./ -type f -exec grep -l 'ap-northeast-1' {} \; | xargs sed -i "" -e 's:ap-northeast-1:us-east-2:g'
find ./ -type f -exec grep -l 'PJ-NAME' {} \; | xargs sed -i "" -e 's:PJ-NAME:'$PJNAME':g'
find ./ -type f -exec grep -l 'OWNER' {} \; | xargs sed -i "" -e 's:OWNER:nobody:g'
```

### ネットワーク

すでにVPCやサブネットがある場合、ネットワークのモジュールは実行しなくても良いです。その場合はVPCとサブネットのIDを確認しておいてください。ネットワークモジュールでVPCやサブネットを作成する場合は以下の手順で作成します。

ネットワークモジュールのディレクトリへ移動します。

``` sh
cd $CLONEDIR/ecs-cicd/$PJNAME/environment/network
```

`network.tf`を編集します。`region`と`locals`配下のパラメータを修正します。VPCやサブネットのCIDRは自身の環境にあわせて任意のアドレス帯に修正してください。

修正したら以下コマンドでモジュールを作成します。

``` sh
terraform init
terraform apply
> yes
```

実行後に出力される`vpc_id`や`public_subent_ids`などは後続のモジュールで使用します。内容を控えておくと良いでしょう。

``` sh
export VPCID=<vpc_id>
export PUBLICSUBNET1=<public_subent_ids 1>
export PUBLICSUBNET2=<public_subent_ids 2>
export PRIVATESUBNET1=<private_subent_ids 1>
export PRIVATESUBNET2=<private_subent_ids 2>
```

### GitLabサーバ

インターネットのSaaS版GitLabを使用する場合、GitLabサーバのモジュールは実行しなくても良いです。SaaS版でもグループの作成やRunnerトークンの確認は実施してください。

GitLabサーバモジュールのディレクトリへ移動します。

``` sh
cd $CLONEDIR/ecs-cicd/$PJNAME/environment/self-host-gitlab
```

`self-host-gitlab.tf`を編集します。`region`と`locals`配下のパラメータを修正します。とくにvpc_idとsubnet_id（パブリックサブネットのID）、SGのインバウンドCIDRは自身の環境にあわせて変更してください。また、自動スケジュールや自動スナップショットを有効にする場合、対応する機能を`true`に設定してください。

**macの場合**

``` sh
sed -i "" -e 's:VPC-ID:'$VPCID':g' self-host-gitlab.tf
sed -i "" -e 's:PUBLIC-SUBNET-1:'$PUBLICSUBNET1':g' self-host-gitlab.tf
sed -i "" -e 's:192.0.2.10:YOURCIDR:g' self-host-gitlab.tf
```

修正したら以下コマンドでモジュールを作成します。

``` sh
terraform init
terraform apply
> yes
```

表示される`public_dns`、`public_ip`、`runner_sg_id`はどこかに控えておいてください。

terraform実行後、以下の通りGitLabサーバにアクセスしてGitLabサーバの準備をしてください。

- GitLabサーバを構築したら許可した端末からGUIに接続します。ブラウザにEC2インスタンスのパブリックIPまたはパブリックDNSを入力してください

- 初回アクセスの場合、rootのパスワード変更を求められるため、パスワードを設定してください

- rootのパスワードを入力するとログイン画面が表示されます。先ほど設定したパスワードでrootにログインします

- 上部メニューバーの[Admin Area（スパナマーク）]を選択します

- 左メニューから[Settings]-[General]-[Visibility and access controls]を開き`Custom Git clone URL for HTTP(S)`にEC2インスタンスのパブリックDNS名を`http://`をつけて入力し[Save changes]します。（デフォルトではAWSの外部で名前解決できない名前になっているためです。名前解決できる名前ならプラベートDNS名以外の任意のドメイン名でも構いません。）

- 左メニューから[Overview]-[Users]を開き`New user`を選択します。任意のユーザを作成してください。（パスワードはユーザ作成後、ユーザの一覧画面でeditすると設定できる。）

- 左メニューから[Overview]-[Groups]を開き`New group`を選択します。任意の名前のグループを作成してください。また、上記作成したユーザをグループのownerに設定してください

- 一度rootからログアウトし、上記ユーザでログインしなおしてください

- 上部メニューバーの[Groups]-[Your groups]を表示し、先ほど作成したグループを選択します

- グループの画面で左メニューから[Settings]-[CICD]-[Runners]を開きます。`Set up a group Runner manually`のURLとトークンを確認します。たとえば以下のような値となっているはずです

1. http://ec2-3-138-55-5.us-east-2.compute.amazonaws.com/
2. 972hz6YiJTWUcN4ECUNk

### GitLab Runner

GitLab Runnerサーバモジュールのディレクトリへ移動します。

``` sh
cd $CLONEDIR/ecs-cicd/$PJNAME/environment/gitlab-runner
```

`gitlab-runner.tf`を編集します。`region`と`locals`配下のパラメータを修正します。とくにvpc_idとsubnet_id（パブリックサブネットのID）は自身の環境に合わせて修正してください。また、ec2_gitlab_urlとec2_registration_tokenも`GitLabサーバ`モジュールで確認した値に必ず修正してください。SaaS版GitLabの場合、urlは`https://gitlab.com`になります。`ec2_sg_id`はセフルホストの場合、GitLabサーバモジュールのoutputで表示された`runner_sg_id`を設定してください。SaaS版GitLabの場合は`空文字`で設定してください。

``` sh
sed -i "" -e 's:VPC-ID:'$VPCID':g' gitlab-runner.tf
sed -i "" -e 's:PUBLIC-SUBNET-1:'$PUBLICSUBNET1':g' gitlab-runner.tf
sed -i "" -e 's:GITLAB-URL:<先ほどGitLabで確認したURL>:g' gitlab-runner.tf # http:の`:`の前にエスケープを入れてください。例 http\:
sed -i "" -e 's:REGIST-TOKEN:<先ほどGitLabで確認したregistraton_token>:g' gitlab-runner.tf
sed -i "" -e 's:RUNNER-SG-ID:<GitLab RunnerのSG>:g' gitlab-runner.tf # セフルホストの場合はSG ID ,SaaSの場合は空文字
```

修正したら以下コマンドでモジュールを作成します。

``` sh
terraform init
terraform apply
> yes
```

上記実行が完了したらGitLab側にRunnerが認識されているか確認します。

- グループに所存したユーザでGitLabにログインしてください

- グループの画面で左メニューから[Settings]-[CICD]-[Runners]を開きます。`Available Runners`に作成したRunnerが表示されていれば登録完了です

### ECSクラスタ

ECSクラスタモジュールのディレクトリへ移動します。

``` sh
cd $CLONEDIR/ecs-cicd/$PJNAME/environment/ecs-cluster
```

`ecs-cluster.tf`を編集します。`region`と`locals`配下のパラメータを修正します。

修正したら以下コマンドでモジュールを作成します。

``` sh
terraform init
terraform apply
> yes
```

## サービス構築

サービスの構築はサービスごとに行います。terraformのコードもサービスごとに作成するため、あらかじめ用意された`service-template`ディレクトリをコピーし、`サービス名`ディレクトリなどの作成がオススメです。以下の手順では`test-app`というサービス名を想定して記載します。

``` sh
cd $CLONEDIR/ecs-cicd/$PJNAME/
export APPNAME=test-app
cp -r service-template $APPNAME
```

また、すべてのモジュールで共通して設定する`pj`、`app`、`vpc_id`の値はsedで置換しておくと後の手順が楽です。なお、ここで設定するpjの値は環境構築モジュールで設定したpjと同じ値にしてください。

**macの場合**

``` sh
cd $APPNAME
find ./ -type f -exec grep -l 'APP-NAME' {} \; | xargs sed -i "" -e 's:APP-NAME:'$APPNAME':g'
find ./ -type f -exec grep -l 'VPC-ID' {} \; | xargs sed -i "" -e 's:VPC-ID:'$VPCID':g'
```

### 事前準備

事前準備モジュールのディレクトリへ移動します。

``` sh
cd $CLONEDIR/ecs-cicd/$PJNAME/$APPNAME/preparation
```

`preparation.tf`を編集します。`region`と`locals`配下のパラメータを修正します。

修正したら以下コマンドでモジュールを作成します。

``` sh
terraform init
terraform apply
> yes
```

outputに出力されるsg_idはService用に作成したセキュリティーグループのIDです。この後の手順で使用します。

### GitLab CICDによるソース配置

これはterraformを実行する手順ではありません。GitLabで実施する手順になります。現在AWS側にはアプリケーションのコンテナイメージとECSのデプロイ設定を配置する場所が用意できています。例で作成した`cicd-dev`プロジェクトの`test-app`アプリケーションの場合、それぞれ以下の通りです。

|対象データ|配置サービス|配置場所|
|-|-|-|
|アプリケーションのコンテナイメージ|ECR|`cicd-dev-test-app`レポジトリ|
|ECSのデプロイ設定|S3|`cicd-dev-test-app`バケット|

上記配置場所に対象データを格納しないとこの後に実行する`サービスデプロイ`モジュールが上手く動きません。GitLab CICDを動かすための環境も準備できているため、GitLab CICDによりGitLabのレポジトリにプッシュすれば自動で配置場所に対象データを格納できるようにしましょう。サンプルとなるレポジトリを`sample-repos`ディレクトリ配下に用意しています。これらのサンプルを参考に以下の様にGitLab側の設定を行います。なお、サンプルディレクトリ配下のファイルを使う場合、以下のように置換を行ってください。

``` sh
cd $CLONEDIR/ecs-cicd/
cp -r sample-repos $APPNAME
cd $APPNAME
find ./ -type f -exec grep -l 'REGION' {} \; | xargs sed -i "" -e 's:REGION:<自身が使用しているリージョン>:g'
find ./ -type f -exec grep -l 'AWS-ID' {} \; | xargs sed -i "" -e 's:AWS-ID:<自身が使用しているAWSアカウントのID>:g'
find ./ -type f -exec grep -l 'PJ-NAME' {} \; | xargs sed -i "" -e 's:PJ-NAME:'$PJNAME':g'
find ./ -type f -exec grep -l 'APP-NAME' {} \; | xargs sed -i "" -e 's:APP-NAME:'$APPNAME':g'
find ./ -type f -exec grep -l 'SG-ID' {} \; | xargs sed -i "" -e 's:SG-ID:<Service用に作成したSGのID>:g'
find ./ -type f -exec grep -l 'PRIVATE-SUBNET-1' {} \; | xargs sed -i "" -e 's:PRIVATE-SUBNET-1:'$PRIVATESUBNET1':g'
find ./ -type f -exec grep -l 'PRIVATE-SUBNET-2' {} \; | xargs sed -i "" -e 's:PRIVATE-SUBNET-2:'$PRIVATESUBNET2':g'
```

- グループに所存したユーザでGitLabにログインしてください

- 上部メニューバーの[Groups]-[Your groups]を表示し、先ほど作成したグループを選択します

- グループの画面で`new project`で新しくプロジェクト（レポジトリ）を作成します。サンプルの例だと`app`と`ecs`という名前の2つのレポジトリを作成します。このとき、個別ユーザのプロジェクトではなく、グループ配下のプロジェクトとして作成してください。そうしないとRunnerが使えません。

- `app`レポジトリにはアプリケーションのソースとDockerfileを配置するため以下のコマンドを実行します。レポジトリのルートに`.gitlab-ci.yml`を配置します。サンプルだとファイル名の頭に`.`を入れていないためリネームを行っておいます。この`.gitlab-ci.yml`はGitLab CICDのパイプライン設定になります。このファイルに記述した内容をGitLab Runnerサーバで実行します。`tags`で指定した値はどのGitLab Runnerを使用するか判別するためのものです。プロジェクト用に作成したGitLab Runnerを指定するためプロジェクト名を指定しています。

  ``` sh
  cd $CLONEDIR
  git clone <appレポジトリのクローンURL>
  cd app
  cp -r $CLONEDIR/ecs-cicd/$APPNAME/app/* ./
  mv gitlab-ci.yml ./.gitlab-ci.yml
  git add .
  git commit -m "init"
  git push
  ```

- `ecs`レポジトリにはCodeDeployによるECS Serviceデプロイをするための設定ファイルを配置するため以下のコマンドを実行します。`app`レポジトリ同様、レポジトリのルートに`.gitlab-ci.yml`を配置し、対象runnerのタグをプロジェクト名にします。

  ``` sh
  cd $CLONEDIR
  git clone <ecsレポジトリのクローンURL>
  cd ecs
  cp -r $CLONEDIR/ecs-cicd/$APPNAME/ecs/* ./
  mv gitlab-ci.yml ./.gitlab-ci.yml
  git add .
  git commit -m "init"
  git push
  ```

- `.gitlab-ci.yml`があるレポジトリをmasterにプッシュするとGitLab CICDによりパイプラインが実行されます。`app`レポジトリでは「docker build」を行い指定したECRへプッシュします。`ecs`レポジトリでは設定ファイル群をzipファイルに圧縮し指定したS3バケットへcpします。パイプラインの実行状態は各レポジトリの左メニュー[CICD]-[パイプライン]で状況を確認できます

- 上手く実行できればECRおよびS3にデータが格納でてきるはずなので確認してください

### サービスデプロイ

サービスデプロイモジュールのディレクトリへ移動します。

``` sh
cd $CLONEDIR/ecs-cicd/$PJNAME/$APPNAME/service-deploy
```

`source.tf`を編集します。`region`と`locals`配下のパラメータを修正します。とくにvpc_idとsubnet_idは自身の環境に合わせて修正してください。

``` sh
sed -i "" -e 's:PRIVATE-SUBNET-1:'$PRIVATESUBNET1':g' service-deploy.tf
sed -i "" -e 's:PRIVATE-SUBNET-2:'$PRIVATESUBNET2':g' service-deploy.tf
sed -i "" -e 's:PUBLIC-SUBNET-1:'$PUBLICSUBNET1':g' service-deploy.tf
sed -i "" -e 's:PUBLIC-SUBNET-1:'$PUBLICSUBNET2':g' service-deploy.tf
sed -i "" -e 's:SERVICESGID:<ServiceのSG ID>:g' service-deploy.tf
```

修正したら以下コマンドでモジュールを作成します。

``` sh
terraform init
terraform apply
> yes
```

実行後に出力される`dns_name`はLBのDNS名です。コピーしてWEBブラウザでアクセスします。すべて上手く行けば以下のようなメッセージの画面が表示されます。なお、デプロイはterraform完了からさらに数分の時間を要します。デプロイ失敗なのか待ちなのか確認するには、マネジメントコンソールでcodepipelineの画面を開き現在の状況を追ってみるとよいでしょう。Deployが進行中であればまだしばらく待ったください。

## 環境削除

構築したときの逆の以下モジュール順に`terraform destroy`を実行してください。

1. サービスデプロイ
2. ソース
3. ECSクラスタ
4. GitLab Runner
5. GitLabサーバ
6. ネットワーク

