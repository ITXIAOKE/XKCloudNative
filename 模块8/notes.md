0. deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpserver
  labels:
    app: httpserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpserver
  template:
    metadata:
      labels:
        app: httpserver
    spec:
      imagePullSecrets:
      - name: cloudnative
      containers:
      - name: httpserver
        image: fmeng.azurecr.io/httpserver:1.0
        ports:
        - containerPort: 8080
```

## 1.优雅启动

### probe 探针

https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

#### readiness Probe

一个服务刚启动，可能会有一堆东西要加载，比如需要load大量数据等等，此时程序启动了，但是并未准备好处理外部请求

```yaml
  readinessProbe:
    httpGet:
      path: /healthz
      port: 8080
      scheme: HTTP
    initialDelaySeconds: 5
    periodSeconds: 3
```

`initialDelaySeconds` 字段告诉 kubelet 在执行第一次探测前应该等待 3 秒
`periodSeconds` 字段指定了 kubelet 每隔 3 秒执行一次存活探测


#### startup Probe 慢启动容器

应用程序在启动时需要较多的初始化时间。 要不影响对引起探测死锁的快速响应，这种情况下，设置存活探测参数是要技巧的。 技巧就是使用一个命令来设置启动探测，针对HTTP 或者 TCP 检测，可以通过设置 failureThreshold * periodSeconds 参数来保证有足够长的时间应对糟糕情况下的启动时间。

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 10
```

应用程序将会有最多 5 分钟(30 * 10 = 300s) 的时间来完成它的启动

### livenessProbe 探活

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
  
  
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Awesome
  initialDelaySeconds: 3
  periodSeconds: 3
```

## 2.优雅中止

### 容器终止流程

1. Pod 被删除，状态置为 Terminating。
2. kube-proxy 更新转发规则，将 Pod 从 service 的 endpoint 列表中摘除掉，新的流量不再转发到该 Pod。
3. 如果 Pod 配置了 preStop Hook ，将会执行。
4. kubelet 对 Pod 中各个 container 发送 `SIGTERM` 信号以通知容器进程开始优雅停止。
5. 等待容器进程完全停止，如果在 terminationGracePeriodSeconds 内 (默认 30s) 还未完全停止，就发送 `SIGKILL` 信号强制杀死进程。
6. 所有容器进程终止，清理 Pod 资源。

#### perStop hook
```yaml
lifecycle:
  preStop:
    exec:
      command:
      - /stop.sh
```
#### SIGTERM & SIGKILL

9 SIGKILL 强制终端

15 SIGTERM 请求中断


#### 资源需求和QoS


* QoS 服务质量
    * Guaranteed
    * Burstable
    * BestEffort

QoS 类为 Guaranteed 的 Pod：

* Pod 中的每个容器都必须指定内存限制和内存请求。
* 对于 Pod 中的每个容器，内存限制必须等于内存请求。
* Pod 中的每个容器都必须指定 CPU 限制和 CPU 请求。
* 对于 Pod 中的每个容器，CPU 限制必须等于 CPU 请求。

```yaml
  limits:
    memory: "200Mi"
    cpu: "700m"
  requests:
    memory: "200Mi"
    cpu: "700m"
```

QoS 类为 Burstable 的 Pod

* Pod 不符合 Guaranteed QoS 类的标准。
* Pod 中至少一个容器具有内存或 CPU 请求。

```yaml
resources:
  limits:
    memory: "200Mi"
  requests:
    memory: "100Mi"
```
QoS 类为 BestEffort 的 Pod
没有设置内存和 CPU 限制或请求

## 3.配置和代码分离，日志等级

https://kubernetes.io/zh/docs/concepts/configuration/configmap/

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"
  loglevel: debug
  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true    
```
```yaml
volumes:
# 你可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
- name: config
  configMap:
    # 提供你想要挂载的 ConfigMap 的名字
    name: game-demo
   ```

```yaml

env:
# 定义环境变量
- name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的
  valueFrom:
    configMapKeyRef:
      name: game-demo           # 这个值来自 ConfigMap
      key: player_initial_lives # 需要取值的键
      
      
- name: UI_PROPERTIES_FILE_NAME
  valueFrom:
    configMapKeyRef:
      name: game-demo
      key: ui_properties_file_name
```


#### 其他
revisionHistoryLimit 节省etcd的存储空间，比如10，表示保留最近10个


progressDeadlineSeconds
检测此状况的一种方法是在 Deployment 规约中指定截止时间参数： （[.spec.progressDeadlineSeconds]（#progress-deadline-seconds））。 

.spec.progressDeadlineSeconds 给出的是一个秒数值，Deployment 控制器在（通过 Deployment 状态）
标示 Deployment 进展停滞之前，需要等待所给的时长。

## 4. service

https://kubernetes.io/zh/docs/concepts/services-networking/service/

四层负载均衡,iptable, 1.11之后变成ipvs，都是netfilter框架的产物

在 `ipvs` 模式下，kube-proxy 监视 Kubernetes 服务和端点，调用 `netlink` 接口相应地创建 IPVS 规则， 并定期将 IPVS 规则与 Kubernetes 服务和端点同步。 该控制循环可确保IPVS 状态与所需状态匹配。访问服务时，IPVS 将流量定向到后端Pod之一。

IPVS代理模式基于类似于 iptables 模式的 netfilter 挂钩函数， 但是使用哈希表作为基础数据结构，并且在内核空间中工作。 这意味着，与 iptables 模式下的 kube-proxy 相比，IPVS 模式下的 kube-proxy 重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。 与其他代理模式相比，IPVS 模式还支持更高的网络流量吞吐量。

IPVS 提供了更多选项来平衡后端 Pod 的流量。 这些是：

* rr：轮替（Round-Robin）
* lc：最少链接（Least Connection），即打开链接数量最少者优先
* dh：目标地址哈希（Destination Hashing）
* sh：源地址哈希（Source Hashing）
* sed：最短预期延迟（Shortest Expected Delay）
* nq：从不排队（Never Queue）

四种类型
* Cluster IP 集群内访问
* NodePort 暴露给外部访问
* loadBalancer 暴露给外部访问
* ExternalName 类似cname，可以实现跨namespace的ingress

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```
当查找主机 `my-service.prod.svc.cluster.local` 时，集群 DNS 服务返回 CNAME 记录， 其值为 `my.database.example.com`

## 5.ingress

7层负载均衡

Ingress 是对集群中服务的外部访问进行管理的 API 对象，典型的访问方式是 HTTP。


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: letsencrypt-staging
  name: fmeng
spec:
  ingressClassName: nginx
  rules:
    - host: fmeng.51.cafe
      http:
        paths:
          - backend:
              service:
                name: httpsvc
                port:
                  number: 80
            path: /
            pathType: Prefix
  tls:
    - hosts:
        - fmeng.51.cafe
      secretName: fmeng-tls
```


```yaml
  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls
```

# kubernetes排错三板斧
1, kubectl describe pod xxx

2, kubectl get pod xxx -oyaml

3, kubectl logs -f xxx
