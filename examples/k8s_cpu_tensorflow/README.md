# A Tutorial: Tensorflow on Kubernetes

本Tutorial记录了从零开始，在k8s上运行一个tensorflow分布式例子的过程

## 摘要
1. pull tensorflow images from github
2. push tensorflow images to harbor
3. 创建、编辑yaml

创建一个ps service(对应一个ps pod)<br>
创建两个worker service(分别对应一个worker pod)<br>

注意：因为ps和worker节点需要对外暴露自己位置(域名)，所以需要每个pod(container)对应一个service，并配合kube-dns进行服务发现，参考下面一个官方例子和一个民间例子<br>

https://github.com/tensorflow/ecosystem/blob/master/kubernetes/template.yaml.jinja <br>
https://github.com/amygdala/tensorflow-workshop/blob/9bbc678e686407c5dccff87db702f9aeef9e34b7/workshop_sections/distributed_tensorflow/k8s-configs/tf-cluster.yaml <br>

4. 获取tensorflow训练代码和数据

大致有两个途径 <br>

把代码和数据build进images，再分发到k8s节点。<br>

分发clean的tf image到k8s节点，通过command字段运行一个启动脚本，从外网curl到container，或挂载一个内网的nfs目录(或ceph)到container <br>

本文实验了从外网curl和从内网mount nfs以获取代码和数据

5. update 2016-11-02：
重新整理了yaml，增加了ConfigMap，简化ps节点和worker节点的定义

## 已知问题
1. 集群内gpu还在调试，暂时无法进行tensorflow的分布式多机多卡实验
2. 集群内的kube-dns似乎还不是很稳定，有时域名无法解析

## Requirements:
1. 正常运行的k8s集群
2. 正常运行的kube-dns服务发现功能
3. k8s集群外，一个nfs服务或ceph集群

## Step 1: Prepare Tensorflow Images:
```shell
[xuerq@bogon train]$ docker pull tensorflow/tensorflow
064f9af02539: Already exists 
390957b2f4f0: Already exists 
cee0974db2b8: Already exists 
ff4090f99abc: Pull complete 
70d51ddf7c95: Pull complete 
da76ab5d6dff: Pull complete 
46d4527e85d3: Pull complete 
528276ea4b2d: Pull complete 
709fc41158c6: Pull complete 
d63463802d36: Pull complete 
b4c3589c6b3a: Pull complete 
Digest: sha256:0635a564b59ac45e9a44d7c38efee9bf6c9567150236e064d8e382fbbf54fe74
Status: Downloaded newer image for tensorflow/tensorflow:latest
```
login harbor:
由于kubernete集群访问外网不是很方便，速度也无法保证。所以我们首先需要将镜像推送至私有仓库，方便日常使用
```shell
docker login https://harbor.ail.unisound.com
Username: xueruiqing
Password: 
Login Succeeded
```
tag & push image to harbor:
```shell
[xuerq@bogon train]$ docker tag ef9825a3a86f harbor.ail.unisound.com/xuerq/tensorflow:0.11.0
[xuerq@bogon train]$ docker push harbor.ail.unisound.com/xuerq/tensorflow:0.11.0
```
## Step 2: create ps & worker pod:
1. 从harbor pull 一个clean的tf image
2. 通过设置command字段，在容器启动时，执行一个curl数据并运行train脚本的动作
注：域名是根据service name生成的，如：
ps节点的域名：tensorflow-ps-service.default.svc.cluster.local
worker1 节点的域名：tensorflow-wk-service0.default.svc.cluster.local
worker2 节点的域名：tensorflow-wk-service1.default.svc.cluster.local
**ps.yaml:**
```yaml

apiVersion: v1
kind: ReplicationController
metadata:
  name: tensorflow-ps-rc
spec:
  replicas: 1
  selector:
    name: tensorflow-ps
  template:
    metadata:
      labels:
        name: tensorflow-ps
        role: ps
    spec:
      containers:
      - name: ps
        image: harbor.ail.unisound.com/xuerq/tensorflow:0.11.0
        ports:
        - containerPort: 2222
        command: ["/bin/sh", "-c"]
        args: ["curl \
                   https://codeload.github.com/tobegit3hub/deep_recommend_system/zip/master\
                   > drs.zip;
               unzip drs.zip;
               cd deep_recommend_system-master/distributed/;\
               python cancer_classifier.py \
                   --ps_hosts=tensorflow-ps-service.default.svc.cluster.local:2222 \
                   --worker_hosts=tensorflow-wk-service0.default.svc.cluster.local:2222,\
                                  tensorflow-wk-service1.default.svc.cluster.local:2222 \
                   --job_name=ps \
                  --task_index=0 >log 2>errlog
               "]
      nodeName: **********
```

