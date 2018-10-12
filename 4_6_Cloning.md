## Trident Fast PVC Cloningの使い方とユースケース

Tridentの特徴的な機能の１つである、``PVC Fast Cloning``について紹介します。

### PVC Fast Cloningとは

既存のPVCを高速にコピーする機能です。
この機能を活用すると実現できることとしては以下の内容のものがあります。

- テストデータ設定済みの Database as a Service の実現
- テスト用途の環境を瞬時にコピー
- 本番環境からデータをコピーして、調査用の環境を作成し what-if 分析

また、ストレージ容量を消費しないというメリットもあります。

### 実際の設定方法

ここでは実際に使用するマニフェストを提示し、設定すべき項目について説明いたします。

まずはmysqlのパスワードをsecretに入れます。

```
$ kubectl create secret generic mysql-pass --from-literal=password=yourpassword -n [ネームスペース]
```

mysql自体のデプロイメントは以下のようなものをつかいます。
ここではベースとなるPVC ``mysql-pv-claim``を作成します。

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysqldatabase
  labels:
    app: database
spec:
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - image: mysql:8
        name: mysqldatabase
        env:
        - name: PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysqldatabase
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: database
  annotations:
    trident.netapp.io/reclaimPolicy: "Retain"
    trident.netapp.io/exportPolicy: "default"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: ontap-gold
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: database
spec:
  ports:
    - port: 3306
  selector:
    app: database
    tier: mysql
  type: NodePort
```

起動後にデータを投入し元となるデータベースを作成します。

クローン元のPVCにはI/Oが発生していないことを強く推奨します。

クローニングはPVCのannotationに ``netapp.io/cloneFromPVC`` を設定し、コピーもとのPVCを指定することで実現できます。

以下の記述を追記します。

``` yaml
annotation:
    netapp.io/cloneFromPVC: mysql-pv-claim
```

上記の通りマニフェストを複数展開する方法もありますが、Helm を使うことでより簡単に展開することができます。
以下の２つの項目をhelm実行時に設定します。

- persistence.storageClass=ontap-gold
- persistence.annotations={netapp.io/cloneFromPVC: XXX}

``` console
$ helm install stable/mysql --name [リリース名] --namespace [ネームスペース] --set persistence.storageClass=ontap-gold --set persistence.annotations."netapp\.io/cloneFromPVC"=[コピー元のPVC名]
```

マニフェストを随時書くのでははなく、上記のように必要部分（今回はストレージクラス名とannotation) のみを定義し迅速に展開することができます。

今回の例はデータベースを例として説明しましたが、大量のデータ・セットを配るようなオペレーションをするときにも同様の手段で実施することができます。
