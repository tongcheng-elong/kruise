同程艺龙大数据平台kruise 0.8.1二次开发
# 一、简介
主要是利用kruise对部署对象自动增加网络标签。以及利用OpenKruise SidecarSet的在新建、扩容、重建 pod 的场景中，实现 sidecar 容器的自动注入。

# 二、环境准备
修改和编译OpenKruise需要准备一下环境：
- golong(编译)
- minikube(开发调试最小化集群)
- Go-bindata(编译的时候需要)
- kubebuilder(crd开发脚手架，用于自动生成crd)
- kustomize(用于编辑和构建配置yaml模板)
## 2.1 go环境安装
```
cd /data/soft && wget https://dl.google.com/go/go1.16.4.linux-amd64.tar.gz
tar zxvf go1.16.4.linux-amd64.tar.gz  -C /usr/local

编辑/etc/profile文件添加如下：

# set go environment
export GOROOT=/usr/local/go
export GOPATH=/usr/local/gopath
export PATH=$PATH:$GOROOT/bin
export PATH=$PATH:$GOPATH/bin

# source /etc/profile 生效

验证：

# go version
go version go1.16.4 linux/amd64
```
国外的比较慢，因此安装完后需要设置代理。

```
// 阿里云go代理
go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct
// go国内代理，七牛代理
go env -w GOPROXY=https://goproxy.cn,direct
```

在下载过程过如果发现慢，可以在两个代理都换着试一试。

## 2.2 安装minikube开发环境

### 2.2.1 安装minikube

