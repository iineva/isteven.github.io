---
title: 修复 Kubernetes ingress-nginx Let's Encrypt OCSP Stapling 失败
date: 2020-05-21 15:08:40
tags: [k8s, kubernetes, ingress-nginx, Let's Encrypt, OCSP]
---

Kubernets 集群，使用 ingress-nginx 作为 ingress-controller，使用的是 Let's Encrypt 证书。
因为国内网络原因，`ocsp.int-x3.letsencrypt.org` 被DNS污染了，直接解析出来的主机无法通过国内网络直接访问。
`iOS` 系统回强制https的在线证书校验逻辑，导致会经常连接超时或初次连接速度很慢，影响用户体验。

解决办法有两个：
* 客户端重新实现TLS连接，跳过`在线证书校验`，找到个现成的库：<https://github.com/zymxxxs/YMHTTP>
* Web服务通过开启 `OCSP Stapling` 可以代替客户端做 `在线证书校验`，将验证结果直接返回客户端

通过客户端的方式解决，在用浏览器直接访问时还是会慢，这里主要介绍服务端解决方案：

`nginx` 是支持 `OCSP Stapling` 配置也很简单

```
server {
    ssl_stapling on;
}
```

集群使用的是 `ingress-nginx`，使用helm部署这样配置values (只展示了关键配置)

```yaml
controller:
    config:
      enable-ocsp: "true"
```

但是还有一个问题，服务端也需要访问 `ocsp.int-x3.letsencrypt.org` 来获取证书验证信息
在这里，找到个解决办法：<https://blog.csdn.net/qq_34581279/article/details/106147220>
只需要将 `ocsp.int-x3.letsencrypt.org` 解析到这个IP `23.32.3.72` 即可

在集群内实现这一点有如下几个方法：

## 1.修改集群coredns配置，添加解析配置

参考配置文档：<https://coredns.io/plugins/hosts/>
注意，配置中 `fallthrough` 必须添加，否则会导致集群网络异常

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        ...
        # 原配置省略，在后面追加即可
        hosts {
            23.32.3.72 ocsp.int-x3.letsencrypt.org
            fallthrough
        }
    }
```

## 2.修改 `ingress-nginx` 的Pod配置，添加dnsConfig

添加dnsConfig指向自部署DNS服务器，自部署的DNS服务器添加A记录 `ocsp.int-x3.letsencrypt.org` --> `23.32.3.72`

这里使用helm通过部署coredns部署DNS服务器 <https://hub.helm.sh/charts/stable/coredns> 我当前使用的版本：`1.10.1`
values配置如下:

```yaml
service:
  # 必须指定集群IP，可以自行修改，注意集群支持的service网段
  clusterIP: 10.96.88.88
servers:
- zones:
  - zone: .
  port: 53
  plugins:
  - name: errors
  # Serves a /health endpoint on :8080, required for livenessProbe
  - name: health
    configBlock: |-
      lameduck 5s
  # Serves a /ready endpoint on :8181, required for readinessProbe
  - name: ready
  # Required to query kubernetes API for data
  - name: kubernetes
    parameters: cluster.local in-addr.arpa ip6.arpa
    configBlock: |-
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
      ttl 30
  - name: forward
    parameters: . /etc/resolv.conf
  - name: cache
    parameters: 30
  - name: loop
  - name: reload
  - name: loadbalance
  # 此处为关键配置
  - name: hosts
    configBlock: |-
      23.32.3.72 ocsp.int-x3.letsencrypt.org
      fallthrough
```

`ingress-nginx`安装使用的是官方提供的 helm chart：<https://kubernetes.github.io/ingress-nginx/deploy/#using-helm>
values关键配置如下:

```yaml
controller:
    ...
    # 修改dns配置
    dnsConfig:
        nameservers:
        # 与上面DNS服务器指定的集群IP对应
        - 10.96.88.88
        searches:
        - ocsp.int-x3.letsencrypt.org
    ...
```


## 3.修改 `ingress-nginx` 的Pod配置，添加hostAliases（此方法无效，仅提供思路给大家测试）

添加hostAliases将域名重新指向
经测试此方法虽然可以让`ingress-nginx`容器内网络按预期走，但是nginx做OCSP时并没有成功按预期解析域名
需要先修改`ingress-nginx`的chart，添加设置hostAliases支持

```bash
# 拉去chart
helm pull ingress-nginx/ingress-nginx --version 2.1.0
```

配置文件较长仅列出关键修改，所有修改都是添加，参照修下面添加到对应位置即可：

```yaml
# values.yaml
controller:
  ...
  # Optionally customize the pod hostAliases.
  hostAliases: {}
  ...
```

```yaml
# templates/controller-deployment.yaml
spec:
  template:
    metadata:
    spec:
    ...
    {{- if .Values.controller.hostAliases }}
      hostAliases: {{ toYaml .Values.controller.hostAliases | nindent 8 }}
    {{- end }}
    ...
```

```yaml
# templates/controller-daemonset.yaml
spec:
  template:
    metadata:
    spec:
    ...
    {{- if .Values.controller.hostAliases }}
      hostAliases: {{ toYaml .Values.controller.hostAliases | nindent 8 }}
    {{- end }}
    ...
```

至此chart修改完成，修改部署时的 values

```yaml
controller:
    ...
    hostAliases:
        - ip: 23.32.3.72
          hostnames:
          - ocsp.int-x3.letsencrypt.org
    ...
```
