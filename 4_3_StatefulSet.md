## StatefulSetを使ったダイナミックプロビジョニングの例

データが分散され保管されるアーキテクチャの場合、スケールするごとにストレージをプロビジョニングする必要があります。

このセクションではMongoDBのレプリカ数を複数としてどのようにプロビジョニングできるかということデプロイしていきます。

MongoDBのインストールにはHelmを使いインストールします。

![MongoDB + StatefulSet](images/4_MongoDBArch.png)


### Helm用設定ファイルの準備

適応する ``values.yaml`` は以下の通りです。
デフォルトからはdynamic provisioningをするため ``storageClass`` を変更しています。

``` yaml values.yaml

replicas: 3
port: 27017

replicaSetName: rs0

podDisruptionBudget: {}
  # maxUnavailable: 1
  # minAvailable: 2

auth:
  enabled: false
  # adminUser: username
  # adminPassword: password
  # metricsUser: metrics
  # metricsPassword: password
  # key: keycontent
  # existingKeySecret:
  # existingAdminSecret:
  # exisitingMetricsSecret:

# Specs for the Docker image for the init container that establishes the replica set
installImage:
  repository: k8s.gcr.io/mongodb-install
  tag: 0.6
  pullPolicy: IfNotPresent

# Specs for the MongoDB image
image:
  repository: mongo
  tag: 3.6
  pullPolicy: IfNotPresent

# Additional environment variables to be set in the container
extraVars: {}
# - name: TCMALLOC_AGGRESSIVE_DECOMMIT
#   value: "true"

# Prometheus Metrics Exporter
metrics:
  enabled: false
  image:
    repository: ssalaues/mongodb-exporter
    tag: 0.6.1
    pullPolicy: IfNotPresent
  port: 9216
  path: "/metrics"
  socketTimeout: 3s
  syncTimeout: 1m
  prometheusServiceDiscovery: true
  resources: {}

# Annotations to be added to MongoDB pods
podAnnotations: {}

securityContext:
  runAsUser: 999
  fsGroup: 999
  runAsNonRoot: true

resources: {}
# limits:
#   cpu: 500m
#   memory: 512Mi
# requests:
#   cpu: 100m
#   memory: 256Mi

## Node selector
## ref: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector
nodeSelector: {}

affinity: {}

tolerations: []

extraLabels: {}

persistentVolume:
  enabled: true
  ## mongodb-replicaset data Persistent Volume Storage Class
  ## If defined, storageClassName: <storageClass>
  ## If set to "-", storageClassName: "", which disables dynamic provisioning
  ## If undefined (the default) or set to null, no storageClassName spec is
  ##   set, choosing the default provisioner.  (gp2 on AWS, standard on
  ##   GKE, AWS & OpenStack)
  ##
  storageClass: "solidfire-gold"
  accessModes:
    - ReadWriteOnce
  size: 10Gi
  annotations: {}

# Annotations to be added to the service
serviceAnnotations: {}

tls:
  # Enable or disable MongoDB TLS support
  enabled: false
  # Please generate your own TLS CA by generating it via:
  # $ openssl genrsa -out ca.key 2048
  # $ openssl req -x509 -new -nodes -key ca.key -days 10000 -out ca.crt -subj "/CN=mydomain.com"
  # After that you can base64 encode it and paste it here:
  # $ cat ca.key | base64 -w0
  # cacert:
  # cakey:

# Entries for the MongoDB config file
configmap:

# Readiness probe
readinessProbe:
  initialDelaySeconds: 5
  timeoutSeconds: 1
  failureThreshold: 3 
  periodSeconds: 10
  successThreshold: 1

# Liveness probe
livenessProbe:
  initialDelaySeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3
  periodSeconds: 10
  successThreshold: 1

```

### MongoDB デプロイ

MongoDB をインストールします。

MongoDBをデプロイするネームスペースを作成します。

``` console

$ kubectl create ns mongo-replica

namespace/mongo-replica created
```

以下のURLを参考に、replicaset の永続化のバックエンドストレージにはiSCSIプロトコルを使用したSolidFireを使用しています。

- https://github.com/helm/charts/tree/master/stable/mongodb-replicaset

Helmを使いMongoDBをデプロイします。

