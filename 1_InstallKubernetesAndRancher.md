# 1. Rancher/Kubernetes の構築

## 1. Kubernetes環境の構成説明

Kubernetesクラスターの構成を以下のように構成する。
共有する環境としてDNSをdns1,dns2を設定し、Kubernetesクラスター毎のセグメントと分離する。Kubernetesクラスターには、Masterノードを1つ、Workernノードをnode0,node1を設置する。同じセグメント内にNetApp SVMを設置し、SSDとHDDのアグリゲートを作成し、それぞれaggr1_01、aggr2_01とした。

![Kubernetesサーバー構成](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/KubernetesServers.png)

## 2. Kubernetesのインストール

Kubernetesクラスターのインストールには、kubeadmでインストールを行った。特別な操作はせず通常の方法でセットアップしたため、ここでは操作方法などは省略する。

## 2. Rancherの特徴

Rancher は オープンソースで開発されているKubernetesを管理するアプリケーションです。Enterprise Kubernetes made Easy を合い言葉にKubernetesの機能を企業ユースで利用しやすく管理する機能が備わっています。  

管理するユーザーインターフェースは、Webブラウザーを使ったグラフィカルなインターフェースとターミナルを使った専用コマンドを通じて操作する方法との2種類があります。  
通常、KubernetesではManifestと呼ばれるコンテナー操作用ファイルを作成し、それをkubectlコマンドにより反映させてコンテナーを起動したり停止したりします。しかし、Rancherではそのようなコマンドラインで操作することなく、ブラウザーの操作でコンテナーを操作します。他にもManifestファイルを使って指定する必要のある様々なオプションをWebUIから設定できるようになっています。このようなコンソール操作をしなくてもよい利便性がRancherの特徴の一つになっています。

もう一つの特徴が複数のKubernetesクラスターを管理できるというマルチクラウドの機能です。このマルチクラウドには、オンプレミスのKubernetesクラスターも含まれ、クラウドやオンプレミス、その他様々なディストリビューションの複数のKubernetesを一元的に管理することができます。  
現在では複数のKubernetesを管理できるというツールは非常に数多くありますが、開発当初から設置場所を問わず、マルチクラウドでマルチディストリビューション対応を打ち出しているツールは多くありません。
 
また、Kubernetesクラスターを企業向けのニーズにマッチさせるということもRancherの大きなミッションの一つになっています。その一つはユーザー認証やセキュリティーポリシーの設定です。今日の企業においてユーザー管理やセキュリティーポリシーの設定は非常に重要な要素となっています。もし、ユーザーをシステム毎にバラバラで管理しなければならない場合、その手間は非常に大きなものとなるでしょう。  
そういった認証システムと統合し、Active Directory や GitHub、Azure AD、FreeIPA、OpenLDAPといった認証システムとの連携ができます。認証ばかりではなく、ネットワークのポリシー、Kubernetesに則ったRBACの設定などをKubernetesのコマンドを利用することなくWebブラウザーのUIを通して設定する事ができます。

もう一つは、企業利用において重要な稼働監視やロギング、モニタリング、アラート通知機能です。様々なクラウド上で動いているマネージドサービスのKubernetesクラスターは管理についてあまり気にする必要がありません。しかし、オンプレミスのKubernetesクラスターではサーバーが止まったりリソースが足りなくなる事が考えられます。当然それらを無視することはできず、何らかの対応をしなけれなりません。しかし、その状況が通知されなければ、そもそも対応することも出来ません。その為には稼働状況を常時記録し、問題があった場合には複数の手段で担当者へ連絡する必要があります。Rancherにはそれらの機能も備えていて、アラート発報方法、通知方法を設定し、障害対応の為のログをログサーバーへの転送することができます。

最後にアプリケーションの開発者向けの機能についてご紹介しておきます。WebUIの操作でコンテナーを操作できると書きましたが、毎回WebUIで全てのコンテナーのパラメータを指定して操作するのは非常に大変です。そこで、Kubernetesのパッケージマネージャーとして知られているHelmを利用して、それらの操作を軽減することができるようになっています。  

