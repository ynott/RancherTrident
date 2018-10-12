<<<<<<< HEAD
## Trident のバックエンドストレージに設定したONTAPでの使い方

ストレージ機能としてストレージ全体を遠隔地にレプリケーションする機能があります。(SVM-DR)

ここでは Heptio ark をつかい、kubernetesの構成情報をバックアップ/リストアし、
実データをSVM-DRを用いデータ転送後、別Kubernetes環境でリストアする方法について説明します。

=======
## Trident のバックエンドストレージに設定したONTAPでの使い方

ストレージ機能としてストレージ全体を遠隔地にレプリケーションする機能があります。(SVM-DR)

ここでは Heptio ark を使用し、kubernetesの構成情報をバックアップ/リストアし、
実データをSVM-DRを用いデータ転送後、別Kubernetes環境でリストアする方法について説明します。

なお、ここでは２つのkubernetesクラスタ、プライマリ、DRが稼働している前提です。

>>>>>>> preview-01
### Heptio Ark

Heptio 社が提供しているkubernetesのデファクトバックアップ・リストアツールです。

- https://github.com/heptio/ark

<<<<<<< HEAD
Kubernetsクラスタの構成情報をバックアップし、リストアすることができます。
この機能を使うことで以下のようなことを実現できます。

- Kubernetesクラスタのバックアップ・リストア、ポータビリティを提供
- Kubernetesクラスタの災害対策
- 本番Kubernetesクラスタから構成情報を取得して、開発クラスタとして構成

現在のバージョン(v0.9.0)ではボリュームのレプリケーションは提供されておりません。

- https://heptio.github.io/ark/v0.9.0/index.html

この章では、Heptio ArkでKubernetesクラスタの構成情報を取得し、
ストレージ機能である ``SVM-DR``を使用して、データとストレージ構成情報を取得し、別クラスタでの復元について実践します。

### NetApp SVM-DR

NetAppのONTAP機能として、SVM-DRで

### Heptio Ark + Trident

これからご紹介する全体概要は以下の概念図となっています。

#### 設定例

#### 実行例

#### 検討事項
=======
heptio ark を使うことでクラスタ全体やLabelで指定したkubernetesオブジェクトをバックアップすることができます。

取得したバックアップを使用することで以下のようなことが実現可能です。

- kubernetes クラスタのバックアップ、DR
- 本番環境のクラスタから開発環境へ構成情報をリストア
- クラスタの引っ越し

![NetAppTrident](images/4_5_DRwithHeptio.png)

稼働を確認した時点ではボリュームのレプリケーションはサポートされていないため、別の手段でバックアップ・リストアをする必要があります。
Heptio ark 0.9以降ではresticインテグレーションがサポートされ、アプリケーションデータをオブジェクトストレージへ取得できるようになりました。
ただし、サポートされているのは local persistent volume となっています。

- local persistent volume: https://kubernetes.io/docs/concepts/storage/volumes/#local

### アプリケーションデータのレプリケーション

アプリケーションデータのバックアップ・DRはストレージレイヤーのレプリケーション機能で実現します。

NetAppではデータレプリケーションの方法としては以下の３つの方法があります。

1. SnapMirror: ボリューム単位のデータレプリケーション、ブロック差分転送が可能
1. SVMDR: 仮想ストレージ単位のデータレプリケーション、ブロック差分転送が可能、かつストレージ情報(アカウントやデータ通信用のIPなど)を維持した状態で転送
1. CloudSync: SaaSのデータ同期ツール、ファイル単位の差分転送が可能

今回はSVMDRを使用して実現しました。

#### SVMDRを選択した理由

Heptio Ark を使用して取得するバックアップにバックエンドストレージのIPや構成情報が含まれている状態でバックアップされます。
そこから復元を行うと、ストレージレイヤの構成情報が変更になると正常に稼働しなくなるため、仮想ストレージ単位ですべてを転送する仕組みを提供できるSVMDRを選択しました。

### 実際の構成

#### Heptio Arkのインストール

インストールについては開発が活発なこともあり、公式のドキュメントを参考とすることをおすすめします。

- https://heptio.github.io/ark/v0.9.0/quickstart

サポートされるバックアップ先のストレージは以下のページに記載があります。

- https://github.com/heptio/ark/blob/master/docs/support-matrix.md

Heptio ark インストール時にどこのオブジェクトストレージにバックアップを取得するかを定義します。

この操作は両方のクラスタで実施する必要があります。

#### Heptio Arkバックアップスケジュール設定

ark ではスケジュール設定をすることで定期的に構成情報のバックアップを取得可能です。

スケジュールは ``--schedule=`` で設定でき、書式はcrontabのものが使用可能です。

``` console
ark create schedule k8sbackup --schedule="*/5 * * * *" --exclude-namespaces kube-system
```

namespace の指定や、ラベルセレクターを指定することも可能です。

#### NetApp ストレージの設定

ここではある程度ONTAPを知っている前提で記載しています。
SVMDRを構成するため、それぞれのクラスタで使用するストレージ同士でレプリケーションの設定を行います。
実施する作業としては以下の流れです。２つのクラスタを行き来しながら操作します。

- Intercluster LIF 作成
- cluster peer 作成
- SVM Peer 作成
- SVMDR のリレーション作成
- SVMDR のデータシンク

以上でストレージレイヤーの事前設定が完了です。

### クラスタフェイルオーバーを実施

ここまでの状態で以下2点の定期的なバックアップが取得できている状態です。

- heptio ark によるkubernetesの構成情報
- ストレージレイヤーにおけるアプリケーションデータの定期的なバックアップ・遠隔地への転送

ここからはストレージが停止したことを想定し、どのような操作が必要かを説明します。

DR側のストレージで以下のコマンドを実行し、DRをプライマリとして扱うようにします。

SVMDRのリレーションをブレイク。

``` console
snapmirror break -destination-path [DR用のSVM]:
```

仮想ストレージサーバを起動

``` console
vserver start -vserver [DR用のSVM]
```

ここまででDR側ストレージの起動が完了しました。

続いて、Arkからバックアップデータをリストアしてkubernetsクラスタがプライマリと同じ状態になるかを確認します。

まずはバックアップデータの一覧を確認します。

```console
$ ark backup get

NAME      STATUS      CREATED                         EXPIRES   SELECTOR
test1     Completed   2018-07-06 16:27:30 +0900 JST   29d       <none>
```

リストアを実行します。

```
$ ark restort create restore1 --from-backup test1

Restore request "restore1" submitted successfully.
Run `ark restore describe restore1` for more details.
[root@dr-master ~]# ark get restore
NAME       BACKUP    STATUS       WARNINGS   ERRORS    CREATED                         SELECTOR
restore1   test1     InProgress   0          0         2018-07-06 16:39:40 +0900 JST   <none>
```

実行が完了したらkubernetesクラスタの確認を行います。

``` console
$ kubectl get all --all-namespaces
```

### 考慮点

- プライマリ、DR側で同じネットワーク構成にしておくとDR時の操作が簡易化が可能。
- 切り替えのタイミングの自動化を検討
- 切り戻しの実施方法
>>>>>>> preview-01