主要是安装minikube用来做本机开发调试环境。无需直接连接到生产或者是测试环境，造成对正常使用环境的干扰。当然也可以选择安装kind来做开发调试环境。[minikube官方安装文档](https://minikube.sigs.k8s.io/docs/start/)

```
 curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
 sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

安装后查看版本

```
[root@master0 yaml]# minikube version
minikube version: v1.20.0
commit: c61663e942ec43b20e8e70839dcca52e44cd85ae
```

安装完成minikube后记需要安装kubectl命令行工具。kubectl工具的版本要想要安装的k8s版本对应。

### 2.2.2 创建集群

由于当前使用的kubernetes版本是1.14.8，因此我们创建集群的时候需要制定kubernetes的版本，否侧minikube会创建一个最新版本的k8s集群。

```
minikube start --image-mirror-country cn --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.5.0.iso  --registry-mirror=https://pe3ox7bd.mirror.aliyuncs.com --driver="none" --memory=2048  --kubernetes-version=v1.14.8 --feature-gates=CustomResourceWebhookConversion=true
```

重要的几个配置：

- `--image-mirror-country`制定镜像国家；
- `--iso-url`指定镜像地址，使用国内提供的镜像更快，这里使用的是阿里云提供的；
- `--registry-mirror`指定镜像加速；
- `--memory`指定内存大小，最小要保证2GB，可以不指定；
- `--kubernetes-version`指定k8s的版本
- `--feature-gates` 指定一些新功能特性

> 由于CustomResourceWebhookConversion在kubernetes 1.16以前的版本属于实验性的功能，因此默认是关闭的，OpenKruise需要使用这个特性，我们就要手动打开这个配置。该配置最终是属于kube-apiserver的配置。生产级部署的kubernetes环境需要在kube-apiserver的配置文件中配置。

设置打开CustomResourceWebhookConversion的原因是从0.7.1版本开始，OpenKruise定义`statefulsets.apps.kruise.io`这个`CustomResourceDefinition`的时候设置了`conversion`使用的策略是`strategy: Webhook`

### 2.2.3 启动集群

如果minikube没有跟着开机自启动，则需要自行启动已经安装好的默认集群。

```
minikube start --feature-gates=CustomResourceWebhookConversion=true
```

## 2.3  安装kubebuilder

使用 kubebuilder 工具，可以构建一个 Kubernetes Operator框架

执行如下命令。（如果curl下载失败，大概率是网络原因，可以手动下载，解压到指定目录）。

```
os=$(go env GOOS)
arch=$(go env GOARCH)
curl -L https://go.kubebuilder.io/dl/2.3.1/${os}/${arch} | tar -xz -C /tmp/
sudo mv /tmp/kubebuilder_2.3.1_${os}_${arch} /usr/local/kubebuilder

export PATH=$PATH:/usr/local/kubebuilder/bin
```

通过源码安装：

```
git clone https://github.com/kubernetes-sigs/kubebuilder
cd kubebuilder
make build
cp bin/kubebuilder $GOPATH/bin
```

> 没安装会出现下面的错误：fork/exec /usr/local/kubebuilder/bin/etcd: no such file or directory

kubebuilder的帮助指令查看：

```
kubebuilder --help
```

## 2.4 安装kustomize

kustomize是渲染yaml的神器。安装操作如下：

```
wget https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh
```

如果不能下载的话直接复制url去浏览器打开把脚本复制机器上执行。

# 三、编译OpenKruise

## 3.1 编译准备

准备编译环境后就可以下载`OpenKruise`源码来编译了。当然针对`k8s 1.14.8`版本，我们还需要对项目中的make执行做一些修改才能直接在后面本地一键命令部署到开发集群，原因主要是kubernetes当前的版本导致api没法认证通过，所以需要在执行`kubectl`指令部署的时候添加`--validate=false`选项。

编辑Makefile文件，在文件中找到如下片段

```
deploy: manifests
	cd config/manager && kustomize edit set image controller=${IMG}
	kustomize build config/default | kubectl apply -f -
	echo "resources:\n- manager.yaml" > config/manager/kustomization.yaml
```

修改为

```
deploy: manifests
	cd config/manager && kustomize edit set image controller=${IMG}
	kustomize build config/default | kubectl apply --validate=false -f -
	echo "resources:\n- manager.yaml" > config/manager/kustomization.yaml
```

如果想直接输出模板，则修改为下面的方式

```
deploy: manifests
	cd config/manager && kustomize edit set image controller=${IMG}
	kustomize build config/default > all-in-one.yaml
    kubectl apply -f all-in-one.yaml --validate=false
	echo "resources:\n- manager.yaml" > config/manager/kustomization.yaml
```
> 在k8s 1.14.8的环境下执行必须要加上`--validate=false`配置。否则没法正常部署。主要是API版本验证的问题。
## 3.2  代码修改

公司内部主要是利用kruise的webhook功能，在创建pod的时候自动给pod添加vlan网络的标签。修改`pkg/webhook/pod/mutating/pod_create_update_handler.go`的`func (h *PodCreateHandler) Handle(ctx context.Context, req admission.Request) admission.Response`方法。

```
// Handle handles admission requests.
func (h *PodCreateHandler) Handle(ctx context.Context, req admission.Request) admission.Response {
	obj := &corev1.Pod{}
	err := h.Decoder.Decode(req, obj)
	if err != nil {
		return admission.Errored(http.StatusBadRequest, err)
	}
	// when pod.namespace is empty, using req.namespace
	if obj.Namespace == "" {
		obj.Namespace = req.Namespace
	}

	// Get vlan tenant from env
	tenant := os.Getenv("io.contiv.tenant")
	// Get vlan network from env
	network := os.Getenv("io.contiv.network")
	klog.Infof("io.contiv.tenant is %v,io.contiv.network is %v",tenant,network)

	copy := obj.DeepCopy()
	injectPodReadinessGate(req, copy)
	err = h.sidecarsetMutatingPod(ctx, req,copy)
	if err != nil {
		return admission.Errored(http.StatusInternalServerError, err)
	}
	if copy.Labels == nil {
		copy.Labels = make(map[string]string)
		copy.Labels["io.contiv.tenant"] = tenant
		copy.Labels["io.contiv.network"] = network
	} else {
		if _, ok := copy.Labels["io.contiv.tenant"]; !ok {
			copy.Labels["io.contiv.tenant"] = tenant
		}
		if _, ok := copy.Labels["io.contiv.network"]; !ok {
			copy.Labels["io.contiv.network"] = network
		}
	}
	marshalled, err := json.Marshal(copy)
	if err != nil {
		return admission.Errored(http.StatusInternalServerError, err)
	}
	return admission.PatchResponseFromRaw(req.AdmissionRequest.Object.Raw, marshalled)
}
```

## 3.3 编译部署步骤

编译步骤如下部署步骤如下：

- `make manager` 执行该指令来验证代码是否可以通过，尤其是自己修改了代码；
- `dockerhub`上创建一个镜像仓库用于放`OpenKruise`，当然使用公司能不仓库可以省略此步骤；
- `docker login`登录镜像仓库；
- `export IMG=<image_name>`指定一个镜像名称，例如：`export IMG=$DOCKERID/kruise:test`；
- `make docker-build` 构建镜像到本地；
- `make docker-push` 推送镜像到仓库；
- `make deploy`部署`OpenKruise`到本地集群；
# 四、验证

## 4.1 验证pod部署

```
cat > nginx-test.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-test
  name: nginx-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:1.20.0-alpine
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-test
spec:
  ports:
  - name: 80-tcp
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx-test
  type: ClusterIP
EOF
```

部署好后查看vlan标签是否被自动添加上去

```
kubectl describe pod nginx-test-xxx
```

显示vlan网络标签则说明已经成功

```
Labels:             app=nginx-test
                    io.contiv.network=vlan-202
                    io.contiv.tenant=default
```

## 4.2 SideCarSet验证
验证`OpenKruise`的`SideCarSet`来实现了将`SideCar`自动注入到容器中。

创建sidecarset
```
cat > sidecarSet.yaml <<EOF
apiVersion: apps.kruise.io/v1alpha1
kind: SidecarSet
metadata:
  name: logtail-sidecarset
spec:
  selector:
    matchLabels:
      app: logtail
  updateStrategy:
    type: RollingUpdate
    maxUnavailable: 10%
  containers:
  - name: logtail
    image: alibabacloudsls/logtail:latest
    # when recevie sigterm, logtail will delay 10 seconds and then stop
    command:
    - sh
    - -c
    - /usr/local/ilogtail/run_logtail.sh 10
    livenessProbe:
      exec:
        command:
        - /etc/init.d/ilogtaild
        - status
    resources:
      limits:
        memory: 512Mi
      requests:
        cpu: 10m
        memory: 30Mi
    ##### share this volume    
    volumeMounts:
    - name: nginx-log
      mountPath: /var/log/nginx
    transferEnv:
    - sourceContainerName: nginx
      envName: TZ 
  volumes:
  - name: nginx-log
    emptyDir: {}
EOF
```
验证sidecarset注入
```aidl
apiVersion: v1
kind: Pod
metadata:
  labels:
    # matches the SidecarSet's selector
    app: logtail
  name: test-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.20.0-alpine
    #command: ["/bin/mock_log"]
    #args: ["--log-type=nginx", "--stdout=false", "--stderr=true", "--path=/var/log/nginx/access.log", "--total-count=1000000000", "--logs-per-sec=100"]
    volumeMounts:
    - name: nginx-log
      mountPath: /var/log/nginx
    env:
    - name: TZ
      value: Asia/Shanghai
  volumes:
  - name: nginx-log
    emptyDir: {}

```
