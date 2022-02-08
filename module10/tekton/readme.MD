### Setup tekton

```sh
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
kubectl apply --filename https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml



kubectl apply --namespace=tekton-argocd --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply -n tekton-argocd --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
kubectl apply -n tekton-argocd --filename https://github.com/tektoncd/dashboard/releases/latest/download/tekton-dashboard-release.yaml

kubectl --namespace tekton-pipelines port-forward --address 0.0.0.0 svc/tekton-dashboard 9097:9097

```

### Setup tekton cli tkn

```sh
curl -LO https://github.com/tektoncd/cli/releases/download/v0.20.0/tektoncd-cli-0.20.0_Linux-64bit.deb
sudo dpkg -i ./tektoncd-cli-0.20.0_Linux-64bit.deb
which tkn
```
