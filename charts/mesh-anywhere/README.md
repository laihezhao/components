## 部署服务网格组件

### 前提依赖

以下组件应当已经在u4a-component模块中部署：

- cert-manager
- capsule
- karmada (only cluster CRD)
- roletemplate CRD
- license-operator
- k8s ingress-controller

### 在“管理集群”部署管理平台

管理集群指的是mesh-api和tdsf-portal所在的k8s集群

给管理集群的 cluster 添加 label 标识

```bash
# cluster-xxxx 为要设置为管理集群的集群名称，在集群管理页面中可查看到集群名称
kubectl label cluster cluster-xxxx primary=true
```

网格的部分功能依赖监控组件，部署 TDSF 前需预先在集群部署监控组件。

#### 1. 创建namespace

管理集群上需要创建三个namespace
- mesh-system
- registry-system
- istio-system

```bash
kubectl --as=admin --as-group=iam.tenxcloud.com create -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  labels:
     capsule.clastix.io/tenant: system-tenant
  name: mesh-system
EOF

kubectl --as=admin --as-group=iam.tenxcloud.com create -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  labels:
     capsule.clastix.io/tenant: system-tenant
  name: registry-system
EOF

kubectl --as=admin --as-group=iam.tenxcloud.com create -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  labels:
     capsule.clastix.io/tenant: system-tenant
  name: istio-system
EOF
```

#### 2. 调整value.yaml

1. 根据需要对values.yaml进行修改。其中global.host值设置为true，表明这是管理集群

2. global.clusterReaderNameOverride 字段，此处与管理工作台部署时的 clusterReaderName 保持一致

获取 clusterReaderName 的方式：kubectl get sa -n addon-system|grep cluster-reader

3. image.registry 表示仓库地址，根据需要调整

4. global.ingress.className 和 global.ingress.hostName 需与管理控制台部署的 ingress 的参数保持一致

查看这两个参数的方式：kubectl get ingress -n u4a-system bff-server-ingress -oyaml：

* global.ingress.className 为 annotations.kubernetes.io/ingress.class 的值
* global.ingress.hostName 为 spec.rules.host 的值

5. 在 mesh-api.env 中，根据需要对管理控制台的 license 所在的 namespace 进行调整，默认在 u4a-system

```yaml
global:
  host: true
  clusterReaderNameOverride: ""
  registry: &registry
    172.22.50.227
  ingress:
    className: &ingressClass
      "portal-ingress"
    hostName: &ingressHost
      "portal.172.22.99.237.nip.io"

mesh-operator:
  image:
    registry: *registry
    repository: dev-branch/mesh-operator
    tag: v5.6.0
  env:
    - name: LEADER_ELECTION_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: ISTIO_VM_SERVER_TAG
      value: v5.6.0

registry-adaptor:
  image:
    registry: *registry
    repository: dev-branch/registry-adaptor-controller
    tag: v5.6.0
  kubeRbacProxyImage:
    registry: *registry
    repository: dev-branch/kubebuilder-kube-rbac-proxy
    tag: v0.8.0

mesh-api:
  image:
    registry: *registry
    repository: dev-branch/mesh-api
    tag: v5.6.0
  env:
    - name: app.license.namespaceOfLicense
      value: "u4a-system"
    - name: MESH_AGENT_IMAGE_SUFFIX
      value: "/system_containers/istio-vm-init:v5.6.0"
    - name: NS_MESH
      value: "mesh-system"
    - name: NS_REGISTRY
      value: "registry-system"
    - name: NS_ISTIO
      value: "istio-system"
    - name: NS_MONITORING
      value: "monitoring-system"
  ingress:
    className: *ingressClass
    hostName: *ingressHost

tdsf-portal:
  image:
    registry: *registry
    repository: dev-branch/tdsf-portal
    tag: v5.6.0
  ingress:
    className: *ingressClass
    hostName: *ingressHost
```

#### 3. 运行helm命令部署

正式部署之前，可以通过dry run方式验证：
```bash
helm install tdsf . -n mesh-system -f primary-values.yaml --dry-run
```

正式部署：
```bash
helm install tdsf . -n mesh-system -f primary-values.yaml
```

### 加入“纳管集群”

纳管集群指的是平台上管理的其他k8s集群，非mesh-api和tdsf-portal所在的集群

网格的部分功能依赖监控组件，部署 TDSF 前需预先在集群内部署监控组件。

如果存在多个纳管集群，每个集群分别都要执行下列操作

#### 1. 创建namespace

托管集群上需要创建两个namespace
- registry-system
- istio-system

```bash
kubectl --as=admin --as-group=iam.tenxcloud.com create -f - <<EOF 
apiVersion: v1
kind: Namespace
metadata:
  labels:
     capsule.clastix.io/tenant: system-tenant
  name: registry-system
EOF

kubectl --as=admin --as-group=iam.tenxcloud.com create -f - <<EOF 
apiVersion: v1
kind: Namespace
metadata:
  labels:
     capsule.clastix.io/tenant: system-tenant
  name: istio-system
EOF
```

#### 2. 调整value.yaml

1. 根据需要对 values.yaml 进行修改，其中global.host值设置为false，表明这不是管理集群

2. global.clusterReaderNameOverride 字段，此处与管理工作台部署时的 clusterReaderName 保持一致

获取 clusterReaderName 的方式：kubectl get sa -n addon-system|grep cluster-reader

3. image.registry  表示仓库地址，根据需要调整

```yaml
global:
  host: false
  clusterReaderNameOverride: ""
  registry: &registry 172.22.50.227

registry-adaptor:
  image:
    registry: *registry
    repository: dev-branch/thirdparty-registry-mesh-controller
    tag: v5.6.0
  kubeRbacProxyImage:
    registry: *registry
    repository: dev-branch/kubebuilder-kube-rbac-proxy
    tag: v0.8.0
```

#### 3. 运行helm命令部署

正式部署之前，可以通过dry run方式验证：
```bash
helm install tdsf . -n mesh-system -f remote-values.yaml --dry-run
```

正式部署：
```bash
helm install tdsf . -n mesh-system -f remote-values.yaml
```


## 更新menu

```
1. 获取menu
docker run --rm -v ${PWD}/menus:/tmp --entrypoint cp  172.22.50.223/release-5.6/portal-menus:v5.6  /tdsf-portal/menu.yaml /tmp/tdsf-portal-menu.yaml

2. 替换本helm 包 中 tdsf-portal-menu.yaml 文件

```
