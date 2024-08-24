---
title: "kube-proxy组件监控"
date: 2023-01-16T16:20:10-08:00
draft: false
categories: ["k8s", "sre", "监控"]
tags: ["sre", "k8s", "监控"]
---

### kube-proxy 监控实操

Kube-Proxy 是在所有工作负载节点上的，默认暴露两个端口，10249 用于暴露监控指标，在 `/metrics` 接口吐出 Prometheus 协议的监控数据：

```Shell
[root@dev01.nj lib]# curl -s http://localhost:10249/metrics | head -n 10
# HELP apiserver_audit_event_total [ALPHA] Counter of audit events generated and sent to the audit backend.
# TYPE apiserver_audit_event_total counter
apiserver_audit_event_total 0
# HELP apiserver_audit_requests_rejected_total [ALPHA] Counter of apiserver requests rejected due to an error in audit logging backend.
# TYPE apiserver_audit_requests_rejected_total counter
apiserver_audit_requests_rejected_total 0
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 2.5307e-05
go_gc_duration_seconds{quantile="0.25"} 2.8884e-05
```

10256 端口作为健康检查的端口，使用 `/healthz` 接口做健康检查，请求之后返回两个时间信息：

```Shell
[root@dev01.nj lib]# curl -s http://localhost:10256/healthz | jq .
{
  "lastUpdated": "2022-11-09 13:14:35.621317865 +0800 CST m=+4802354.950616250",
  "currentTime": "2022-11-09 13:14:35.621317865 +0800 CST m=+4802354.950616250"
}
```

所以，我们只要从 `http://localhost:10249/metrics` 采集监控数据即可。既然是 Prometheus 协议的数据，使用 Categraf 的 input.prometheus 来搞定即可。

## Categraf prometheus 插件

配置文件在 `conf/input.prometheus/prometheus.toml`，把 Kube-Proxy 的地址配置进来即可：

```toml
interval = 15
[[instances]]
urls = [
     "http://localhost:10249/metrics"
]
labels = { job="kube-proxy" }
```

urls 字段配置 endpoint 列表，即所有提供 metrics 数据的接口，我们使用下面的命令做个测试：

```Shell
[work@dev01.nj categraf]$ ./categraf --test --inputs prometheus | grep kubeproxy_sync_proxy_rules
2022/11/09 13:30:17 main.go:110: I! runner.binarydir: /home/work/go/src/categraf
2022/11/09 13:30:17 main.go:111: I! runner.hostname: dev01.nj
2022/11/09 13:30:17 main.go:112: I! runner.fd_limits: (soft=655360, hard=655360)
2022/11/09 13:30:17 main.go:113: I! runner.vm_limits: (soft=unlimited, hard=unlimited)
2022/11/09 13:30:17 config.go:33: I! tracing disabled
2022/11/09 13:30:17 provider.go:63: I! use input provider: [local]
2022/11/09 13:30:17 agent.go:87: I! agent starting
2022/11/09 13:30:17 metrics_agent.go:93: I! input: local.prometheus started
2022/11/09 13:30:17 prometheus_scrape.go:14: I! prometheus scraping disabled!
2022/11/09 13:30:17 agent.go:98: I! agent started
13:30:17 kubeproxy_sync_proxy_rules_endpoint_changes_pending agent_hostname=dev01.nj instance=http://localhost:10249/metrics 0
13:30:17 kubeproxy_sync_proxy_rules_duration_seconds_count agent_hostname=dev01.nj instance=http://localhost:10249/metrics 319786
13:30:17 kubeproxy_sync_proxy_rules_duration_seconds_sum agent_hostname=dev01.nj instance=http://localhost:10249/metrics 17652.749911909214
13:30:17 kubeproxy_sync_proxy_rules_duration_seconds_bucket agent_hostname=dev01.nj instance=http://localhost:10249/metrics le=+Inf 319786
13:30:17 kubeproxy_sync_proxy_rules_duration_seconds_bucket agent_hostname=dev01.nj instance=http://localhost:10249/metrics le=0.001 0
13:30:17 kubeproxy_sync_proxy_rules_duration_seconds_bucket agent_hostname=dev01.nj instance=http://localhost:10249/metrics le=0.002 0
13:30:17 kubeproxy_sync_proxy_rules_duration_seconds_bucket agent_hostname=dev01.nj instance=http://localhost:10249/metrics le=0.004 0
13:30:17 kubeproxy_sync_proxy_rules_duration_seconds_bucket agent_hostname=dev01.nj instance=http://localhost:10249/metrics le=0.008 0

```

Kube-Proxy 在 Kubernetes 架构中，负责从 APIServer 同步规则，然后修改 iptables 或 ipvs 配置，同步规则相关的指标就非常关键了，这里我就 grep 了这些指标作为样例。