**worker.yaml:**
```yaml

apiVersion: v1
kind: ReplicationController
metadata:
  name: tensorflow-worker0-rc
spec:
  replicas: 1
  selector:
    name: tensorflow-worker0
  template:
    metadata:
      labels:
        name: tensorflow-worker0
        role: worker
    spec:
      containers:
      - name: worker
        image: harbor.ail.unisound.com/xuerq/tensorflow:0.11.0
        ports:
        - containerPort: 2222
        command: ["/bin/sh", "-c"]
        args: ["curl \
                   https://codeload.github.com/tobegit3hub/deep_recommend_system/zip/master\
                   > drs.zip;
               unzip drs.zip;
               cd deep_recommend_system-master/distributed/;\
               python cancer_classifier.py \
                   --ps_hosts=tensorflow-ps-service.default.svc.cluster.local:2222 \
                   --worker_hosts=tensorflow-wk-service0.default.svc.cluster.local:2222,\
                                  tensorflow-wk-service1.default.svc.cluster.local:2222 \
                   --job_name=worker \
                  --task_index=0 1>log 2>errlog
               "]
      nodeName: **********
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: tensorflow-worker1-rc
spec:
  replicas: 1
  selector:
    name: tensorflow-worker1
  template:
    metadata:
      labels:
        name: tensorflow-worker1
        role: worker
    spec:
      containers:
      - name: worker
        image: harbor.ail.unisound.com/xuerq/tensorflow:0.11.0
        ports:
        - containerPort: 2222
        command: ["/bin/sh", "-c"]
        args: ["curl \
                   https://codeload.github.com/tobegit3hub/deep_recommend_system/zip/master\
                   > drs.zip;
               unzip drs.zip;
               cd deep_recommend_system-master/distributed/;\
               python cancer_classifier.py \
                   --ps_hosts=tensorflow-ps-service.default.svc.cluster.local:2222 \
                   --worker_hosts=tensorflow-wk-service0.default.svc.cluster.local:2222,\
                                  tensorflow-wk-service1.default.svc.cluster.local:2222 \
                   --job_name=worker \
                  --task_index=1 1>log 2>errlog
               "]
      nodeName: **********

```
```shell
[xuerq@bogon train]$ kubectl create -f ps.yaml
[xuerq@bogon train]$ kubectl create -f worker.yaml 
```
## Step 3: create ps & worker server:
**ps-srv.yaml:**
```yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    name: tensorflow-ps
    role: service
  name: tensorflow-ps-service
spec:
  ports:
    - port: 2222
      targetPort: 2222
  selector:
    name: tensorflow-ps
```
**worker-srv.yaml:**
```yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    name: tensorflow-worker0
    role: service
  name: tensorflow-wk-service0
spec:
  ports:
    - port: 2222
      targetPort: 2222
  selector:
    name: tensorflow-worker0
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: tensorflow-worker1
    role: service
  name: tensorflow-wk-service1
spec:
  ports:
    - port: 2222
      targetPort: 2222
  selector:
    name: tensorflow-worker1
```
```shell
[xuerq@bogon train]$ kubectl create -f ps-srv.yaml
[xuerq@bogon train]$ kubectl create -f worker-srv.yaml
```
## Step 4: 登陆节点配置证书(如需要):
```shell
core@00-25-90-c0-f6-ee ~/harbor $ sudo bash ./update_certs_coreos.sh harbor.ail.unisound.com ca.crt
...
```
ps和work似乎不能在同一node中并存，否则会tf会core掉
```shell
F tensorflow/core/distributed_runtime/graph_mgr.cc:55] 'unit.device' Must be non NULL
Aborted (core dumped)
```
## 部分结果
**work0**:
```
Step: 2520, loss: 0.0276989098638, accuracy: 0.9521484375, auc: 0.976532936096
Step: 2540, loss: 0.0272848084569, accuracy: 0.94921875, auc: 0.976501882076
Step: 2560, loss: 0.0282758921385, accuracy: 0.951171875, auc: 0.976519346237
Step: 2580, loss: 0.0278273113072, accuracy: 0.9423828125, auc: 0.976500332355
Step: 2600, loss: 0.0276386849582, accuracy: 0.9443359375, auc: 0.976470351219
Step: 2620, loss: 0.0275988020003, accuracy: 0.9462890625, auc: 0.976462066174
Step: 2640, loss: 0.02752911672, accuracy: 0.9541015625, auc: 0.976464569569
Step: 2660, loss: 0.0272494927049, accuracy: 0.9482421875, auc: 0.976428210735
Step: 2670, loss: 0.0255885906518, accuracy: 0.9423828125, auc: 0.976406395435
Step: 2690, loss: 0.0274116098881, accuracy: 0.943359375, auc: 0.976406633854
Step: 2700, loss: 0.0269550140947, accuracy: 0.9423828125, auc: 0.976405143738
Step: 2710, loss: 0.0257418975234, accuracy: 0.9453125, auc: 0.976407825947
Step: 2730, loss: 0.0290637034923, accuracy: 0.9443359375, auc: 0.976405143738
Step: 2750, loss: 0.025800453499, accuracy: 0.939453125, auc: 0.976356506348
Step: 2770, loss: 0.0271668545902, accuracy: 0.9384765625, auc: 0.976332306862
Step: 2780, loss: 0.024051791057, accuracy: 0.947265625, auc: 0.976318478584
Step: 2800, loss: 0.0246728882194, accuracy: 0.939453125, auc: 0.976280868053
Step: 2820, loss: 0.0277465172112, accuracy: 0.9453125, auc: 0.976256966591
Step: 2840, loss: 0.0266785826534, accuracy: 0.94921875, auc: 0.976251900196
Step: 2860, loss: 0.0254744384438, accuracy: 0.9462890625, auc: 0.976260721684
Step: 2880, loss: 0.0262948013842, accuracy: 0.9462890625, auc: 0.976273298264
Step: 2890, loss: 0.0251776557416, accuracy: 0.9462890625, auc: 0.976263046265
Step: 2910, loss: 0.0248399861157, accuracy: 0.94140625, auc: 0.976237237453
Step: 2930, loss: 0.0244574509561, accuracy: 0.9453125, auc: 0.976218521595
Step: 2940, loss: 0.0245014317334, accuracy: 0.951171875, auc: 0.97965067625
Step: 2950, loss: 0.0238579288125, accuracy: 0.9404296875, auc: 0.97618919611
Step: 2960, loss: 0.0254054088145, accuracy: 0.9462890625, auc: 0.976156651974
Step: 2970, loss: 0.0292413253337, accuracy: 0.947265625, auc: 0.976162552834
Step: 2980, loss: 0.0324414521456, accuracy: 0.951171875, auc: 0.97615903616
Step: 3000, loss: 0.0259569510818, accuracy: 0.9462890625, auc: 0.976138174534
Step: 3020, loss: 0.0257361158729, accuracy: 0.9453125, auc: 0.976105451584
```
**work1:**
```
Step: 2400, loss: 0.0288917049766, accuracy: 0.9423828125, auc: 0.976580619812
Step: 2420, loss: 0.0278494395316, accuracy: 0.9482421875, auc: 0.976597607136
Step: 2440, loss: 0.0287574753165, accuracy: 0.947265625, auc: 0.976580560207
Step: 2450, loss: 0.0289686173201, accuracy: 0.943359375, auc: 0.976590812206
Step: 2460, loss: 0.0286611653864, accuracy: 0.9443359375, auc: 0.976590335369
Step: 2480, loss: 0.026562217623, accuracy: 0.9443359375, auc: 0.976576983929
Step: 2500, loss: 0.0298435036093, accuracy: 0.9482421875, auc: 0.976565361023
Step: 2530, loss: 0.0272848941386, accuracy: 0.943359375, auc: 0.976512610912
Step: 2550, loss: 0.0276657473296, accuracy: 0.9443359375, auc: 0.976513445377
Step: 2570, loss: 0.0321980603039, accuracy: 0.94140625, auc: 0.976504266262
Step: 2590, loss: 0.0256077200174, accuracy: 0.9404296875, auc: 0.976488888264
Step: 2610, loss: 0.0280858129263, accuracy: 0.94140625, auc: 0.976459980011
Step: 2630, loss: 0.0283435098827, accuracy: 0.9482421875, auc: 0.976456165314
Step: 2650, loss: 0.0268683731556, accuracy: 0.9482421875, auc: 0.97644418478
Step: 2680, loss: 0.0273669157177, accuracy: 0.9423828125, auc: 0.976413726807
Step: 2720, loss: 0.027266446501, accuracy: 0.9541015625, auc: 0.976407945156
Step: 2740, loss: 0.0260527543724, accuracy: 0.93359375, auc: 0.976380228996
Step: 2760, loss: 0.0303766354918, accuracy: 0.947265625, auc: 0.976351678371
Step: 2790, loss: 0.0270864069462, accuracy: 0.9365234375, auc: 0.976292490959
Step: 2810, loss: 0.0260408222675, accuracy: 0.94140625, auc: 0.976268112659
Step: 2830, loss: 0.026640124619, accuracy: 0.9541015625, auc: 0.976262569427
Step: 2850, loss: 0.0259609017521, accuracy: 0.94921875, auc: 0.976246595383
Step: 2870, loss: 0.0263495109975, accuracy: 0.9560546875, auc: 0.976269721985
Step: 2900, loss: 0.0269010905176, accuracy: 0.9482421875, auc: 0.97625631094
Step: 2920, loss: 0.0251851622015, accuracy: 0.9453125, auc: 0.976214468479
Step: 2940, loss: 0.0265841782093, accuracy: 0.951171875, auc: 0.976156234741
```

