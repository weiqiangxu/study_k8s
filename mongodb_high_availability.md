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

http://t.zoukankan.com/klvchen-p-13685380.html

### 测试验证


### 一、MongoDB ReplicaSet

1. 创建命名空间

```
kubectl create namespace dev
# 查看节点名称
kubectl get nodes
```

2. mongod-replicaset.yaml

```
apiVersion: apps/v1 
kind: Deployment
metadata:
  namespace: dev
  name: mongodb
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      nodeName: k8s-node1    # 固定在 k8s-node1 节点
      containers:
      - name: mongodb
        image: mongo:4.2.9
        resources:
          limits:            # 限定资源
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 1Gi
        env:
          - name: MONGO_INITDB_ROOT_USERNAME  # 设置用户名
            value: root
          - name: MONGO_INITDB_ROOT_PASSWORD  # 设置密码
            value: '123456'
        volumeMounts:
          - mountPath: /data/db                    
            name: mongodb-volume
      volumes:
        - name: mongodb-volume
          hostPath:
            path: /data/rs-mongodb-volume          # 映射的宿主机目录
            type: DirectoryOrCreate
 
---

apiVersion: v1
kind: Service
metadata:
  namespace: dev
  name: mongodb
spec:
  type: ClusterIP
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
```

4. 执行安装 

```
kubectl apply -f mongod-replicaset.yaml
kubectl get pod -n dev -o wide
# 查看详细-如果起不来需要用于排查原因
kubectl describe pod mongodb -n dev
```

5. 查看用户名密码

```
kubectl -n dev exec -it POD_NAME /bin/bash
```

6. 登陆集群

```
mongo admin

db.auth('root','123456')

use test

db.createUser(
   {
     user: "test",
     pwd: "test123",
     roles: [ { role: "readWrite", db: "test" } ]
   }
)

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
kubectl exec -it podName  -c  containerName -n namespace -- shell comand

kubectl exec -it mongodb -n op -- shell comand
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














