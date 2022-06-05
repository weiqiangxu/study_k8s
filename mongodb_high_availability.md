# mongodb基于k8s实现高可用


### 高可用原理

全面剖析 MongoDB 高可用架构

https://mp.weixin.qq.com/s/jLsviuQ0wCcsmkskXSFdEQ



MongoDB 副本集之入门篇

https://jelly.jd.com/article/5f990ebbbfbee00150eb620a



Mongodb主从复制/ 副本集/分片集群介绍

https://www.cnblogs.com/kevingrace/p/5685486.html



### 实践

kubernetes生产实践之mongodb

https://zhuanlan.zhihu.com/p/356658594

### 测试验证


### 一、MongoDB ReplicaSet

1. kube支持的mongodb版本

```
kubectl get mongodbversions
```

2. 定义配置文件，创建secret

touch mongod.conf

```
net:
   maxIncomingConnections: 10000
```

kubectl create secret generic -n op mg-configuration --from-file=./mongod.conf

kubectl get secret -n op


3. mongod-replicaset.yaml

```
apiVersion: kubedb.com/v1alpha2
kind: MongoDB
metadata:
  name: mongodb-replicaset
  namespace: op
spec:
  version: "4.2.3"
  replicas: 3
  replicaSet:
    name: rs0
  configSecret:
    name: mg-configuration
  storage:
    storageClassName: "rbd"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

4. 执行安装 

```
kubectl apply -f mongod-replicaset.yaml

kubectl get po -n op
```

5. 查看用户名密码

```
kubectl get secrets -n demo mgo-replicaset-auth -o jsonpath='{.data.\username}' | base64 -d

kubectl get secrets -n demo mgo-replicaset-auth -o jsonpath='{.data.\password}' | base64 -d
```

6. 登陆集群

```
kubectl -n op  exec -ti mongodb-replicaset-0 -- /bin/bash

mongo admin -uroot -p123456

查看当前PRIMARY节点

rs.isMaster().primary

show dbs
```


### 二、MongoDB Sharding

1. 编写mongodb-sharding.yaml

```
apiVersion: kubedb.com/v1alpha2
kind: MongoDB
metadata:
  name: mongo-sharding
  namespace: op
spec:
  version: 4.2.3
  shardTopology:
    configServer:
      replicas: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: rbd
    mongos:
      replicas: 2
    shard:
      replicas: 3
      shards: 2
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: rbd
```

2. 执行部署，并查看部署状态

```
kubectl apply -f mongodb-sharding.yaml
kubectl get po -n op
```

3. 验证集群状态及读写

# 获取账号密码
kubectl get secrets -n demo mongo-sh-auth -o jsonpath='{.data.\username}' | base64 -d
root
kubectl get secrets -n demo mongo-sh-auth -o jsonpath='{.data.\password}' | base64 -d
123456

mongo admin -u root -p

查看分片集群状态
sh.status()


创建分片
sh.enableSharding("test");

创建集合
sh.shardCollection("test.testcoll", {"myfield": 1});

写入数据
db.testcoll.insert({"myfield": "scofield", "agefield": "18"});
db.testcoll.insert({"myfield": "amos", "otherfield": "d", "kube" : "db" });

获取数据
db.testcoll.find();














