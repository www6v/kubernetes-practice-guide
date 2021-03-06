---
title: "应用高可用部署"
weight: 10
---

为了提高服务容错能力，我们通常会设置 `replicas` 给服务创建多个副本，但这并不意味着服务就实现高可用了，下面来介绍应用部署的最佳实践。

## 使用反亲和性避免单点故障

k8s 的设计就是假设节点是不可靠的，节点越多，发生软硬件故障导致节点不可用的几率就越高，所以我们通常需要给服务部署多个副本，根据实际情况调整 `replicas` 的值，如果值为 1 就必然存在单点故障，如果大于 1 但所有副本都调度到同一个节点，那还是有单点故障，所以我们不仅要有合理的副本数量，还需要让这些不同副本调度到不同的节点，打散开来避免单点故障，这个可以利用反亲和性来实现，示例:

``` yaml
affinity:
 podAntiAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
   - weight: 100
     labelSelector:
       matchExpressions:
       - key: k8s-app
         operator: In
         values:
         - kube-dns
     topologyKey: kubernetes.io/hostname
```

* `requiredDuringSchedulingIgnoredDuringExecution` 调度时必须满足该反亲和性条件，如果没有节点满足条件就不调度到任何节点 (Pending)。如果不用这种硬性条件可以使用 `preferredDuringSchedulingIgnoredDuringExecution` 来指示调度器尽量满足反亲和性条件，如果没有满足条件的也可以调度到某个节点。
* `labelSelector.matchExpressions` 写该服务对应 pod 中 labels 的 key 与 value。
* `topologyKey` 这里用 `kubernetes.io/hostname` 表示避免 pod 调度到同一节点，如果你有更高的要求，比如避免调度到同一个可用区，实现异地多活，可以用 `failure-domain.beta.kubernetes.io/zone`。通常不会去避免调度到同一个地域，因为一般同一个集群的节点都在一个地域，如果跨地域，即使用专线时延也会很大，所以 `topologyKey` 一般不至于用 `failure-domain.beta.kubernetes.io/region`。

## 使用 preStopHook 和 readinessProbe 保证服务平滑更新不中断

如果服务不做配置优化，默认情况下更新服务期间可能会导致部分流量异常，下面我们来分析并给出最佳实践。

### 服务更新场景

我们先看下服务更新有哪些场景:

* 手动调整服务的副本数量
* 手动删除 Pod 触发重新调度
* 驱逐节点 (主动或被动驱逐，Pod会先删除再在其它节点重建)
* 触发滚动更新 (比如修改镜像 tag 升级程序版本)
* HPA (HorizontalPodAutoscaler) 自动对服务进行水平伸缩
* VPA (VerticalPodAutoscaler) 自动对服务进行垂直伸缩

### 更新过程连接异常的原因

滚动更新时，Service 对应的 Pod 会被创建或销毁，Service 对应的 Endpoint 也会新增或移除相应的 Pod IP:Port，然后 kube-proxy 会根据 Service 的 Endpoint 里的 Pod IP:Port 列表更新节点上的转发规则，而这里 kube-proxy 更新节点转发规则的动作并不是那么及时，主要是由于 K8S 的设计理念，各个组件的逻辑是解耦的，各自使用 Controller 模式 listAndWatch 感兴趣的资源并做出相应的行为，所以从 Pod 创建或销毁到 Endpoint 更新再到节点上的转发规则更新，这个过程是异步的，所以会造成转发规则更新不及时，从而导致服务更新期间部分连接异常。

我们分别分析下 Pod 创建和销毁到规则更新期间的过程:

1. Pod 被创建，但启动速度没那么快，还没等到 Pod 完全启动就被 Endpoint Controller 加入到 Service 对应 Endpoint 的 Pod IP:Port 列表，然后 kube-proxy watch 到更新也同步更新了节点上的 Service 转发规则 (iptables/ipvs)，如果这个时候有请求过来就可能被转发到还没完全启动完全的 Pod，这时 Pod 还不能正常处理请求，就会导致连接被拒绝。
2. Pod 被销毁，但是从 Endpoint Controller watch 到变化并更新 Service 对应 Endpoint 再到 kube-proxy 更新节点转发规则这期间是异步的，有个时间差，Pod 可能已经完全被销毁了，但是转发规则还没来得及更新，就会造成新来的请求依旧还能被转发到已经被销毁的 Pod，导致连接被拒绝。

### 平滑更新最佳实践

* 针对第一种情况，可以给 Pod 里的 container 加 readinessProbe (就绪检查)，通常是容器完全启动后监听一个 HTTP 端口，kubelet 发就绪检查探测包，正常响应说明容器已经就绪，然后修改容器状态为 Ready，当 Pod 中所有容器都 Ready 了这个 Pod 才会被 Endpoint Controller 加进 Service 对应 Endpoint IP:Port 列表，然后 kube-proxy 再更新节点转发规则，更新完了即便立即有请求被转发到的新的 Pod 也能保证能够正常处理连接，避免了连接异常。
* 针对第二种情况，可以给 Pod 里的 container 加 preStop hook，让 Pod 真正销毁前先 sleep 等待一段时间，留点时间给 Endpoint controller 和 kube-proxy 更新 Endpoint 和转发规则，这段时间 Pod 处于 Terminating 状态，即便在转发规则更新完全之前有请求被转发到这个 Terminating 的 Pod，依然可以被正常处理，因为它还在 sleep，没有被真正销毁。

最佳实践 yaml 示例:

``` yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - name: nginx
        image: "nginx"
        ports:
        - name: http
          hostPort: 80
          containerPort: 80
          protocol: TCP
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
            httpHeaders:
            - name: X-Custom-Header
              value: Awesome
          initialDelaySeconds: 15
          timeoutSeconds: 1
        lifecycle:
          preStop:
            exec:
              command: ["/bin/bash", "-c", "sleep 30"]
```

## 使用 PodDisruptionBudget 避免驱逐导致服务不可用

驱逐节点是一种有损操作，驱逐的原理:

1. 封锁节点 (设为不可调度，避免新的 Pod 调度上来)。
2. 将该节点上的 Pod 删除。
3. ReplicaSet 控制器检测到 Pod 减少，会重新创建一个 Pod，调度到新的节点上。

这个过程是先删除，再创建，并非是滚动更新，因此更新过程中，如果一个服务的所有副本都在被驱逐的节点上，则可能导致该服务不可用。

我们再来下什么情况下驱逐会导致服务不可用:

1. 服务存在单点故障，所有副本都在同一个节点，驱逐该节点时，就可能造成服务不可用。
2. 服务在多个节点，但这些节点都被同时驱逐，所以这个服务的所有服务同时被删，也可能造成服务不可用。

针对第一点，我们可以 [使用反亲和性避免单点故障](#use-antiaffinity-to-avoid-single-points-of-failure)。

针对第二点，我们可以通过配置 PDB (PodDisruptionBudget) 来避免所有副本同时被删除，下面给出示例。

示例一 (保证驱逐时 zookeeper 至少有两个副本可用):

``` yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: zookeeper
```

示例二 (保证驱逐时 zookeeper 最多有一个副本不可用，相当于逐个删除并在其它节点重建):

``` yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: zookeeper
```

更多请参考官方文档: https://kubernetes.io/docs/tasks/run-application/configure-pdb/