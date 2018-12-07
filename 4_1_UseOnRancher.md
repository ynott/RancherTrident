# 4. コンテナーでの利用(Rancherでの利用方法)

## 1. Dynamic Provisioing でのコンテナーへのストレージの割り当て
### 1. Rancher のメニュー構成について

まず、Rancher でDynamic Provisioingを使う為の画面へたどり着くために Rancher UI画面のメニュー構成について確認しておきます。
クラスターのメニュー構成は以下のようになっており、ストレージ関連はクラスターに紐付いていることが分かります。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_RancherUI-menu.png)

### 2. クラスターのストレージ

次にクラスターのストレージクラスで、第3章 StorageClassの定義例 で定義した ontap-gold があるのを確認します。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_Cluster-StorageClass.png)

#### デフォルトストレージクラスへの変更

しかし上記で確認したストレージクラスはデフォルトになっていません。これはUI上から変更可能です。
デフォルトになっていないストレージクラスの右側の縦三つをクリックして「デフォルトに設定」をクリックします。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_Cluster-StorageClass-Default.png)

### 3. プロジェクトのメニュー構成について

Rancherのメニュー構成で確認したクラスターのメニューからは、コンテナーをデプロイすることはできませんでした。コンテナーをデプロイするには、Rancherのプロジェクトメニューから行う必要があります。

まず、Rancherのプロジェクトのメニュー構成について確認しておきます。
以下のようになっており、今回利用するのは網掛けのワークロードとボリュームです。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_RancherUI-Project-menu.png)

プロジェクトメニューにいくには、グローバルメニューからデフォルトで用意されている「Default」プロジェクトに切り換えます。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_Project.png)

### 4. コンテナーをデプロイする

コンテナーをデプロイして、ストレージを割り当てます。ワークロードからワークロードを選択し、「デプロイ」ボタンを押します。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_Workload-Workload-deploy.png)

次の画面でワークロードデプロイに名前とイメージを指定します。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_Workload-Deploy.png)

### 5. ボリュームの割り当て

次にボリュームを割り当てます。「ボリューム」をクリックします。「ボリュームを追加」をクリックして、「新しい永続ボリューム(要求)を作成」をクリックします。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_dynamic-pv-create-03.png)


名前とストレージクラスを指定して、容量を入力します。ソースは、「新しい永続ボリュームの作成にストレージクラスを使用」を選択します。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_dynamic-pv-create-04.png)

「定義」ボタンをクリックします。

### 5. ボリュームのマウント先を指定

ボリュームは定義はできたので、それをコンテナーのどこに割り当てるか設定します。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_dynamic-pv-create-04.png)

マウントポイントを指定します。これがコンテナー側から見られるパスになります。さらにその先にパスがある場合は、Sub Path in Volume を指定します。

ワークロードをデプロイの一番下にある「起動」を押してコンテナーを起動します。

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_RunContainer.png)

### 6. マウントを確認

起動したコンテナーに接続して、マウント状況を確認する

![](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/4_Execute-shell.png)

