---
title: "kubelet组件监控"
date: 2023-03-05T15:20:10-08:00
draft: false
categories: ["k8s", "sre", "监控"]
tags: ["sre", "k8s", "监控"]
---

### kubelet 监控实操

Kube-Proxy 的`/metrics`接口没有认证，相对比较容易，这一篇我们介绍一下 Kubelet，Kubelet 的监控相比 Kube-Proxy 增加了认证机制，相对更复杂一些。

## Kubelet 端口说明

如果你有多台 Node 节点，可以批量执行 `ss -tlnp|grep kubelet` 看一下，Kubelet 监听两个固定端口(我的环境，你的环境可能不同)，一个是 10248，一个是 10250，通过下面的命令可以知道，10248 是健康检查的端口：

```Shell
[root@dev01.nj ~]# ps aux|grep kubelet
root      163490  0.0  0.0  12136  1064 pts/1    S+   13:34   0:00 grep --color=auto kubelet
root      166673  3.2  1.0 3517060 81336 ?       Ssl  Aug16 4176:52 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --hostname-override=10.206.0.16 --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.6

[root@dev01.nj ~]# cat /var/lib/kubelet/config.yaml | grep 102
healthzPort: 10248

[root@dev01.nj ~]# curl localhost:10248/healthz
ok
```

我们再看一下 10250，10250 实际是 Kubelet 的默认端口，`/metrics` 接口就是在这个端口暴露的，我们请求一下：

```Shell
[root@dev01.nj ~]# curl localhost:10250/metrics
Client sent an HTTP request to an HTTPS server.

[root@dev01.nj ~]# curl https://localhost:10250/metrics
curl: (60) SSL certificate problem: self signed certificate in certificate chain
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
[root@dev01.nj ~]# curl -k https://localhost:10250/metrics
Unauthorized
```

`-k` 表示不校验 SSL 证书是否正确，最后的命令可以看到返回了 Unauthorized，表示认证失败，我们先来解决一下认证问题。认证是 Kubernetes 的一个知识点，这里先不展开（你需要 Google 一下了解基本常识），直接实操。

## 认证信息

下面的信息可以保存为 auth.yaml，创建了 ClusterRole、ServiceAccount、ClusterRoleBinding。

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: categraf-daemonset
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/metrics
      - nodes/stats
      - nodes/proxy
    verbs:
      - get
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: categraf-daemonset
  namespace: flashcat
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: categraf-daemonset
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: categraf-daemonset
subjects:
  - kind: ServiceAccount
    name: categraf-daemonset
    namespace: flashcat
```

ClusterRole 是个全局概念，不属于任一个 namespace，定义了很多权限点，都是读权限，监控嘛，读权限就可以了，ServiceAccount 则是 namespace 颗粒度的一个概念，这里我们创建了一个名为 categraf-daemonset 的 ServiceAccount，然后绑定到 ClusterRole 上面，具备了各种查询权限。apply 一下即可：

```Shell
[work@dev01.nj yamls]$ kubectl apply -f auth.yaml
clusterrole.rbac.authorization.k8s.io/categraf-daemonset created
serviceaccount/categraf-daemonset created
clusterrolebinding.rbac.authorization.k8s.io/categraf-daemonset created

[work@dev01.nj yamls]$ kubectl get ClusterRole | grep categraf-daemon
categraf-daemonset                                                     2022-11-14T03:53:54Z
[work@dev01.nj yamls]$ kubectl get sa -n flashcat
NAME                 SECRETS   AGE
categraf-daemonset   1         90m
default              1         4d23h
[work@dev01.nj yamls]$ kubectl get ClusterRoleBinding -n flashcat | grep categraf-daemon
categraf-daemonset ClusterRole/categraf-daemonset 91m
```

## 测试权限

上面的命令行输出可以看出来，我们已经成功创建了 ServiceAccount，把 ServiceAccount 的内容打印出来看一下：

```Shell
[root@dev01.nj qinxiaohui]# kubectl get sa categraf-daemonset -n flashcat -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"categraf-daemonset","namespace":"flashcat"}}
  creationTimestamp: "2022-11-14T03:53:54Z"
  name: categraf-daemonset
  namespace: flashcat
  resourceVersion: "120570510"
  uid: 22f5a785-871c-4454-b82e-12bf104450a0
