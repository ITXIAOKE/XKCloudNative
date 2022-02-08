### Install Kubernetes Cluster

### kubeadm

通过 kubeadm 安装，需要手工安装 docker，kubelet，kubeadm 等

- [ubuntu 安装](k8s-by-kubeadm)
- [centos 安装](kubeadm-centos7-install.md) @水果爸爸

### kind

通过 [kind](1.2.kind-setup.md) 安装，全自动，无手工步骤，快捷简单，但是节点是基于容器而不是虚拟机的。

### minikube

通过 [minikube](1.minikube-setup.md) 安装。

### Vagrant

如果配置 kubeadm 有困难可参考 [Vagrantfile](Vagrantfile) @nevil

### 跨公网安装集群

- [利用 Kilo 跨公网组建 k3s 集群](https://zsnmwy.notion.site/Kilo-k3s-a3b5c98fab14432594f6bdbaeffb7c77)
- [利用 FabEdge 跨公网组建 k3s 集群](https://zsnmwy.notion.site/FabEdge-k3s-43ef0b8662be4c70bc74185c05cd517e)


###dashboard的密码生成
kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"


###访问
访问前先执行以下命令
kubectl proxy

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/node/docker-desktop?namespace=default