``` console
$ helm install --name mongodb --namespace mongo-replica -f values.yaml stable/mongodb-replicaset

NAME:   mongodb
LAST DEPLOYED: Fri Jul 27 20:43:15 2018
NAMESPACE: mongo-replica
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                                DATA  AGE
mongodb-mongodb-replicaset-init     1     0s
mongodb-mongodb-replicaset-mongodb  1     0s
mongodb-mongodb-replicaset-tests    1     0s

==> v1/Service
NAME                        TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)    AGE
mongodb-mongodb-replicaset  ClusterIP  None        <none>       27017/TCP  0s

==> v1beta2/StatefulSet
NAME                        DESIRED  CURRENT  AGE
mongodb-mongodb-replicaset  3        1        0s

==> v1/Pod(related)
NAME                          READY  STATUS   RESTARTS  AGE
mongodb-mongodb-replicaset-0  0/1    Pending  0         0s


NOTES:
1. After the statefulset is created completely, one can check which instance is primary by running:

    $ for ((i = 0; i < 3; ++i)); do kubectl exec --namespace mongo-replica mongodb-mongodb-replicaset-$i -- sh -c 'mongo --eval="printjson(rs.isMaster())"'; done

2. One can insert a key into the primary instance of the mongodb replica set by running the following:
    MASTER_POD_NAME must be replaced with the name of the master found from the previous step.

    $ kubectl exec --namespace mongo-replica MASTER_POD_NAME -- mongo --eval="printjson(db.test.insert({key1: 'value1'}))"

3. One can fetch the keys stored in the primary or any of the slave nodes in the following manner.
    POD_NAME must be replaced by the name of the pod being queried.

    $ kubectl exec --namespace mongo-replica POD_NAME -- mongo --eval="rs.slaveOk(); db.test.find().forEach(printjson)"
```

生成されたKubernetesオブジェクトを確認します。

