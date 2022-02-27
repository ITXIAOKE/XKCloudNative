### Create a kubebuilder project, which requires an empty folder

```sh
kubebuilder init --domain cncamp.io
```

### Check project layout

```sh
cat PROJECT

domain: cncamp.io
layout:
- go.kubebuilder.io/v3
projectName: mysts
repo: github.com/cncamp/demo-operator
version: "3"
```

### Create API, create resource[Y], create controller[Y]

```sh
kubebuilder create api --group apps --version v1beta1 --kind MyDaemonset
```

### Open project with IDE and edit `api/v1alpha1/simplestatefulset_types.go`

```sh
// MyDaemonsetSpec defines the desired state of MyDaemonset
type MyDaemonsetSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of MyDaemonset. Edit mydaemonset_types.go to remove/update
	Image string `json:"image,omitempty"`
}

// MyDaemonsetStatus defines the observed state of MyDaemonset
type MyDaemonsetStatus struct {
	AvaiableReplicas int `json:"avaiableReplicas,omitempty"`
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
}
```

### Check Makefile

```makefile
Build targets:
    ### create code skeletion
    manifests: generate crd
    generate: generate api functions, like deepCopy

    ### generate crd and install
    run: Run a controller from your host.
    install: Install CRDs into the K8s cluster specified in ~/.kube/config.

    ### docker build and deploy
    docker-build: Build docker image with the manager.
    docker-push: Push docker image with the manager.
    deploy: Deploy controller to the K8s cluster specified in ~/.kube/config.
```

### Edit `controllers/mydaemonset_controller.go`, add permissions to the controller
```sh
//+kubebuilder:rbac:groups=apps.cncamp.io,resources=mydaemonsets/finalizers,verbs=update
// Add the following
//+kubebuilder:rbac:groups=core,resources=nodes,verbs=get;list;watch
//+kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch;create;update;patch;delete
```

### Generate crd

```sh
make manifests
```

### Build & install

```sh
make build
make docker-build
make docker-push
make deploy
```

## Enable webhooks

### Install cert-manager

```sh
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.1/cert-manager.yaml
```

### Create webhooks

```sh
kubebuilder create webhook --group apps --version v1beta1 --kind MyDaemonset --defaulting --programmatic-validation

 kubebuilder create webhook --group xiaoke --version v1 --kind Xiaoke --defaulting --programmatic-validation

```
##使webhook生效改变代码地方如下：


### Enable webhook in `config/default/kustomization.yaml`

### Redeploy

### Operator 开发三大步骤
```sh
1，定义crd的struct

type XiaokeSpec struct {
	Image string `json:"image,omitempty"`
}

2，写controller 控制器

//消费者
func (r *XiaokeReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// your logic here


	return ctrl.Result{}, nil
}

//接收者
func (r *XiaokeReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&xiaokev1.Xiaoke{}).
		Complete(r)
}
 
3，建webhook ，然后通过certmanager进行签发证书或者手动通过openssl签证书

func (r *Xiaoke) SetupWebhookWithManager(mgr ctrl.Manager) error {
	return ctrl.NewWebhookManagedBy(mgr).
		For(r).
		Complete()
}

var _ webhook.Defaulter = &Xiaoke{}

func (r *Xiaoke) Default() {
	xiaokelog.Info("default", "name", r.Name)
}

var _ webhook.Validator = &Xiaoke{}

func (r *Xiaoke) ValidateCreate() error {
	xiaokelog.Info("validate create", "name", r.Name)
	return nil
}

func (r *Xiaoke) ValidateUpdate(old runtime.Object) error {
	xiaokelog.Info("validate update", "name", r.Name)
	return nil
}

func (r *Xiaoke) ValidateDelete() error {
	xiaokelog.Info("validate delete", "name", r.Name)
	return nil
}


```

###webhook手动 openssl签证书

```sh
workdir=${1}
keydir=$workdir/keys
mkdir -p $keydir

echo Generating the CA cert and private key to ${keydir}
openssl req -nodes -new -x509 -keyout ${keydir}/ca.key -out ${keydir}/ca.crt -subj "/CN=crane"

echo Generating the private key for the webhook server
openssl genrsa -out ${keydir}/tls.key 2048

# Generate a Certificate Signing Request (CSR) for the private key, and sign it with the private key of the CA.
echo Signing the CSR, and generating cert into ${keydir}
openssl req -new -key ${keydir}/tls.key -subj "/CN=webhook-service.crane-system.svc" -config ${workdir}/scripts/webhook.csr \
    | openssl x509 -req -days 3650 -CA ${keydir}/ca.crt -CAkey ${keydir}/ca.key -CAcreateserial -out ${keydir}/tls.crt -extensions v3_req -extfile ${workdir}/scripts/webhook.csr

```
