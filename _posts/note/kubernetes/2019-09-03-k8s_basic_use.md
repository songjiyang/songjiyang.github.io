---
bg: 'kubernetes-tutorial.png'
layout: post
title:  "kubernetes基础使用"
crawlertitle: "kubernetes"
summary: ""
date:   2019-09-03 16:53:00 +0700
categories: posts
tags: 'kubernetes'
author: 宋天
---

kubernetes基础使用，创建一个Deployment,创建一个Service




### 使用k8s部署一个nginx


#### 创建一个Deployment
 - controllers/nginx-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

##### 必需字段
在想要创建的 Kubernetes 对象对应的 .yaml 文件中，需要配置如下的字段：
- apiVersion - 创建该对象所使用的 Kubernetes API 的版本
- kind - 想要创建的对象的类型
- metadata - 帮助识别对象唯一性的数据，包括一个 name 字符串、UID 和可选的 namespace
- spec 字段。对象 spec 的精确格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。

[Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#)参考能够帮助我们找到任何我们想创建的对象的 spec 格式。

#### 问题

1. Deployment的matchLabels必选能选到template的label,否则创建资源的时候会报错。==为什么==, 因为这个Deployment选择不到对应的pod，达不到要求的replica为3，所以会不停新建template的Pod。
2. 现在如何访问这个nginx
- `kubectl port-forward <podname> 80:80`，使用kubectl代理，这样只有拥有k8s集群权限才行，用不了k8s也访问不到
- 建一个Service，下面介绍

#### 创建一个Service

- service/nginx-service.yml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 31100
  type: NodePort
```

- spec.type分别有ClusterIP(default), NodePort, and LoadBalancer，
1. 默认的ClusterIP会给这个Service生成一个虚拟IP,在k8s集群中，我们可以通过这个ip,或者Service的name(借助的k8s的dns解析)来访问这个服务，但我们还是无法在外部访问到这个服务
2. NodePort会在k8s集群中的所有Node开启一个nodePort指定的端口或者随机端口，然后我们可以拿k8s集群中任意一个机器的IP加这个nodePort就可以访问这个服务，但这个任然不是最佳的方式，因为我们需要一个额外的文件来维护不同服务的不同端口，在后面可以借助ingress，和ingress controller边缘路由器来暴露端口，并对k8s内部的服务进行反向代理。
3. LoadBalancer借助云服务商提供的LoadBalancer软件，将服务暴露在云服务商的LB软件上，每个厂商的配置不太相同