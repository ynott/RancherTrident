## Trident のバックエンドストレージに設定したONTAPでの使い方

ストレージ機能としてストレージ全体を遠隔地にレプリケーションする機能があります。(SVM-DR)

ここでは Heptio ark をつかい、kubernetesの構成情報をバックアップ/リストアし、
実データをSVM-DRを用いデータ転送後、別Kubernetes環境でリストアする方法について説明します。


### Heptio Ark

Heptio 社が提供しているkubernetesのデファクトバックアップ・リストアツールです。