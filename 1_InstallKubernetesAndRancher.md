# 1. Rancher/Kubernetes の構築

## 1. Kubernetes環境の構成説明

Kubernetesクラスターの構成を以下のように構成する。
共有する環境としてDNSをdns1,dns2を設定し、Kubernetesクラスター毎のセグメントと分離する。Kubernetesクラスターには、Masterノードを1つ、Workernノードをnode0,node1を設置する。同じセグメント内にNetApp SVMを設置し、SSDとHDDのアグリゲートを作成し、それぞれaggr1_01、aggr2_01とした。

![Kubernetesサーバー構成](https://raw.githubusercontent.com/ynott/RancherTrident/master/images/KubernetesServers.png)

### 1. Kubernetesのインストール

Kubernetesクラスターのインストールには、kubeadmでインストールを行った。特別な操作はせず通常の方法でセットアップしたため、ここでは操作方法などは省略する。

## 2. Rancherの概要



## 3. Rancherのインストール
### 1. Rancherインストールに必要な構成
### 2. Dockerのインストール
### 3. Rancherのインストール

## 4. RancherからKubernetesのインポート