secrets:
- name: categraf-daemonset-token-7mccq
```

注意最后两行，这个 ServiceAccount 实际是关联了一个 Secret，我们再看看这个 Secret 的内容：

```Shell
[root@dev01.nj k8s]# kubectl get secret categraf-daemonset-token-7mccq -n flashcat -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeU1ERXdPVEF4TXpjek9Gb1hEVE15TURFd056QXhNemN6T0Zvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBS2F1Ck9wU3hHdXB0ZlNraW1zbmlONFVLWnp2b1p6akdoTks1eUVlZWFPcmptdXIwdTFVYlFHbTBRWlpMem8xVi9GV1gKVERBOUthcFRNVllyS2hBQjNCVXdqdGhCaFp1NjJVQzg5TmRNSDVzNFdmMGtMNENYZWQ3V2g2R05Md0MyQ2xKRwp3Tmp1UkZRTndxMWhNWjY4MGlaT1hLZk1NbEt6bWY4aDJWZmthREdpVHk0VzZHWE5sRlRJSFFkVFBVMHVMY3dYCmc1cUVsMkd2cklmd05JSXBOV3ZoOEJvaFhyc1pOZVNlNHhGMVFqY0R2QVE4Q0xta2J2T011UGI5bGtwalBCMmsKV055RTVtVEZCZ2NCQ3dzSGhjUHhyN0E3cXJXMmtxbU1MbUJpc2dHZm9ieXFWZy90cTYzS1oxYlRvWjBIbXhicQp6TkpOZUJpbm9jbi8xblJBK3NrQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZLVkxrbVQ5RTNwTmp3aThsck5UdXVtRm1MWHNNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSm5QR24rR012S1ZadFVtZVc2bQoxanY2SmYvNlBFS2JzSHRkN2dINHdwREI3YW9pQVBPeTE0bVlYL2d5WWgyZHdsRk9hTWllVS9vUFlmRDRUdGxGCkZMT08yVkdLVTJBSmFNYnVBekw4ZTlsTFREM0xLOGFJUm1FWFBhQkR2V3VUYXZuSTZCWDhiNUs4SndraVd0R24KUFh0ejZhOXZDK1BoaWZDR0phMkNxQWtJV0Nrc0lWenNJcWJ0dkEvb1pHK1dhMlduemFlMC9OUFl4QS8waldOMwpVcGtDWllFaUQ4VlUwenRIMmNRTFE4Z2Mrb21uc3ljaHNjaW5KN3JsZS9XbVFES3ZhVUxLL0xKVTU0Vm1DM2grCnZkaWZtQStlaFZVZnJaTWx6SEZRbWdzMVJGMU9VczNWWUd0REt5YW9uRkc0VFlKa1NvM0IvRlZOQ0ZtcnNHUTYKZWV3PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  namespace: Zmxhc2hjYXQ=
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklqRTJZVTlNU2pObFFVbEhlbmhDV1dsVmFIcEVTRlZVWVdoZlZVaDZSbmd6TUZGZlVWUjJUR0pzVUVraWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUptYkdGemFHTmhkQ0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVmpjbVYwTG01aGJXVWlPaUpqWVhSbFozSmhaaTFrWVdWdGIyNXpaWFF0ZEc5clpXNHROMjFqWTNFaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNXVZVzFsSWpvaVkyRjBaV2R5WVdZdFpHRmxiVzl1YzJWMElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVkV2xrSWpvaU1qSm1OV0UzT0RVdE9EY3hZeTAwTkRVMExXSTRNbVV0TVRKaVpqRXdORFExTUdFd0lpd2ljM1ZpSWpvaWMzbHpkR1Z0T25ObGNuWnBZMlZoWTJOdmRXNTBPbVpzWVhOb1kyRjBPbU5oZEdWbmNtRm1MV1JoWlcxdmJuTmxkQ0o5Lm03czJ2Z1JuZDJzMDJOUkVwakdpc0JYLVBiQjBiRjdTRUFqb2RjSk9KLWh6YWhzZU5FSDFjNGNDbXotMDN5Z1Rkal9NT1VKaWpCalRmaW9FSWpGZHRCS0hEMnNjNXlkbDIwbjU4VTBSVXVDemRYQl9tY0J1WDlWWFM2bE5zYVAxSXNMSGdscV9Sbm5XcDZaNmlCaWp6SU05QUNuckY3MGYtd1FZTkVLc2MzdGhubmhSX3E5MkdkZnhmdGU2NmhTRGthdGhPVFRuNmJ3ZnZMYVMxV1JCdEZ4WUlwdkJmVXpkQ1FBNVhRYVNPck00RFluTE5uVzAxWDNqUGVZSW5ka3NaQ256cmV6Tnp2OEt5VFRTSlJ2VHVKMlZOU2lHaDhxTEgyZ3IzenhtQm5Qb1d0czdYeFhBTkJadG0yd0E2OE5FXzY0SlVYS0tfTlhfYmxBbFViakwtUQ==
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: categraf-daemonset
    kubernetes.io/service-account.uid: 22f5a785-871c-4454-b82e-12bf104450a0
  creationTimestamp: "2022-11-14T03:53:54Z"
  name: categraf-daemonset-token-7mccq
  namespace: flashcat
  resourceVersion: "120570509"
  uid: 0a228da5-6e60-4b22-beff-65cc56683e41
