# Install mesh-anywhere with Kubebb

**Note:This documentation suppose your cluster deployed by [Sample Cluster](https://kubebb.github.io/website/docs/core/get_started#%E5%87%86%E5%A4%87kubernetes%E9%9B%86%E7%BE%A4)**

## Prerequisites

- [kubebb-core](https://github.com/kubebb/components/tree/main/charts/kubebb-core) installed

- Repository [kubebb](https://github.com/kubebb/components/blob/main/repos/repository_kubebb.yaml) created and synced

```shell
    kubectl apply -n kubebb-system -f repos/repository_kubebb.yaml
```

- Component [u4a-component](https://github.com/kubebb/components/tree/main/charts/u4a-component) installed with [this `ComponentPlan`](https://github.com/kubebb/components/blob/main/examples/u4a-component/componentplan.yaml)

## Install mesh-anywhere in primary cluster

### 1. Add the label for the primary cluster

```shell
    kubectl label cluster cluster-xxxx primary=true
```

### 2. Create the namespaces

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

### 3. Modify the primary-values.yaml file

- 根据需要对primary-values.yaml进行修改。其中global.host值设置为true，表明这是管理集群

- global.clusterReaderNameOverride 字段，此处与管理工作台部署时的 clusterReaderName 保持一致

获取 clusterReaderName 的方式：kubectl get sa -n addon-system|grep cluster-reader

- image.registry 表示仓库地址，根据需要调整

- global.ingress.className 和 global.ingress.hostName 需与管理控制台部署的 ingress 的参数保持一致

查看这两个参数的方式：kubectl get ingress -n u4a-system bff-server-ingress -oyaml：

* global.ingress.className 为 annotations.kubernetes.io/ingress.class 的值
* global.ingress.hostName 为 spec.rules.host 的值

- 在 mesh-api.env 中，根据需要对管理控制台的 license 所在的 namespace 进行调整，默认在 u4a-system

```yaml
global:
  host: true
  clusterReaderNameOverride: ""
  registry: &registry
    docker.io
  ingress:
    className: &ingressClass
      "portal-ingress"
    hostName: &ingressHost
      "portal.172.22.99.237.nip.io"

mesh-operator:
  image:
    registry: *registry
    repository: hyperledgerk8s/mesh-operator
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
    repository: hyperledgerk8s/registry-adaptor-controller
    tag: v5.6.0
  kubeRbacProxyImage:
    registry: *registry
    repository: hyperledgerk8s/kubebuilder-kube-rbac-proxy
    tag: v0.8.0

mesh-api:
  image:
    registry: *registry
    repository: hyperledgerk8s/mesh-api
    tag: v5.6.0
  env:
    - name: app.license.namespaceOfLicense
      value: "u4a-system"
    - name: MESH_AGENT_IMAGE_SUFFIX
      value: "/hyperledgerk8s/istio-vm-init:v5.6.0"
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
    repository: hyperledgerk8s/tdsf-portal
    tag: v5.6.0
  ingress:
    className: *ingressClass
    hostName: *ingressHost
```

### 4. Create configMaps for storing configuration of primary-values.yaml file

```bash
  kubectl create configMap mesh-anywhere-helm-values --from-file=primary.yaml=primary-values.yaml -n kubebb-system
```

### 5. Apply `componentplan_primary.yaml`

```shell
  kubectl apply -f  examples/mesh-anywhere/componentplan_primary.yaml
```

## Install mesh-anywhere in remote cluster

remote cluster指的是平台上管理的其他k8s集群，非mesh-api和tdsf-portal所在的集群

网格的部分功能依赖监控组件，需预先在集群内部署监控组件。

如果存在多个remote cluster，每个集群分别都要执行下列操作

### 1. 创建namespace

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

### 2. Modify remote-values.yaml

1. 根据需要对 values.yaml 进行修改，其中global.host值设置为false，表明这不是管理集群

2. global.clusterReaderNameOverride 字段，此处与管理工作台部署时的 clusterReaderName 保持一致

获取 clusterReaderName 的方式：kubectl get sa -n addon-system|grep cluster-reader

3. image.registry  表示仓库地址，根据需要调整

```yaml
global:
  host: false
  clusterReaderNameOverride: ""
  registry: &registry docker.io

registry-adaptor:
  image:
    registry: *registry
    repository: hyperledgerk8s/thirdparty-registry-mesh-controller
    tag: v5.6.0
  kubeRbacProxyImage:
    registry: *registry
    repository: hyperledgerk8s/kubebuilder-kube-rbac-proxy
    tag: v0.8.0
```

### 3. Create configMaps for storing configuration of remote-values.yaml file

```bash
  kubectl create configMap mesh-anywhere-helm-values --from-file=remote.yaml=remote-values.yaml -n kubebb-system
```

### 4. Apply `componentplan_remote.yaml`

```shell
  kubectl apply -f  examples/mesh-anywhere/componentplan_remote.yaml
```