```　console
$ kubectl get all -n mongo-replica

NAME                               READY     STATUS    RESTARTS   AGE
pod/mongodb-mongodb-replicaset-0   1/1       Running   0          3m
pod/mongodb-mongodb-replicaset-1   1/1       Running   0          2m
pod/mongodb-mongodb-replicaset-2   1/1       Running   0          1m

NAME                                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
service/mongodb-mongodb-replicaset   ClusterIP   None         <none>        27017/TCP   3m

NAME                                          DESIRED   CURRENT   AGE
statefulset.apps/mongodb-mongodb-replicaset   3         3         3m

$ kubectl get statefulset -n mongo-replica
NAME                         DESIRED   CURRENT   AGE
mongodb-mongodb-replicaset   3         3         3m

$ kubectl describe statefulset -n mongo-replica
Name:               mongodb-mongodb-replicaset
Namespace:          mongo-replica
CreationTimestamp:  Fri, 27 Jul 2018 20:43:15 +0900
Selector:           app=mongodb-replicaset,release=mongodb
Labels:             app=mongodb-replicaset
                    chart=mongodb-replicaset-3.5.2
                    heritage=Tiller
                    release=mongodb
Annotations:        <none>
Replicas:           3 desired | 3 total
Update Strategy:    RollingUpdate
Pods Status:        3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       app=mongodb-replicaset
                release=mongodb
  Annotations:  prometheus.io/path=/metrics
                prometheus.io/port=9216
                prometheus.io/scrape=true
  Init Containers:
   copy-config:
    Image:      busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
    Args:
      -c
      set -e
set -x

cp /configdb-readonly/mongod.conf /data/configdb/mongod.conf

    Environment:  <none>
    Mounts:
      /configdb-readonly from config (rw)
      /data/configdb from configdir (rw)
      /work-dir from workdir (rw)
   install:
    Image:      k8s.gcr.io/mongodb-install:0.6
    Port:       <none>
    Host Port:  <none>
    Args:
      --work-dir=/work-dir
    Environment:  <none>
    Mounts:
      /work-dir from workdir (rw)
   bootstrap:
    Image:      mongo:3.6
    Port:       <none>
    Host Port:  <none>
    Command:
      /work-dir/peer-finder
    Args:
      -on-start=/init/on-start.sh
      -service=mongodb-mongodb-replicaset
    Environment:
      POD_NAMESPACE:   (v1:metadata.namespace)
      REPLICA_SET:    rs0
    Mounts:
      /data/configdb from configdir (rw)
      /data/db from datadir (rw)
      /init from init (rw)
      /work-dir from workdir (rw)
  Containers:
   mongodb-replicaset:
    Image:      mongo:3.6
    Port:       27017/TCP
    Host Port:  0/TCP
    Command:
      mongod
    Args:
      --config=/data/configdb/mongod.conf
      --dbpath=/data/db
      --replSet=rs0
      --port=27017
      --bind_ip=0.0.0.0
    Liveness:     exec [mongo --eval db.adminCommand('ping')] delay=30s timeout=5s period=10s #success=1 #failure=3
    Readiness:    exec [mongo --eval db.adminCommand('ping')] delay=5s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /data/configdb from configdir (rw)
      /data/db from datadir (rw)
      /work-dir from workdir (rw)
  Volumes:
   config:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      mongodb-mongodb-replicaset-mongodb
    Optional:  false
   init:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      mongodb-mongodb-replicaset-init
    Optional:  false
   workdir:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
   configdir:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
Volume Claims:
  Name:          datadir
  StorageClass:  solidfire-gold
  Labels:        <none>
  Annotations:   <none>
  Capacity:      10Gi
  Access Modes:  [ReadWriteOnce]
Events:
  Type    Reason            Age   From                    Message
  ----    ------            ----  ----                    -------
  Normal  SuccessfulCreate  3m    statefulset-controller  create Claim datadir-mongodb-mongodb-replicaset-0 Pod mongodb-mongodb-replicaset-0 in StatefulSet mongodb-mongodb-replicaset success
  Normal  SuccessfulCreate  3m    statefulset-controller  create Pod mongodb-mongodb-replicaset-0 in StatefulSet mongodb-mongodb-replicaset successful
  Normal  SuccessfulCreate  2m    statefulset-controller  create Claim datadir-mongodb-mongodb-replicaset-1 Pod mongodb-mongodb-replicaset-1 in StatefulSet mongodb-mongodb-replicaset success
  Normal  SuccessfulCreate  2m    statefulset-controller  create Pod mongodb-mongodb-replicaset-1 in StatefulSet mongodb-mongodb-replicaset successful
  Normal  SuccessfulCreate  1m    statefulset-controller  create Claim datadir-mongodb-mongodb-replicaset-2 Pod mongodb-mongodb-replicaset-2 in StatefulSet mongodb-mongodb-replicaset success
  Normal  SuccessfulCreate  1m    statefulset-controller  create Pod mongodb-mongodb-replicaset-2 in StatefulSet mongodb-mongodb-replicaset successful
```

kubernetes再度より作成されたPVCを確認します。

``` console
$ kubectl get pvc -n mongo-replica -o wide

NAME                                   STATUS    VOLUME                                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
datadir-mongodb-mongodb-replicaset-0   Bound     mongo-replica-datadir-mongodb-mongodb-replicaset-0-3f8e9   10Gi       RWO            solidfire-gold   24m
datadir-mongodb-mongodb-replicaset-1   Bound     mongo-replica-datadir-mongodb-mongodb-replicaset-1-5f8c1   10Gi       RWO            solidfire-gold   23m
datadir-mongodb-mongodb-replicaset-2   Bound     mongo-replica-datadir-mongodb-mongodb-replicaset-2-7baf3   10Gi       RWO            solidfire-gold   22m
```

バックエンドのストレージに作成された実態のストレージボリュームを確認します。

``` console
+----------------------------------------------------------+--------+----------------+----------+-------------------------+----------+
|                           NAME                           |  SIZE  | STORAGE CLASS  | PROTOCOL |         BACKEND         |   POOL   |
+----------------------------------------------------------+--------+----------------+----------+-------------------------+----------+
| mongo-replica-datadir-mongodb-mongodb-replicaset-2-7baf3 | 10 GiB | solidfire-gold | block    | solidfire_192.168.0.240 | Gold     |
| default-mysql-pv-claim-4c942                             | 20 GiB | ontap-gold     | file     | svm12                   | aggr1_01 |
| default-wp-pv-claim-7bb75                                | 20 GiB | ontap-gold     | file     | svm12                   | aggr1_01 |
| mongo-replica-datadir-mongodb-mongodb-replicaset-0-3f8e9 | 10 GiB | solidfire-gold | block    | solidfire_192.168.0.240 | Gold     |
| mongo-replica-datadir-mongodb-mongodb-replicaset-1-5f8c1 | 10 GiB | solidfire-gold | block    | solidfire_192.168.0.240 | Gold     |
+----------------------------------------------------------+--------+----------------+----------+-------------------------+----------+
```