## Step 5: **Update** (2016-11-02)：: Add nfs & ConfigMap
1. 从harbor pull 一个clean的tf image
2. 通过设置volumeMounts和volumes字段mount一个nfs 目录到容器中
3. 通过设置ConfigMap，把一些train脚本运行参数写入容器的环境变量
    主要是ps节点和workder节点的域名
4. 通过设置command字段，在容器启动后，执行一个从nfs拷贝数据和代码、并运行train脚本(配合ConfigMap设置的环境变量)的动作

**work_ps.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tensorflow-cluster-config
data:
  ps: 
     "tensorflow-ps-service.default.svc.cluster.local:2222"
  worker:
     "tensorflow-wk-service0.default.svc.cluster.local:2222,tensorflow-wk-service1.default.svc.cluster.local:2222"
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: tensorflow-ps-rc
spec:
  replicas: 1
  selector:
    name: tensorflow-ps
  template:
    metadata:
      labels:
        name: tensorflow-ps
        role: ps
    spec:
      containers:
      - name: ps
        image: harbor.ail.unisound.com/xuerq/tensorflow:0.11.0
        ports:
        - containerPort: 2222
        env:
        - name: PS_KEY
          valueFrom:
            configMapKeyRef:
              name: tensorflow-cluster-config
              key: ps
        - name: WORKER_KEY
          valueFrom:
            configMapKeyRef:
              name: tensorflow-cluster-config
              key: worker
        command: ["/bin/sh", "-c"]
        args: ["cp -r /nfs/deep_recommend_system-master ./;\
               cd deep_recommend_system-master/distributed/;\
               python cancer_classifier.py \
                   --ps_hosts=$(PS_KEY) \
                   --worker_hosts=$(WORKER_KEY) \
                   --job_name=ps \
                   --task_index=0 1>log 2>errlog
               "]
        volumeMounts:
        - name: nfs
          mountPath: "/nfs"
      volumes:
      - name: nfs
        nfs:
          server: 10.10.10.39
          path: "/home/xuerq/nfs"
      nodeName: **********
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: tensorflow-ps
    role: service
  name: tensorflow-ps-service
spec:
  ports:
    - port: 2222
      targetPort: 2222
  selector:
    name: tensorflow-ps
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: tensorflow-worker0-rc
spec:
  replicas: 1
  selector:
    name: tensorflow-worker0
  template:
    metadata:
      labels:
        name: tensorflow-worker0
        role: worker
    spec:
      containers:
      - name: worker
        image: harbor.ail.unisound.com/xuerq/tensorflow:0.11.0
        ports:
        - containerPort: 2222
        env:
        - name: PS_KEY
          valueFrom:
            configMapKeyRef:
              name: tensorflow-cluster-config
              key: ps
        - name: WORKER_KEY
          valueFrom:
            configMapKeyRef:
              name: tensorflow-cluster-config
              key: worker
        command: ["/bin/sh", "-c"]
        args: ["cp -r /nfs/deep_recommend_system-master ./;\
               cd deep_recommend_system-master/distributed/;\
               python cancer_classifier.py \
                   --ps_hosts=$(PS_KEY) \
                   --worker_hosts=$(WORKER_KEY) \
                   --job_name=worker \
                   --task_index=0 1>log 2>errlog
               "]
        volumeMounts:
        - name: nfs
          mountPath: "/nfs"
      volumes:
      - name: nfs
        nfs:
          server: 10.10.10.39
          path: "/home/xuerq/nfs"
      nodeName: **********
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: tensorflow-worker0
    role: service
  name: tensorflow-wk-service0
spec:
  ports:
    - port: 2222
      targetPort: 2222
  selector:
    name: tensorflow-worker0
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: tensorflow-worker1-rc
spec:
  replicas: 1
  selector:
    name: tensorflow-worker1
  template:
    metadata:
      labels:
        name: tensorflow-worker1
        role: worker
    spec:
      containers:
      - name: worker
        image: harbor.ail.unisound.com/xuerq/tensorflow:0.11.0
        ports:
        - containerPort: 2222
        env:
        - name: PS_KEY
          valueFrom:
            configMapKeyRef:
              name: tensorflow-cluster-config
              key: ps
        - name: WORKER_KEY
          valueFrom:
            configMapKeyRef:
              name: tensorflow-cluster-config
              key: worker
        command: ["/bin/sh", "-c"]
        args: ["cp -r /nfs/deep_recommend_system-master ./;\
               cd deep_recommend_system-master/distributed/;\
               python cancer_classifier.py \
                   --ps_hosts=$(PS_KEY) \
                   --worker_hosts=$(WORKER_KEY) \
                   --job_name=worker \
                   --task_index=1 1>log 2>errlog
               "]
        volumeMounts:
        - name: nfs
          mountPath: "/nfs"
      volumes:
      - name: nfs
        nfs:
          server: 10.10.10.39
          path: "/home/xuerq/nfs"
      nodeName: **********
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: tensorflow-worker1
    role: service
  name: tensorflow-wk-service1
spec:
  ports:
    - port: 2222
      targetPort: 2222
  selector:
    name: tensorflow-worker1
```
## To be continued
## References
* http://www.cnblogs.com/xuxinkun/p/5983633.html
* http://kubernetes.io/docs/user-guide/getting-into-containers/

