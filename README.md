# test-operator
kubebuilder 的简单使用

创建一个简单的web crd



## test-operator开发流程

**前言**

作者是用的windows系统，但windows系统暂时不能使用kubebuilder，所以操作均在一套k8s的master linux服务器上，通过gitlab同步代码，具体代码是在windows下编写。

**工具下载**

```
kubebuilder
https://github.com/kubernetes-sigs/kubebuilder/releases/download/v2.3.2/kubebuilder_2.3.2_linux_amd64.tar.gz

controller-tool
git clone https://github.com/kubernetes-sigs/controller-tools.git
go build cmd/controller-gen/main.go
```

**环境**

```
go version
go version go1.14.4 linux/amd64

kubebuilder version
Version: version.Version{KubeBuilderVersion:"2.3.2", KubernetesVendor:"1.16.4", GitCommit:"5da27b892ae310e875c8719d94a5a04302c597d0", BuildDate:"2021-02-16T19:31:47Z", GoOs:"unknown", GoArch:"unknown"}

controller-gen --version
Version: v0.4.1
```

****

**step1**

```shell
创建目录
# mkdir test-kubebuilder ;cd test-kubebuilder
初始化项目脚手架
# go mod init zhangjinhui.online/m
# kubebuilder init --domain test.zhangjinhui.online
此时目录结构如下
# tree 
.
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   └── role_binding.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── PROJECT

9 directories, 30 files
```

**step2**

```shell
创建API
# kubebuilder create api --group test --version v1 --kind Test
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
api/v1/test_types.go
controllers/test_controller.go
Running make:
$ make
/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go

此时目录结构如下
# tree 
.
├── api
│   └── v1
│       ├── groupversion_info.go
│       ├── test_types.go
│       └── zz_generated.deepcopy.go
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── crd
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_tests.yaml
│   │       └── webhook_in_tests.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── role_binding.yaml
│   │   ├── test_editor_role.yaml
│   │   └── test_viewer_role.yaml
│   ├── samples
│   │   └── test_v1_test.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── controllers
│   ├── suite_test.go
│   └── test_controller.go
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── PROJECT

15 directories, 42 files
```

**step3**

```shell
此时只修改两个文件
- api/v1/test_types.go
- controllers/test_controller.go

在test_type.go中，添加自己需要的内容
# cat api/v1/test_types.go
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)


type TestSpec struct {
	Image string `json:"image"`
	Replicas int32 `json:"replicas"`
	Port int32 `json:"port"`
	TargetPort int32 `json:"targetPort,omitempty"`
	NodePort int32 `json:"nodePort,omitempty"`
}
type TestStatus struct {
	Replicas int32 `json:"replicas"`
}

// +kubebuilder:object:root=true

type Test struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`
	Spec   TestSpec   `json:"spec,omitempty"`
	Status TestStatus `json:"status,omitempty"`
}
// +kubebuilder:object:root=true

type TestList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Test `json:"items"`
}

func init() {
	SchemeBuilder.Register(&Test{}, &TestList{})
}
```

```
在test_controller.go中，修改逻辑（删减后）
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps,resources=deployments/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=test.test.zhangjinhui.online,resources=tests,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=test.test.zhangjinhui.online,resources=tests/status,verbs=get;update;patch

func (c *TestReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	ctx := context.Background()
	reqLogger = c.Log.WithValues("Request.Namespace", req.Namespace, "Request.Name", req.Name)
	//判断该次Reconcile应该做什么操作
	var test testv1.Test
	//判断该cr还存不存在， 如果不存在，则清理资源
	test, err := c.isCrdExist(ctx, req)
	if err != nil{
		//报错则已经被删除， 此时需要清理该crd的资源，然后退出
		c.clear(ctx, req)
		return ctrl.Result{}, nil
	}


	//如果获取到该cr， 则更新，或创建资源
	//判断是否存在deployment，存在则更新deploy，不存在则创建
	if deploy, err := c.isDeployExist(ctx, req);err != nil{
		reqLogger.Info("没有获取到该deployment,正在创建该deployment.")
		c.createDeployment(ctx, test)
	}else {
		reqLogger.Info("获取到deployment,正在更新该deployment.")
		c.updateDeployment(ctx, test, deploy)
	}

	if service, err := c.isServiceExist(ctx, req);err != nil{
		reqLogger.Info("没有获取到该service,正在创建该service.")
		c.createService(ctx, test)
	}else {
		reqLogger.Info("获取到service,正在更新该service.")
		c.updateService(ctx, test, service)
	}
	// your logic here

	return ctrl.Result{}, nil
}
```



**step4**

```shell
编译及打成镜像推送
export IMG=你们的harbor或者repo地址+image:tag
# make docker-build docker-push
...

生成RBAC配置
# make manifests
...

如果你也在k8smaster节点或者可连接k8s的kubectl节点，就可以这样部署
# make deploy
...

此时
# kubectl get pod -n test-kubebuilder-system
NAME                                                  READY   STATUS    RESTARTS   AGE
test-kubebuilder-controller-manager-9ffb6d565-k959r   2/2     Running   0          41m

# kubectl get crd
NAME                                        CREATED AT
tests.test.test.zhangjinhui.online          2021-03-23T08:29:52Z
...
```

**step5**

```shell
创建一个cr(对应上面test_types.go)
# cat config/samples/test_v1_test.yaml 
apiVersion: test.test.zhangjinhui.online/v1
kind: Test
metadata:
  name: test-sample
spec:
  # Add fields here
  image: nginx:latest
  replicas: 3
  port: 80
  targetPort: 80
  nodePort: 30043

# kubectl apply -f config/samples/test_v1_test.yaml
...
# kubectl get pod,svc
NAME                                       READY   STATUS    RESTARTS   AGE
pod/test-sample-669d698cbd-7f5kd           1/1     Running   0          42m
pod/test-sample-669d698cbd-kr9d6           1/1     Running   0          42m
pod/test-sample-669d698cbd-zjvmn           1/1     Running   0          42m

NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/test-sample           NodePort    10.0.208.123   <none>        80:30043/TCP     44m
```

**step6**

```shell
修改配置，测试功能
# cat config/samples/test_v1_test.yaml 
apiVersion: test.test.zhangjinhui.online/v1
kind: Test
metadata:
  name: test-sample
spec:
  # Add fields here
  image: nginx:latest
  replicas: 4
  port: 80
  targetPort: 80
  nodePort: 30044
  
# kubectl apply -f config/samples/test_v1_test.yaml 
...
# kubectl get pod,svc
NAME                                       READY   STATUS    RESTARTS   AGE
pod/test-sample-669d698cbd-7f5kd           1/1     Running   0          45m
pod/test-sample-669d698cbd-9qk4t           1/1     Running   0          4s
pod/test-sample-669d698cbd-kr9d6           1/1     Running   0          45m
pod/test-sample-669d698cbd-zjvmn           1/1     Running   0          45m

NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/test-sample           NodePort    10.0.208.123   <none>        80:30044/TCP     46m

```

### 注意

```
1.go环境提前做goproxy,以及GO111MODULE="on"
2.kubebulder下载的四个二进制工具最好放在/usr/local/kubebuilder/bin下，然后加PATH，不然会报错找不到etcd，或者apiserver
3.Dockerfile提前修改，不然拉不到镜像
```

这样，一个简单的test-operator就写好了

参考连接 <a href="https://www.jianshu.com/p/b678ee6545a1">点我跳转</a>