MongoDBのマスターノードを確認します、マスタが選出され正常稼働していることが確認できました。

``` console
$ for ((i = 0; i < 3; ++i)); do kubectl exec --namespace mongo-replica mongodb-mongodb-replicaset-$i -- sh -c 'mongo --eval="printjson(rs.isMaster())"'; done

MongoDB shell version v3.6.6
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.6.6
{
        "hosts" : [
                "mongodb-mongodb-replicaset-0.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
                "mongodb-mongodb-replicaset-1.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
                "mongodb-mongodb-replicaset-2.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017"
        ],
        "setName" : "rs0",
        "setVersion" : 3,
        "ismaster" : true,
        "secondary" : false,
        "primary" : "mongodb-mongodb-replicaset-0.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
        "me" : "mongodb-mongodb-replicaset-0.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
        "electionId" : ObjectId("7fffffff0000000000000002"),
        "lastWrite" : {
                "opTime" : {
                        "ts" : Timestamp(1532692403, 1),
                        "t" : NumberLong(2)
                },
                "lastWriteDate" : ISODate("2018-07-27T11:53:23Z"),
                "majorityOpTime" : {
                        "ts" : Timestamp(1532692403, 1),
                        "t" : NumberLong(2)
                },
                "majorityWriteDate" : ISODate("2018-07-27T11:53:23Z")
        },
        "maxBsonObjectSize" : 16777216,
        "maxMessageSizeBytes" : 48000000,
        "maxWriteBatchSize" : 100000,
        "localTime" : ISODate("2018-07-27T11:53:25.350Z"),
        "logicalSessionTimeoutMinutes" : 30,
        "minWireVersion" : 0,
        "maxWireVersion" : 6,
        "readOnly" : false,
        "ok" : 1,
        "operationTime" : Timestamp(1532692403, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1532692403, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}
MongoDB shell version v3.6.6
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.6.6
{
        "hosts" : [
                "mongodb-mongodb-replicaset-0.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
                "mongodb-mongodb-replicaset-1.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
                "mongodb-mongodb-replicaset-2.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017"
        ],
        "setName" : "rs0",
        "setVersion" : 3,
        "ismaster" : false,
        "secondary" : true,
        "primary" : "mongodb-mongodb-replicaset-0.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
        "me" : "mongodb-mongodb-replicaset-1.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
        "lastWrite" : {
                "opTime" : {
                        "ts" : Timestamp(1532692403, 1),
                        "t" : NumberLong(2)
                },
                "lastWriteDate" : ISODate("2018-07-27T11:53:23Z"),
                "majorityOpTime" : {
                        "ts" : Timestamp(1532692403, 1),
                        "t" : NumberLong(2)
                },
                "majorityWriteDate" : ISODate("2018-07-27T11:53:23Z")
        },
        "maxBsonObjectSize" : 16777216,
        "maxMessageSizeBytes" : 48000000,
        "maxWriteBatchSize" : 100000,
        "localTime" : ISODate("2018-07-27T11:53:25.705Z"),
        "logicalSessionTimeoutMinutes" : 30,
        "minWireVersion" : 0,
        "maxWireVersion" : 6,
        "readOnly" : false,
        "ok" : 1,
        "operationTime" : Timestamp(1532692403, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1532692403, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}
MongoDB shell version v3.6.6
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.6.6
{
        "hosts" : [
                "mongodb-mongodb-replicaset-0.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
                "mongodb-mongodb-replicaset-1.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
                "mongodb-mongodb-replicaset-2.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017"
        ],
        "setName" : "rs0",
        "setVersion" : 3,
        "ismaster" : false,
        "secondary" : true,
        "primary" : "mongodb-mongodb-replicaset-0.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
        "me" : "mongodb-mongodb-replicaset-2.mongodb-mongodb-replicaset.mongo-replica.svc.cluster.local:27017",
        "lastWrite" : {
                "opTime" : {
                        "ts" : Timestamp(1532692403, 1),
                        "t" : NumberLong(2)
                },
                "lastWriteDate" : ISODate("2018-07-27T11:53:23Z"),
                "majorityOpTime" : {
                        "ts" : Timestamp(1532692403, 1),
                        "t" : NumberLong(2)
                },
                "majorityWriteDate" : ISODate("2018-07-27T11:53:23Z")
        },
        "maxBsonObjectSize" : 16777216,
        "maxMessageSizeBytes" : 48000000,
        "maxWriteBatchSize" : 100000,
        "localTime" : ISODate("2018-07-27T11:53:26.068Z"),
        "logicalSessionTimeoutMinutes" : 30,
        "minWireVersion" : 0,
        "maxWireVersion" : 6,
        "readOnly" : false,
        "ok" : 1,
        "operationTime" : Timestamp(1532692403, 1),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1532692403, 1),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}
```

