# metric-server
metric-server-v0.3.6 tutorial


**Metrics-server-v0.3.6安装 **

https://github.com/kubernetes-sigs/metrics-server/releases/

**1、Metrics-server简介**

```go
Metrics Server 是 Kubernetes 内置自动缩放管道的可扩展、高效的容器资源指标来源。
Metrics Server 从 Kubelets 收集资源指标，并通过 Metrics API 将它们暴露在 Kubernetes apiserver 中，以供 HPA(Horizontal Pod Autoscaler) 和 VPA(Vertical Pod Autoscaler) 使用。
Metrics API 也可以通过 kubectl top 访问，从而更容易调试自动缩放管道。
本次安装演示也是通过kubectl top。 
```

查看node的负载 cpu利用率  内存占用率

```shell
[root@vms51 ~]#  kubectl top nodes -n kube-system
W0821 17:44:06.814710   59539 top_node.go:119] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
error: Metrics API not available
[root@vms51 ~]#
##报错是因为未安装Metrics-server 监控软件，软件安装后即可看到监控结果
```

k8s演示环境状态（使用kubeadm 安装k8s）

1master和2 works状态均已经Ready

```she
[root@vms51 ~]# kubectl get nodes
NAME            STATUS   ROLES                  AGE    VERSION
vms51.rhce.cc   Ready    control-plane,master   127m   v1.21.0
vms52.rhce.cc   Ready    <none>                 125m   v1.21.0
vms53.rhce.cc   Ready    <none>                 124m   v1.21.0
[root@vms51 ~]#
```

master节点上查看默认namespace和全部namespace内的pod状态

```she
[root@vms51 ~]# kubectl get pods -n kube-system  

[root@vms51 ~]# kubectl get pods  -n kube-system  -owide

[root@vms51 ~]# kubectl get pod -owide --all-namespaces
```

**2、Metrics-server的安装**

GitHub下载v0.3.6 的components.yaml

```shek
[root@vms51 ~]# wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

查看安装metrics-server-v0.3.6所需要的docker镜像

```shel
[root@vms51 ~]# grep image components.yaml
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
        image: k8s.gcr.io/metrics-server-amd64:v0.3.6
        imagePullPolicy: IfNotPresent
[root@vms51 ~]#
```

下载k8s.gcr.io/metrics-server-amd64:v0.3.6，发现无法下载k8s.gcr.io的镜像

```shel
[root@vms51 ~]# docker pull k8s.gcr.io/metrics-server-amd64:v0.3.6
Error response from daemon: Get "https://k8s.gcr.io/v2/": context deadline exceeded
```

上传metrics-img.tar到每个节点并docker load

```sh
[root@localhost ~]# md5sum metrics-img.tar
55e1a83dcf124da59b68365b262e051f  metrics-img.tar
链接：https://pan.baidu.com/s/1E_23nPlZznIN8AAZQvIZww 提取码：h9hn 
```

```shel 
[root@vms51 ~]# docker load -i metrics-img.tar
932da5156413: Loading layer [==================================================>]  3.062MB/3.062MB
7bf3709d22bb: Loading layer [==================================================>]  38.13MB/38.13MB
Loaded image: k8s.gcr.io/metrics-server-amd64:v0.3.6
[root@vms51 ~]# docker images |grep metrics-server
k8s.gcr.io/metrics-server-amd64                                   v0.3.6     9dd718864ce6   22 months ago   39.9MB
[root@vms51 ~]#
```

```shel
[root@vms52 ~]# docker load -i metrics-img.tar
932da5156413: Loading layer [==================================================>]  3.062MB/3.062MB
7bf3709d22bb: Loading layer [==================================================>]  38.13MB/38.13MB
Loaded image: k8s.gcr.io/metrics-server-amd64:v0.3.6
[root@vms51 ~]# docker images |grep metrics-server
k8s.gcr.io/metrics-server-amd64                                   v0.3.6     9dd718864ce6   22 months ago   39.9MB
[root@vms52 ~]#
```

```shell
[root@vms53 ~]# docker load -i metrics-img.tar
932da5156413: Loading layer [==================================================>]  3.062MB/3.062MB
7bf3709d22bb: Loading layer [==================================================>]  38.13MB/38.13MB
Loaded image: k8s.gcr.io/metrics-server-amd64:v0.3.6
[root@vms53 ~]# docker images |grep metrics-server
k8s.gcr.io/metrics-server-amd64                                   v0.3.6     9dd718864ce6   22 months ago   39.9MB
```

**问：**为什么非得安装metrics-server-amd64:v0.3.6？

**答：**网络原因无法下载到最新的metrics-server，我手里只有metrics-server-amd64:v0.3.6的镜像，并通过本地docker load;

如果能网络条件允许，能下载到各个版本的metrics-server镜像，并配相同版本的components.yaml，都是可以安装的。



**1）、**Master节点上为Metrics-server创建一个namespace,起名叫kubernetes-infra

```shell
[root@vms51 ~]# kubectl config set-context --namespace=kubernetes-infra --current
```

**2）、**vi编辑components.yaml，将系统默认kube-system namespace修改为kubernetes-infra

```she
[root@vms51 ~]#  vi components.yaml
:1,$s/kube-system/kubernetes-infra/
```

 **3）、**kubectl create创建Metrics-server pod

```shell
[root@vms51 ~]# kubectl create -f components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

