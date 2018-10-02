## Trident のバックエンドストレージに設定したONTAPでの使い方

ストレージ機能としてストレージ全体を遠隔地にレプリケーションする機能があります。(SVM-DR)

ここでは Heptio ark をつかい、kubernetesの構成情報をバックアップ/リストアし、
実データをSVM-DRを用いデータ転送後、別Kubernetes環境でリストアする方法について説明します。

### Heptio Ark

Heptio 社が提供しているkubernetesのデファクトバックアップ・リストアツールです。

- https://github.com/heptio/ark

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