### ノード障害発生時の動作確認

#### 確認事項

ノード障害が発生し、Deploymentで管理されているポッドが別ポッドとして起動した場合にもデータが永続化されていることを確認します。

以下のフローで永続化の確認をします。

1. 障害発生前にデータを保存
2. Deploymentで管理されているPodをすべて削除
3. 再度DeploymentでPodが起動され、別ノードで起動時に「1.」で保存したデータを参照できることを確認

確認のためのテストデータを投入します。

``` console
$ kubectl exec mongodb-mongodb-replicaset-0 -n mongo-replica -- mongo --eval="printjson(db.test.insert({key1: 'trident fail test'}))"
MongoDB shell version v3.6.6
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.6.6
{ "nInserted" : 1 }
```

投入したデータを確認します。

``` console

$ kubectl exec mongodb-mongodb-replicaset-0 -n mongo-replica -- mongo --eval="rs.slaveOk(); db.test.find({key1:{\$exists:true}}).forEach(printjson)"

MongoDB shell version v3.6.6
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.6.6
{
        "_id" : ObjectId("5b5b0d43e521f72e61cf2f9f"),
        "key1" : "trident fail test"
}
```

Repulica Set各ノードに配置されています。

``` console
mongodb-mongodb-replicaset-0   1/1       Running   0          5m        10.244.3.6   node2              │
mongodb-mongodb-replicaset-1   1/1       Running   0          5m        10.244.4.8   node3              │
mongodb-mongodb-replicaset-2   1/1       Running   0          4m        10.244.1.5   node0
```

ここでHelm Chartsで付与されているラベルでPodをすべて削除します。

``` console
localadmin@master:~/manifest/mongdb-helm$ kubectl delete po -l "app=mongodb-replicaset,release=mongodb" -n mongo-replica
pod "mongodb-mongodb-replicaset-0" deleted
pod "mongodb-mongodb-replicaset-1" deleted
pod "mongodb-mongodb-replicaset-2" deleted
localadmin@master:~/manifest/mongdb-helm$
```

Deployment により管理されているため、Podが停止したのを検知して起動されます。

