# Nginx-Ingress

[toc]

## 1.创建名称空间

```shell
kubectl create namespaces ingress-test
```
之后查看名称空间
```shell
kubectl get ns
```
看到名称空间已经建立

## 2.创建service

创建一个在前面名称空间的service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-test-service
  namespace: ingress-test
spec:
  type: NodePort
  ports:
  - name: http-port
    nodePort: 30010
    port: 80
    targetPort: 80
  selector:
    app: ingress-test-pod
```

查看service的具体信息

```shell
kubectl get svc -n ingress-test

NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
ingress-test-service   NodePort   10.111.123.115   <none>        80:30010/TCP   3m28s
```

至此，service创建完毕

## 3.创建pod或者dep

这边为方便演示，直接创建3个pod,此时，pod的标签要和前面创建的service的selector要对上,namespace也要对上。

```yaml
apiVersion: v1 
kind: Pod
metadata: 
  name: myfirst-pod1
  namespace: ingress-test
  labels:
    app: ingress-test-pod
spec:
  containers:   
  - name: app
    image: docker.io/library/nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html/
      name: test-volume1
  volumes:
  - name: test-volume1
    hostPath:
      path: /data/nginx/html/
      type: "Directory"
---
apiVersion: v1 
kind: Pod
metadata: 
  name: myfirst-pod2
  namespace: ingress-test
  labels:
    app: ingress-test-pod
spec:
  containers:   
  - name: app
    image: docker.io/library/nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html/
      name: test-volume2
  volumes:
  - name: test-volume2
    hostPath:
      path: /data/nginx/html1/
      type: "Directory"
---
apiVersion: v1 
kind: Pod
metadata: 
  name: myfirst-pod3
  namespace: ingress-test
  labels:
    app: ingress-test-pod
spec:
  containers:   
  - name: app
    image: docker.io/library/nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html/
      name: test-volume3
  volumes:
  - name: test-volume3
    hostPath:
      path: /data/nginx/html2/
      type: "Directory"
```

之后我们查看创建三个pod的信息

```shell
kubectl get po -n ingress-test -o wide

NAME           READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
myfirst-pod1   1/1     Running   0          24s   10.244.1.44   node1   <none>           <none>
myfirst-pod2   1/1     Running   0          24s   10.244.1.46   node1   <none>           <none>
myfirst-pod3   1/1     Running   0          24s   10.244.1.45   node1   <none>           <none>
```

### 3.1 查看service和pod是否能够访问

首先我们直接访问三个pod的ip，因为是k8s内网，为了方便访问直接在集群内部curl

```shell
curl http://10.244.1.44
v1
curl http://10.244.1.45
v3
curl http://10.244.1.46
v2
```

访问pod正常，接下来访问service，访问前面得到的service 的cluster-ip

```shell
curl http://10.111.123.115
v2
curl http://10.111.123.115
v1
curl http://10.111.123.115
v3
```

因为使用的是nodeport模式service。所以会做一个物理机和service之间的端口映射，在之前创建的yaml中，我定义好了本机的30010端口转到service的80端口，所以curl本机ip的30010端口，也同样能访问

```shell
curl http://172.17.0.13:30010
v3
curl http://172.17.0.13:30010
v1
curl http://172.17.0.13:30010
v2
```

之所以三次访问结果不一样，是因为k8s集群，优先使用ipvs的rr。我们查看ipvs规则便可知。

```shell
TCP  172.17.0.13:30010 rr
  -> 10.244.1.44:80               Masq    1      0          0
  -> 10.244.1.45:80               Masq    1      0          0
  -> 10.244.1.46:80               Masq    1      0          0
  
TCP  10.111.123.115:80 rr
  -> 10.244.1.44:80               Masq    1      0          0
  -> 10.244.1.45:80               Masq    1      0          0
  -> 10.244.1.46:80               Masq    1      0          0
```

## 4.创建ingress

首先进入github ingress的yaml

https://github.com/kubernetes/ingress-nginx/tree/master/deploy/static

之后我们可以下载mandatory.yaml，也可自己逐个安装,为了方便，把ingress pod的网络模式改成hostnetwork模式，这样，ingress直接只用宿主机网络，而不用转发

```yaml
  
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      # wait up to five minutes for the drain of connections
      terminationGracePeriodSeconds: 300
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        kubernetes.io/os: linux
      hostNetwork: true
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.26.1
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
            - name: https
              containerPort: 443
              protocol: TCP
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          lifecycle:
            preStop:
              exec:
                command:
                  - /wait-shutdown
---
```

创建ingress的pod

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: ingress-test
spec:
  rules:
  - host: www.k8stest.com
    http:
      paths:
      - backend:
          serviceName: ingress-test-service
          servicePort: 80
```

### 4.1 测试是否能够通过域名访问

因为在云上部署且没有备案，所以只能通过服务器内网写本地host测试，这里，我们把 www.k8stest.com 这个域名解析到node1上去。

```shell
172.17.0.9  www.k8stest.com
```

接下来，我们curl这个域名就可以访问到后端了，

```shell
curl http://www.k8stest.com
v2
curl http://www.k8stest.com
v1
curl http://www.k8stest.com
v3
```

## 5.四层LB转发到ingress

我们使用腾讯云自带的LB服务，创建一个四层LB

![image-20191213162923630](/Users/zhengjiayang/Library/Application Support/typora-user-images/image-20191213162923630.png)

此LB的vip地址为，172.17.0.11，所以，我们把www.k8stest.com这个域名解析到这个LB上

```shell
172.17.0.11  www.k8stest.com
```

我们继续curl这个域名，成功转发，

```shell
curl http://www.k8stest.com
v1
curl http://www.k8stest.com
v3
curl http://www.k8stest.com
v2
```

## 6. 流程图总结

![image-20191213165827700](/Users/zhengjiayang/Library/Application Support/typora-user-images/image-20191213165827700.png)



