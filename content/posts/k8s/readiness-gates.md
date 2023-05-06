---
title: "使用 Readiness Gates 暂时摘除流量"
date: 2023-03-28T16:05:06+08:00
draft: false
tags:
  - k8s
---
## 背景

在 OnCall 值班时收到线上 HTTP 504 状态码的报警，排查发现是开发的同事在用 arthas 排查问题的时候导致了 Java 应用程序 FullGC，引发调用超时。由于我们公司的 Pod 的 readinessProbe 和 livenessProbe 用的是同一个脚本，那有什么办法在不修改原有逻辑的情况下，在排查问题的时候把 Pod 暂时从 Service 的 Endpoint 中摘除，这样就不会有生产流量进来，通过查找资料，发现 K8S 中的 Pod Readiness Gates 可以实现。

## Readiness Gates

K8S 从 1.11 版本开始引入了 Pod Ready++ 特性对 Readiness 探测机制进行扩展，在 1.14 版本时达到稳定版本，称为 Pod Readiness Gates，可以自定义的 ReadinessProbe 探测方式设置在 Pod 上，辅助设置 Pod 何时达到服务可用状态 Ready。

### 测试

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: test
spec:
  replicas: 2
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
        image: "nginx"
        ports:
        - containerPort: 80
          name: http
        resources:
          limits:
            cpu: 1
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 128Mi
      readinessGates:
      - conditionType: "www.example.com/feature-1"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: test
  labels:
    app: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
```

kubectl apply 后查看 Pod

```
Readiness Gates:
  Type                        Status
  www.example.com/feature-1   <none>
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   True
  PodScheduled      True
```

可以看到，该 Pod Ready  状态值为 False，有一条 Readiness Gates 状态值为 none。

<font color=#FF0000>注意：设置了 Readiness Gates 的容器启动后状态是 NotReady 的，要设置 status 中Readiness Gates的值为 True 才会 Ready。</font>

由于kubectl 的 patch 无法直接修改 status 的值，我们直接用 curl 测试
```shell
cat  ~/.kube/config |grep client-certificate-data | awk -F ' ' '{print $2}' |base64 -d > ./client-cert.pem
cat  ~/.kube/config |grep client-key-data | awk -F ' ' '{print $2}' |base64 -d > ./client-key.pem
APISERVER=`cat  ~/.kube/config |grep server | awk -F ' ' '{print $2}'`

curl --cert ./client-cert.pem --key ./client-key.pem -k ${APISERVER}/api/v1/namespaces/test/pods/nginx-65f47c4bf7-sn4n8/status -X PATCH -H "Content-Type: application/json-patch+json" -d '[{"op": "add", "path": "/status/conditions/-", "value": {"type": "www.example.com/feature-1", "status": "True", "lastProbeTime": null}}]'
```

设置值后 Pod 的状态变为 Ready，Service 的 Endpoint 中就可以看到这个容器的 ip。

```
Readiness Gates:
  Type                        Status
  www.example.com/feature-1   True
Conditions:
  Type                        Status
  www.example.com/feature-1   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
```

Pod 的 status 值如下：

```
    "status": {
        "conditions": [
            {
                "lastProbeTime": null,
                "lastTransitionTime": null,
                "status": "True",
                "type": "www.example.com/feature-1"
            },
            {
                "lastProbeTime": null,
                "lastTransitionTime": "2023-05-02T09:24:34Z",
                "status": "True",
                "type": "Initialized"
            },
            {
                "lastProbeTime": null,
                "lastTransitionTime": "2023-05-02T10:20:56Z",
                "status": "True",
                "type": "Ready"
            },
            {
                "lastProbeTime": null,
                "lastTransitionTime": "2023-05-02T09:24:41Z",
                "status": "True",
                "type": "ContainersReady"
            },
            {
                "lastProbeTime": null,
                "lastTransitionTime": "2023-05-02T09:24:34Z",
                "status": "True",
                "type": "PodScheduled"
            }
        ],
    ...
    }
```

再将值设为 False 或删除该 status 后，Pod 重新变为 NotReady。

```shell
#设置该 status 为 False
curl --cert ./client-cert.pem --key ./client-key.pem -k ${APISERVER}/api/v1/namespaces/test/pods/nginx-65f47c4bf7-sn4n8/status -X PATCH -H "Content-Type: application/json-patch+json" -d '[{"op": "replace", "path": "/status/conditions/0/status", "value": "False"}]'
#删除该 status
curl --cert ./client-cert.pem --key ./client-key.pem -k ${APISERVER}/api/v1/namespaces/test/pods/nginx-65f47c4bf7-sn4n8/status -X PATCH -H "Content-Type: application/json-patch+json" -d '[{"op": "remove", "path": "/status/conditions/0"}]'
```

### 其它应用场景
- 阶段更新
- 原地升级
- 依赖于其他组件提供服务

## 参考
- https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/