それらは、Rancher Catalogという名前で利用できますが、通常のHelmのカタログとRancher独自のカスタマイズ可能なRancher Chartを利用したカタログの2種類を利用することができます。これは運用する人にとって便利な機能となっているのみならず、開発者にとっても事前に作成しておいた環境を簡単に動かすことができるというメリットをもたらしています。

これらの機能を No Vendor Lock-In つまり **どこのベンターにも依存しない** という一貫したポリシーにより開発されています。Rancher社のどのベンダーからも、どのクラウドにも、ツールにも依存しないという思想は逆にどこにでも置けて、どこにでもつなげて、Kubernetesの中立的な立ち位置で利用できるという製品のポジションを確立しているのです。


## 3. Rancherのインストール
### 1. Rancherインストールに必要な構成

Rancher 2.1.1がサポートしているDockerのバージョンは以下の通りです。
* Ubuntu 16.04 (64-bit)
    * Docker 17.03.2
* Red Hat Enterprise Linux (RHEL)/CentOS 7.5 (64-bit)
    * RHEL Docker 1.13
    * Docker 17.03.2
* RancherOS 1.4 (64-bit)
    * Docker 17.03.2
* Windows Server version 1803 (64-bit)
    * Docker 17.06

基本的にKubernetesがサポートしているDockerのバージョンが対応している
バージョンとなります。

### 2. Dockerのインストール

Dockerのバージョンは、上記のものをインストールしますが、
インストールする方法は、Rancher Labs社のインストールスクリプトを使うのが便利です。
ここでは、Ubuntu 16.04 で動かします。

```
curl https://releases.rancher.com/install-docker/17.03.sh | sh
```

curlコマンドが入っていない場合は、curl コマンドをインストールします
```
sudo apt install curl
```
### 3. Rancherのインストール

Rancher をインストールします。

```
$ sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher
```

## 4. RancherからKubernetesのインポート

### 1. Rancherへのログイン

インストール後にRancherをインストールしたサーバーをブラウザーで表示します。
以下のような画面が表示されます。
(※SSL証明書が自己証明書の場合は、承認を追加します)

![Rancherログイン画面](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/RancherLogin.png)

パスワード設定後、Rancher管理画面が表示されます。

### 2. Kubernetesクラスター追加方法

起動したばかりのRancherにはKubernetesクラスターがありません。Kubernetesクラスターの追加方法にはいくつかの方法がありますが、ここでは事前にKubeadmで構築しておいたKubernetesクラスターをインポートする方法をご紹介します。

Kubeadmを使ったKubernetesクラスターの構築方法については下記公式ドキュメントを参照してください。

Installing kubeadm - Kubernetes  
https://kubernetes.io/docs/setup/independent/install-kubeadm/

次に、作った Kubernetesクラスターを Rancherから認識できるようにインポートします。
Globalから **Add Cluster** ボタンを押します。

![クラスターインポート画面](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/Add-Cluster-Dashboard.png)

クラスター追加画面が出てきますが、右上の **IMPORT** ボタンを押します。

![クラスター追加画面](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/Import-Cluster.png)

次に、Cluster Nameを指定して **Create** ボタンを押します(Memberは自分一人で使う分には追加する必要はありません)。

![クラスター名設定画面](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/Set-ClusterName.png)

以下のページで表示されたコマンドを実行します。
kubectlコマンドは事前にインストールし、kubernetesに接続できるよう設定しておいてください。

![インポートコマンド画面](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/Import-command.png)

```
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user [USER_ACCOUNT]
```

上記の [USER_ACCOUNT] は上記コマンドを実行するユーザーIDを指定します。

上記のコマンドで証明書の問題のエラーが発生する場合は、以下のコマンドを実行して下さい。

```
curl --insecure -sfL https://xxxxxxxxxxxxxx.com/v3/import/XXXXXXXXXXXXXXXXXXXXXXXXX.yaml | kubectl apply -f -
```

KubernetesクラスターがRancherにインポートされると以下のようにGlobalのClusterダッシュボードにインポートされたクラスターが表示されます。

![クラスターインポート完了画面](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/cluster-list.png)