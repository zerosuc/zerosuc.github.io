## 控制器

Kubernetes 控制器会监听资源的 `创建/更新/删除` 事件，并触发 `Reconcile` 调谐函数作为响应，整个调整过程被称作 `“Reconcile Loop”（调谐循环）` 或者 `“Sync Loop”（同步循环）`。Reconcile 是一个使用资源对象的命名空间和资源对象名称来调用的函数，使得资源对象的实际状态与 资源清单中定义的状态保持一致。调用完成后，Reconcile 会将资源对象的状态更新为当前实际状态。我们可以用下面的一段伪代码来表示这个过程：

```
for {
  desired := getDesiredState()  // 期望的状态
  current := getCurrentState()  // 当前实际状态
  if current == desired {  // 如果状态一致则什么都不做
    // nothing to do
  } else {  // 如果状态不一致则调整编排，到一致为止
    // change current to desired status
  }
}
```

这个编排模型就是 Kubernetes 项目中的一个通用编排模式，即：`控制循环（control loop）`。



## Headless Service

在我们学习 StatefulSet 对象之前，我们还必须了解一个新的概念：`Headless Service`。`Service` 其实在之前我们和大家提到过，Service 是应用服务的抽象，通过 Labels 为应用提供负载均衡和服务发现，每个 Service 都会自动分配一个 cluster IP 和 DNS 名，在集群内部我们可以通过该地址或者通过 FDQN 的形式来访问服务。比如，一个 Deployment 有 3 个 Pod，那么我就可以定义一个 Service，有如下两种方式来访问这个 Service：

- cluster IP 的方式，比如：当我访问 10.109.169.155 这个 Service 的 IP 地址时，10.109.169.155 其实就是一个 VIP，它会把请求转发到该 Service 所代理的 Endpoints 列表中的某一个 Pod 上。具体原理我们会在后面的 Service 章节中和大家深入了解。
- Service 的 DNS 方式，比如我们访问`“mysvc.mynamespace.svc.cluster.local”`这条 DNS 记录，就可以访问到 mynamespace 这个命名空间下面名为 mysvc 的 Service 所代理的某一个 Pod。

对于 DNS 这种方式实际上也有两种情况：

- 第一种就是普通的 Service，我们访问`“mysvc.mynamespace.svc.cluster.local”`的时候是通过集群中的 DNS 服务解析到的 mysvc 这个 Service 的 cluster IP 的
- 第二种情况就是`Headless Service`，对于这种情况，我们访问`“mysvc.mynamespace.svc.cluster.local”`的时候是直接解析到的 mysvc 代理的某一个具体的 Pod 的 IP 地址，中间少了 cluster IP 的转发，这就是二者的最大区别，Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 的记录方式解析到后面的 Pod 的 IP 地址。

实际上 `Headless Service` 在定义上和普通的 Service 几乎一致, 只是他的 `clusterIP=None`，所以，这个 Service 被创建后并不会被分配一个 cluster IP，而是会以 DNS 记录的方式暴露出它所代理的 Pod，而且还有一个非常重要的特性，对于 `Headless Service` 所代理的所有 Pod 的 IP 地址都会绑定一个如下所示的 DNS 记录：

```
<pod-name>.<svc-name>.<namespace>.svc.cluster.local

/ # nslookup nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.244.1.175 web-1.nginx.default.svc.cluster.local
Address 2: 10.244.4.83 web-0.nginx.default.svc.cluster.local
/ # ping nginx
PING nginx (10.244.1.175): 56 data bytes
64 bytes from 10.244.1.175: seq=0 ttl=62 time=1.076 ms
64 bytes from 10.244.1.175: seq=1 ttl=62 time=1.029 ms
64 bytes from 10.244.1.175: seq=2 ttl=62 time=1.075 ms
```

我们直接解析 `Headless Service` 的名称，可以看到得到的是两个 Pod 的解析记录，但实际上如果我们通过`nginx`这个 DNS 去访问我们的服务的话，并不会随机或者轮询背后的两个 Pod，而是访问到一个固定的 Pod，所以不能代替普通的 Service。如果分别解析对应的 Pod 呢？

```
$ / # nslookup web-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.4.83 web-0.nginx.default.svc.cluster.local
/ # nslookup web-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.1.175 web-1.nginx.default.svc.cluster.local
```

可以看到解析 `web-0.nginx` 的时候解析到了 `web-0` 这个 Pod 的 IP，`web-1.nginx` 解析到了 `web-1` 这个 Pod 的 IP，而且这个 DNS 地址还是稳定的，因为 Pod 名称就是固定的。