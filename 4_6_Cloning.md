## Trident Fast PVC Cloning の使い方とユースケース

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

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: oracle-pv-claim-clone
  labels:
    app: database-cloned
  annotations:
    trident.netapp.io/reclaimPolicy: "Delete"
    trident.netapp.io/exportPolicy: "default"
    trident.netapp.io/cloneFromPVC: "oracle-pv-claim"  
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Ti
  storageClassName: ontap-gold
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  oracle-se2-cloned
  labels:
    app:  database-cloned
spec:
  selector:
    matchLabels:
      app: database-cloned
  template:
    metadata:
      labels:
        app:  database-cloned
    spec:
      containers:
      - image: docker-registry.default.svc:5000/default/oracledb:12.2.0.1-se2
        name: oracle-se2
        env:
        - name: ORACLE_SID
          value: "tridentsid"
        - name: ORACLE_PDB
          value: "tridentpdb"
        - name: ORACLE_PWD
          value: "changeme"
        ports:
        - containerPort:  1521
          name:  oraclelistener
        - containerPort:  5500
          name:  manager
        volumeMounts:
        - mountPath: /opt/oracle/oradata
          name: oradata
      volumes:
        - name: oradata
          persistentVolumeClaim:
            claimName: oracle-pv-claim-clone
```

annotation を設定し、コピーもとのPVCを指定することで実現できます。

``` yaml

annotation:
    netapp.io/cloneFromPVC: XXX
```

