# 使用kubernetes搭建开源博客平台solo

[toc]

## 1.机器配置

```shell
master:192.168.122.109
node1:192.168.122.110 （假设为cpu密集型）
node2:192.168.122.111 (假设为io密集型)
```

## 2.创建名称空间

```shell
kubectl create namespace solo-blog
```

## 3.安装本次所需要的dokcer镜像

```shell
docker pull b3log/solo
docker pull mysql:5.6
```

## 4. 给节点打标签用以pod调度

因为我们假设110是cpu密集型，111是io密集型，所以打上不同的标签

```shell
kubectl label nodes node1 type=cpu
kubectl label nodes node2 type=io
```

## 5.创建数据库相关

### 5.1 创建数据库的无头服务

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: solo-blog
  name: mysql-headless
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
```

### 5.2 创建数据库的statefulset

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-stateful
  namespace: solo-blog
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql-headless
  replicas: 1
  template:
    metadata:
      name: mysql
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql-containner
          image: mysql:5.6
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3306
              name: mysql-port
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: mysql-datapath
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
      volumes:
        - name: mysql-datapath
          hostPath:
              path: /data/mysql/
              type: "DirectoryOrCreate"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
                - key: type
                  operator: In
                  values:
                  - io
```

完成之后进入数据库建库

```shell
create database solo DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

### 5.3 验证无头服务是否被解析

我们使用dig命令，首先查看kube-system里service的dns

```shell
kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   46h
```

可以看到集群dns的地址为10.96.0.10，有了这个地址，我们使用dig命令

```shell
dig @10.96.0.10 mysql-headless.solo-blog.svc.cluster.local
...
;; ANSWER SECTION:
mysql-headless.solo-blog.svc.cluster.local. 5 IN A 10.244.2.6

;; Query time: 1 msec
;; SERVER: 10.96.0.10#53(10.96.0.10)
;; WHEN: Wed Dec 18 16:14:52 CST 2019
;; MSG SIZE  rcvd: 129
```

可以看到，mysql-headless这个我们定义的无头服务，直接解析到了10.244.2.6这个ip上，而这个ip，就是pod的ip

```shell
kubectl get po -n solo-blog -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   
mysql-stateful-0   1/1     Running   0          43m   10.244.2.6   node2   <none>           
```

至此，数据库的工作已经结束，数据库集群内部的访问地址为:mysql-headless.solo-blog.svc.cluster.local

## 6.创建blog相关

相比sql，blog的部署就方便的多，因为blog是无状态。

### 6.1 创建blog的deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-deplyment
  namespace: solo-blog
spec:
  selector:
    matchLabels:
      app: blog
  replicas: 1
  template:
    metadata:
      labels:
        app: blog
    spec:
      containers:
      - name: blog-containner
        image: b3log/solo
        imagePullPolicy: IfNotPresent
        env:
        - name: RUNTIME_DB
          value: "mysql"
        - name: JDBC_USERNAME
          value: "root"
        - name: JDBC_PASSWORD
          value: "123456"
        - name: JDBC_DRIVER
          value: "com.mysql.cj.jdbc.Driver"
        - name: JDBC_URL
          value: "jdbc:mysql://mysql-headless.solo-blog.svc.cluster.local:3306/solo?useUnicode=yes&characterEncoding=UTF-8&useSSL=false&serverTimezone=UTC"
        ports:
        - name: web
          containerPort: 80
          hostPort: 80
          protocol: TCP
			  volumeMounts:
        - mountPath: /opt/solo/latke.properties
          name: conf
          subPath: latke.properties
      volumes:
      - name: conf
        hostPath:
          path: /solo/
			affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: type
                operator: In
                values:
                - cpu
```

其中之所以要把*latke.properties*给单独挂出来，是因为这个官方镜像没有把前端的配置文件写成环境变量，所以要单独挂到宿主机上，*latke.properties* 配置如下

```java
serverScheme=http
serverHost=www.soloblog.com
serverPort=80
runtimeMode=PRODUCTION
```

其中的配置就是最终的访问方式 http://serverHost:serverPort/

### 6.2 创建blog的service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: blog-service
  namespace: solo-blog
spec:
  type: NodePort
  selector:
    app: blog
  ports:
  - name:  blog
    port:  80
    targetPort:  8080
```

### 6.3 创建blog的ingress

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: solo-blog
spec:
  rules:
  - host: www.soloblog.com
    http:
      paths:
      - backend:
          serviceName: blog-service
          servicePort: 80                          
```

## 7.LB

​	部署LB非常简单，随便找个机器做一下四层端口转发即可，转发到ingress所在机器暴露的nginx端口即可。