通过 `--test` 看到输出了，就说明正常采集到数据了，你有几个工作负载节点，就分别去修改 Categraf 的配置即可。当然，这样做非常直观，只是略麻烦，如果未来扩容新的 Node 节点，也要去修改 Categraf 的采集配置，把 Kube-Proxy 这个 `/metrics` 地址给加上，如果你是用脚本批量跑的，倒是还可以，如果是手工部署就略麻烦。我们可以把 Categraf 采集器做成 Daemonset，这样就不用担心扩容的问题了，Daemonset 会被自动调度到所有 Node 节点。

## Categraf 作为 Daemonset 部署

Categraf 作为 Daemonset 运行，首先要创建一个 namespace，然后相关的 ConfigMap、Daemonset 等都归属这个 namespace。只是监控 Kube-Proxy 的话，Categraf 的配置就只需要主配置 config.toml 和 prometheus.toml，下面我们就实操演示一下。

### 创建 namespace

```Shell
[work@dev01.nj categraf]$ kubectl create namespace flashcat
namespace/flashcat created

[work@dev01.nj categraf]$ kubectl get ns | grep flashcat
flashcat                                 Active   29s
```

### 创建 ConfigMap

ConfigMap 是用于放置 config.toml 和 prometheus.toml 的内容，我把 yaml 文件也给你准备好了，请保存为 categraf-configmap-v1.yaml

```yaml
---
kind: ConfigMap
metadata:
  name: categraf-config
apiVersion: v1
data:
  config.toml: |
    [global]
    hostname = "$HOSTNAME"
    interval = 15
    providers = ["local"]
    [writer_opt]
    batch = 2000
    chan_size = 10000
    [[writers]]
    url = "http://10.206.0.16:19000/prometheus/v1/write"
    timeout = 5000
    dial_timeout = 2500
    max_idle_conns_per_host = 100
---
kind: ConfigMap
metadata:
  name: categraf-input-prometheus
apiVersion: v1
data:
  prometheus.toml: |
    [[instances]]
    urls = ["http://127.0.0.1:10249/metrics"]
    labels = { job="kube-proxy" }
```

上面的 `10.206.0.16:19000` 只是举个例子，请改成你自己的 n9e-server 的地址。当然，如果不想把监控数据推给 Nightingale 也 OK，写成其他的时序库(支持 remote write 协议的接口)也可以。`hostname = "$HOSTNAME"` 这个配置用了 `$` 符号，后面创建 Daemonset 的时候会把 HOSTNAME 这个环境变量注入，让 Categraf 自动拿到。

下面我们把 ConfigMap 创建出来：

```Shell
[work@dev01.nj yamls]$ kubectl apply -f categraf-configmap-v1.yaml -n flashcat
configmap/categraf-config created
configmap/categraf-input-prometheus created

[work@dev01.nj yamls]$ kubectl get configmap -n flashcat
NAME                        DATA   AGE
categraf-config             1      19s
categraf-input-prometheus   1      19s
kube-root-ca.crt            1      22m
```

### 创建 Daemonset

配置文件准备好了，开始创建 Daemonset，注意把 HOSTNAME 环境变量注入进去，yaml 文件如下，你可以保存为 categraf-daemonset-v1.yaml：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: categraf-daemonset
  name: categraf-daemonset
spec:
  selector:
    matchLabels:
      app: categraf-daemonset
  template:
    metadata:
      labels:
        app: categraf-daemonset
    spec:
      containers:
        - env:
            - name: TZ
              value: Asia/Shanghai
            - name: HOSTNAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: spec.nodeName
            - name: HOSTIP
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: status.hostIP
          image: flashcatcloud/categraf:v0.2.18
          imagePullPolicy: IfNotPresent
          name: categraf
          volumeMounts:
            - mountPath: /etc/categraf/conf
              name: categraf-config
            - mountPath: /etc/categraf/conf/input.prometheus
              name: categraf-input-prometheus
      hostNetwork: true
      restartPolicy: Always
      tolerations:
        - effect: NoSchedule
          operator: Exists
      volumes:
        - configMap:
            name: categraf-config
          name: categraf-config
        - configMap:
            name: categraf-input-prometheus
          name: categraf-input-prometheus
```

apply 一下这个 Daemonset 文件：

```Shell
[work@dev01.nj yamls]$ kubectl apply -f categraf-daemonset-v1.yaml -n flashcat
daemonset.apps/categraf-daemonset created

[work@dev01.nj yamls]$ kubectl get ds -o wide -n flashcat
NAME                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE     CONTAINERS   IMAGES                           SELECTOR
categraf-daemonset   6         6         6       6            6           <none>          2m20s   categraf     flashcatcloud/categraf:v0.2.17   app=categraf-daemonset

[work@dev01.nj yamls]$ kubectl get pods -o wide -n flashcat
NAME                       READY   STATUS    RESTARTS   AGE     IP            NODE          NOMINATED NODE   READINESS GATES
categraf-daemonset-4qlt9   1/1     Running   0          2m10s   10.206.0.7    10.206.0.7    <none>           <none>
categraf-daemonset-s9bk2   1/1     Running   0          2m10s   10.206.0.11   10.206.0.11   <none>           <none>
```