``` console
$ kubectl get po --watch-only -n mongo-replica -o wide
NAME                           READY     STATUS        RESTARTS   AGE       IP           NODE
mongodb-mongodb-replicaset-0   1/1       Terminating   0          7m        10.244.3.6   node2
mongodb-mongodb-replicaset-1   1/1       Terminating   0         6m        10.244.4.8   node3
mongodb-mongodb-replicaset-2   1/1       Terminating   0         6m        10.244.1.5   node0
mongodb-mongodb-replicaset-0   1/1       Terminating   0         7m        10.244.3.6   node2
mongodb-mongodb-replicaset-1   1/1       Terminating   0         6m        10.244.4.8   node3
mongodb-mongodb-replicaset-2   1/1       Terminating   0         6m        10.244.1.5   node0
mongodb-mongodb-replicaset-0   0/1       Terminating   0         7m        10.244.3.6   node2
mongodb-mongodb-replicaset-1   0/1       Terminating   0         6m        <none>    node3
mongodb-mongodb-replicaset-2   0/1       Terminating   0         6m        <none>    node0
mongodb-mongodb-replicaset-2   0/1       Terminating   0         6m        <none>    node0
mongodb-mongodb-replicaset-1   0/1       Terminating   0         6m        <none>    node3
mongodb-mongodb-replicaset-2   0/1       Terminating   0         6m        <none>    node0
mongodb-mongodb-replicaset-1   0/1       Terminating   0         6m        <none>    node3
mongodb-mongodb-replicaset-0   0/1       Terminating   0         7m        10.244.3.6   node2
mongodb-mongodb-replicaset-0   0/1       Terminating   0         7m        10.244.3.6   node2
mongodb-mongodb-replicaset-0   0/1       Pending   0         0s        <none>    <none>
mongodb-mongodb-replicaset-0   0/1       Pending   0         0s        <none>    node3
mongodb-mongodb-replicaset-0   0/1       Init:0/3   0         0s        <none>    node3
mongodb-mongodb-replicaset-0   0/1       Init:1/3   0         25s       10.244.4.9   node3
mongodb-mongodb-replicaset-0   0/1       Init:2/3   0         26s       10.244.4.9   node3
mongodb-mongodb-replicaset-0   0/1       Init:2/3   0         27s       10.244.4.9   node3
mongodb-mongodb-replicaset-0   0/1       PodInitializing   0         30s       10.244.4.9   node3
mongodb-mongodb-replicaset-0   0/1       Running   0         31s       10.244.4.9   node3
mongodb-mongodb-replicaset-0   1/1       Running   0         44s       10.244.4.9   node3
mongodb-mongodb-replicaset-1   0/1       Pending   0         0s        <none>    <none>
mongodb-mongodb-replicaset-1   0/1       Pending   0         0s        <none>    node2
mongodb-mongodb-replicaset-1   0/1       Init:0/3   0         0s        <none>    node2
mongodb-mongodb-replicaset-1   0/1       Init:1/3   0         17s       10.244.3.7   node2
mongodb-mongodb-replicaset-1   0/1       Init:2/3   0         18s       10.244.3.7   node2
mongodb-mongodb-replicaset-1   0/1       Init:2/3   0         19s       10.244.3.7   node2
mongodb-mongodb-replicaset-1   0/1       PodInitializing   0         22s       10.244.3.7   node2
mongodb-mongodb-replicaset-1   0/1       Running   0         23s       10.244.3.7   node2
mongodb-mongodb-replicaset-1   1/1       Running   0         35s       10.244.3.7   node2
mongodb-mongodb-replicaset-2   0/1       Pending   0         0s        <none>    <none>
mongodb-mongodb-replicaset-2   0/1       Pending   0         0s        <none>    node1
mongodb-mongodb-replicaset-2   0/1       Init:0/3   0         0s        <none>    node1
mongodb-mongodb-replicaset-2   0/1       Init:1/3   0         9s        10.244.2.6   node1
mongodb-mongodb-replicaset-2   0/1       Init:2/3   0         10s       10.244.2.6   node1
mongodb-mongodb-replicaset-2   0/1       Init:2/3   0         11s       10.244.2.6   node1
mongodb-mongodb-replicaset-2   0/1       PodInitializing   0         18s       10.244.2.6   node1
mongodb-mongodb-replicaset-2   0/1       Running   0         19s       10.244.2.6   node1
mongodb-mongodb-replicaset-2   1/1       Running   0         27s       10.244.2.6   node1
```

最終的に起動したreplicasetがPod停止前と別のノードで起動されています。

``` console
$ kubectl get po -n mongo-replica -o wide
NAME                           READY     STATUS    RESTARTS   AGE       IP           NODE
mongodb-mongodb-replicaset-0   1/1       Running   0          2m        10.244.4.9   node3
mongodb-mongodb-replicaset-1   1/1       Running   0          1m        10.244.3.7   node2
mongodb-mongodb-replicaset-2   1/1       Running   0          1m        10.244.2.6   node1
```

テスト前に入れたデータベースの値を確認する。

``` console
$ kubectl exec mongodb-mongodb-replicaset-0 -n mongo-replica -- mongo --eval="rs.slaveOk(); db.test.find({key1:{\$exists:true}}).forEach(printjson)"
MongoDB shell version v3.6.6
connecting to: mongodb://127.0.0.1:27017
MongoDB server version: 3.6.6
{
        "_id" : ObjectId("5b5b0d43e521f72e61cf2f9f"),
        "key1" : "trident fail test"
}
```

ポッドが削除され、再作成された上でもデータは永続化している状態が確認できました。
