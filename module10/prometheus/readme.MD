### Sync VM time with host

```sh
shutdown vm
cd ~/"VirtualBox VMs"
VBoxManage list vms
"localkube" {014a8874-1cbe-43ec-a47c-ce7248bce13e}
vboxmanage setextradata "localkube" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" "1"
vboxmanage setextradata "crane" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" "1"
```

### Start VM and then you can get prometheus data

### Grafana dashboard

```sh
cluster health
6417
pod dashboard
9729
Istio Mesh
7639
```

#prometheus的黑盒监控
###比如检测下 gaoxin.kubeasy.com（使用任何一个公网域名或者公司内的域名探测即可）网站
的状态，可以通过如下命令进行检查：
```sh
curl -s "http://192.168.204.117:19115/probe?target=gaoxin.kubeasy.com&module=http_2xx" | tail -1

probe_tls_version_info{version="TLS 1.2"} 1

probe 是接口地址，target 是检测的目标，module 是使用哪个模块进行探测。

如果集群中没有配置 Blackbox Exporter，可以参考
https://github.com/prometheus/blackbox_exporter 进行安装。
```

# 通过 Service Monitor 监控应用时，如果监控没有找到目标主机的排查步骤时，排查步骤如下：
```sh
➢ 确认 Service Monitor 是否成功创建

➢ 确认 Service Monitor 标签是否配置正确

➢ 确认 Prometheus 是否生成了相关配置

➢ 确认存在 Service Monitor 匹配的 Service

➢ 确认通过 Service 能够访问程序的 Metrics 接口

➢ 确认 Service 的端口和 Scheme 和 Service Monitor 一致
```