**4）、**再开一个终端使用watch命令查看Metrics-server pod的创建过程

```shel
[root@vms51 ~]# watch 'kubectl get pods -n kubernetes-infra'
Every 2.0s: kubectl get pods -n kubernetes-infra                                                                                                               Sat Aug 21 21:20:03 2021

NAME                             READY   STATUS    RESTARTS   AGE
metrics-server-8bbfb4bdb-5l7mk   1/1     Running   0          28m

##备注
[root@vms51 ~]# kubectl get pod -owide --all-namespaces
```

**5）、**使用kubectl top node验证 查看Metrics-server是否运行正常，

```shell
[root@vms51 ~]# kubectl top node
W0821 21:26:00.105720   53095 top_node.go:119] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flag
error: Metrics API not available
[root@vms51 ~]#
```

**6）、**kubectl logs -f 查看log排障

```sh
[root@vms51 ~]# kubectl logs -f metrics-server-5f4b6b9889-hwxfv
I0822 06:28:12.610330       1 serving.go:312] Generated self-signed cert (/tmp/apiserver.crt, /tmp/apiserver.key)
I0822 06:28:13.888333       1 secure_serving.go:116] Serving securely on [::]:4443
E0822 06:29:02.705049       1 reststorage.go:135] unable to fetch node metrics for node "vms51.rhce.cc": no metrics known for node
E0822 06:29:02.705135       1 reststorage.go:135] unable to fetch node metrics for node "vms52.rhce.cc": no metrics known for node
E0822 06:29:02.705152       1 reststorage.go:135] unable to fetch node metrics for node "vms53.rhce.cc": no metrics known for node

```

**7）、** kubectl edit 修改metrics-server配置文件，添加

```she
[root@vms51 ~]# kubectl edit deployments.apps metrics-server
spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-insecure-tls                        ##添加--kubelet-insecure
        - --kubelet-preferred-address-types=InternalIP  ##添加--kubelet-preferred-address-types

–kubelet-preferred-address-types优先使用 InternalIP 来访问 kubelet，这样可以避免节点名称没有 DNS 解析记录时，通过节点名称调用节点 kubelet API 失败的情况
–kubelet-insecure-tls不验证客户端证书

```

使用watch 'kubectl top node' 等2-3分钟，即可看到metrics-server正常监控

```she
Every 2.0s: kubectl top node                                                                                                                                   Sat Aug 21 21:33:04 2021W0821 21:33:04.814094   62302 top_node.go:119] Using json format to get metrics. Next release will switch to protocol-buffers, switch early by passing --use-protocol-buffers flagNAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%vms51.rhce.cc   314m         15%    2091Mi          54%vms52.rhce.cc   134m         6%     1376Mi          35%vms53.rhce.cc   158m         7%     1415Mi          36%
```

删除pod使用kubectl delete删除components.yaml pod将自动删除

```shel
[root@vms51 ~]# kubectl delete -f components.yaml
```