type: kubernetes.io/service-account-token
```

我们把这个 token 字段拿到，然后 base64 转码一下，作为 Bearer Token 来请求测试一下：

```Shell
[root@dev01.nj k8s]# token=`kubectl get secret categraf-daemonset-token-7mccq -n flashcat -o jsonpath={.data.token} | base64 -d`
[root@dev01.nj k8s]# curl -s -k -H "Authorization: Bearer $token" https://localhost:10250/metrics > aaaa
[root@dev01.nj k8s]# head -n 5 aaaa
# HELP apiserver_audit_event_total [ALPHA] Counter of audit events generated and sent to the audit backend.
# TYPE apiserver_audit_event_total counter
apiserver_audit_event_total 0
# HELP apiserver_audit_requests_rejected_total [ALPHA] Counter of apiserver requests rejected due to an error in audit logging backend.
# TYPE apiserver_audit_requests_rejected_total counter
apiserver_audit_requests_rejected_total 0
```

通了！

这就说明我们创建的 ServiceAccount 是好使的，后面我们把 Categraf 作为采集器搞成 Daemonset，再为 Categraf 这个 Daemonset 指定 ServiceAccountName，Kubernetes 就会自动把 Token 的内容挂到 Daemonset 的目录里，下面开始实操。

## 升级 Daemonset

上一篇咱们为 Kube-Proxy 的采集准备了 Daemonset，咱们就继续修改这个 Daemonset，让这个 Daemonset 不但可以采集 Kube-Proxy，也可以采集 Kubelet，先给 Categraf 准备一下相关的配置，可以把下面的内容保存为 categraf-configmap-v2.yaml

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
    [[instances]]
    urls = ["https://127.0.0.1:10250/metrics"]
    bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
    use_tls = true
    insecure_skip_verify = true
    labels = { job="kubelet" }
    [[instances]]
    urls = ["https://127.0.0.1:10250/metrics/cadvisor"]
    bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
    use_tls = true
    insecure_skip_verify = true
    labels = { job="cadvisor" }
```

apply 一下，让新的配置生效：

```Shell
[work@dev01.nj yamls]$ kubectl apply -f categraf-configmap-v2.yaml -n flashcat
configmap/categraf-config unchanged
configmap/categraf-input-prometheus configured
```

Categraf 的 Daemonset 需要把 ServiceAccountName 给绑定上，上一讲咱们用的 yaml 命名为：categraf-daemonset-v1.yaml ，咱们升级一下这个文件到 categraf-daemonset-v2.yaml 版本，内容如下：

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
      serviceAccountName: categraf-daemonset
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

这里跟 v1 版本相比，唯一的变化，就是加了 `serviceAccountName: categraf-daemonset` 这个配置，把原来的 Daemonset 删掉，从新创建一下：

```Shell
[work@dev01.nj yamls]$ kubectl delete ds categraf-daemonset -n flashcat
daemonset.apps "categraf-daemonset" deleted

[work@dev01.nj yamls]$ kubectl apply -f categraf-daemonset-v2.yaml -n flashcat
daemonset.apps/categraf-daemonset created

# waiting...
[work@dev01.nj yamls]$ kubectl get pods -n flashcat
NAME                       READY   STATUS    RESTARTS   AGE
categraf-daemonset-d8jt8   1/1     Running   0          37s
categraf-daemonset-fpx8v   1/1     Running   0          43s
categraf-daemonset-mp468   1/1     Running   0          32s
```
