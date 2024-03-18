# 部署k8s多集群管理利器Karmada


# 部署k8s多集群管理利器Karmada

## 创建两个kubernetes集群

```bash
cat <<EOF > cluster01-kind.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cluster01
nodes:
- role: control-plane
networking:
  podSubnet: "10.10.0.0/16"
  serviceSubnet: "10.11.0.0/16"
EOF

kind create cluster --config  cluster01-kind.yaml
kubectl config set-cluster kind-cluster01 --server=https://$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' cluster01-control-plane):6443

cat <<EOF > cluster02-kind.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cluster02
nodes:
- role: control-plane
networking:
  podSubnet: "10.20.0.0/16"
  serviceSubnet: "10.21.0.0/16"
EOF

kind create cluster --config  cluster02-kind.yaml
kubectl config set-cluster kind-cluster02 --server=https://$(docker inspect -f '{{.NetworkSettings.Networks.kind.IPAddress}}' cluster02-control-plane):6443
```

## Karmada kubectl 插件
> kubectl-krew: https://krew.sigs.k8s.io/

```bash
kubectl-krew install karmada
```

## 通过 Karmada 命令行工具安装 Karmada

```bash
kubectl karmada init --context="kind-cluster01"

I0219 19:40:52.380107   12365 deploy.go:245] kubeconfig file: , kubernetes: https://172.18.0.2:6443
I0219 19:40:52.410975   12365 deploy.go:265] karmada apiserver ip: [172.18.0.2]
I0219 19:40:52.960643   12365 cert.go:246] Generate ca certificate success.
I0219 19:40:53.186575   12365 cert.go:246] Generate karmada certificate success.
I0219 19:40:53.266349   12365 cert.go:246] Generate apiserver certificate success.
I0219 19:40:53.436140   12365 cert.go:246] Generate front-proxy-ca certificate success.
I0219 19:40:53.525006   12365 cert.go:246] Generate front-proxy-client certificate success.
I0219 19:40:53.921380   12365 cert.go:246] Generate etcd-ca certificate success.
I0219 19:40:54.088352   12365 cert.go:246] Generate etcd-server certificate success.
I0219 19:40:54.224864   12365 cert.go:246] Generate etcd-client certificate success.
I0219 19:40:54.225086   12365 deploy.go:361] download crds file:https://github.com/karmada-io/karmada/releases/download/v1.8.1/crds.tar.gz
Downloading...[ 100.00% ]
Download complete.
I0219 19:40:56.373735   12365 deploy.go:621] Create karmada kubeconfig success.
I0219 19:40:56.383437   12365 idempotency.go:267] Namespace karmada-system has been created or updated.
I0219 19:40:56.420546   12365 idempotency.go:291] Service karmada-system/etcd has been created or updated.
I0219 19:40:56.420606   12365 deploy.go:427] Create etcd StatefulSets
I0219 19:41:43.446937   12365 deploy.go:436] Create karmada ApiServer Deployment
I0219 19:41:43.461798   12365 idempotency.go:291] Service karmada-system/karmada-apiserver has been created or updated.
I0219 19:42:14.479243   12365 deploy.go:451] Create karmada aggregated apiserver Deployment
I0219 19:42:14.502694   12365 idempotency.go:291] Service karmada-system/karmada-aggregated-apiserver has been created or updated.
I0219 19:42:36.546334   12365 idempotency.go:267] Namespace karmada-system has been created or updated.
I0219 19:42:36.546669   12365 deploy.go:85] Initialize karmada bases crd resource `/etc/karmada/crds/bases`
I0219 19:42:36.548480   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.563038   12365 deploy.go:250] Create CRD cronfederatedhpas.autoscaling.karmada.io successfully.
I0219 19:42:36.569360   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.599014   12365 deploy.go:250] Create CRD federatedhpas.autoscaling.karmada.io successfully.
I0219 19:42:36.600462   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.611881   12365 deploy.go:250] Create CRD resourceinterpretercustomizations.config.karmada.io successfully.
I0219 19:42:36.613298   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.624451   12365 deploy.go:250] Create CRD resourceinterpreterwebhookconfigurations.config.karmada.io successfully.
I0219 19:42:36.628038   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.639566   12365 deploy.go:250] Create CRD serviceexports.multicluster.x-k8s.io successfully.
I0219 19:42:36.640702   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.654625   12365 deploy.go:250] Create CRD serviceimports.multicluster.x-k8s.io successfully.
I0219 19:42:36.658436   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.720222   12365 deploy.go:250] Create CRD multiclusteringresses.networking.karmada.io successfully.
I0219 19:42:36.725982   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.762646   12365 deploy.go:250] Create CRD multiclusterservices.networking.karmada.io successfully.
I0219 19:42:36.783995   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.849103   12365 deploy.go:250] Create CRD clusteroverridepolicies.policy.karmada.io successfully.
I0219 19:42:36.855469   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.901329   12365 deploy.go:250] Create CRD clusterpropagationpolicies.policy.karmada.io successfully.
I0219 19:42:36.903346   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.918197   12365 deploy.go:250] Create CRD federatedresourcequotas.policy.karmada.io successfully.
I0219 19:42:36.925860   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:36.975270   12365 deploy.go:250] Create CRD overridepolicies.policy.karmada.io successfully.
I0219 19:42:36.979416   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:37.171362   12365 deploy.go:250] Create CRD propagationpolicies.policy.karmada.io successfully.
I0219 19:42:37.181269   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:37.377398   12365 deploy.go:250] Create CRD clusterresourcebindings.work.karmada.io successfully.
I0219 19:42:37.386202   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:37.572088   12365 deploy.go:250] Create CRD resourcebindings.work.karmada.io successfully.
I0219 19:42:37.573803   12365 deploy.go:240] Attempting to create CRD
I0219 19:42:37.761955   12365 deploy.go:250] Create CRD works.work.karmada.io successfully.
I0219 19:42:37.762142   12365 deploy.go:96] Initialize karmada patches crd resource `/etc/karmada/crds/patches`
I0219 19:42:38.196299   12365 deploy.go:108] Create MutatingWebhookConfiguration mutating-config.
I0219 19:42:38.211401   12365 webhook_configuration.go:362] MutatingWebhookConfiguration mutating-config has been created or updated successfully.
I0219 19:42:38.211480   12365 deploy.go:113] Create ValidatingWebhookConfiguration validating-config.
I0219 19:42:38.228864   12365 webhook_configuration.go:333] ValidatingWebhookConfiguration validating-config has been created or updated successfully.
I0219 19:42:38.228932   12365 deploy.go:119] Create Service 'karmada-aggregated-apiserver' and APIService 'v1alpha1.cluster.karmada.io'.
I0219 19:42:38.236205   12365 idempotency.go:291] Service karmada-system/karmada-aggregated-apiserver has been created or updated.
I0219 19:42:38.246742   12365 check.go:42] Waiting for APIService(v1alpha1.cluster.karmada.io) condition(Available), will try
I0219 19:42:39.285601   12365 tlsbootstrap.go:49] [bootstrap-token] configured RBAC rules to allow Karmada Agent Bootstrap tokens to post CSRs in order for agent to get long term certificate credentials
I0219 19:42:39.289988   12365 tlsbootstrap.go:63] [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Karmada Agent Bootstrap Token
I0219 19:42:39.295954   12365 tlsbootstrap.go:77] [bootstrap-token] configured RBAC rules to allow certificate rotation for all agent client certificates in the member cluster
I0219 19:42:39.457588   12365 deploy.go:143] Initialize karmada bootstrap token
I0219 19:42:39.465962   12365 deploy.go:469] Create karmada kube controller manager Deployment
I0219 19:42:39.477854   12365 idempotency.go:291] Service karmada-system/kube-controller-manager has been created or updated.
I0219 19:42:56.498945   12365 deploy.go:483] Create karmada scheduler Deployment
I0219 19:43:14.519491   12365 deploy.go:494] Create karmada controller manager Deployment
I0219 19:43:35.537095   12365 deploy.go:505] Create karmada webhook Deployment
I0219 19:43:35.555681   12365 idempotency.go:291] Service karmada-system/karmada-webhook has been created or updated.

------------------------------------------------------------------------------------------------------
 █████   ████   █████████   ███████████   ██████   ██████   █████████   ██████████     █████████
░░███   ███░   ███░░░░░███ ░░███░░░░░███ ░░██████ ██████   ███░░░░░███ ░░███░░░░███   ███░░░░░███
 ░███  ███    ░███    ░███  ░███    ░███  ░███░█████░███  ░███    ░███  ░███   ░░███ ░███    ░███
 ░███████     ░███████████  ░██████████   ░███░░███ ░███  ░███████████  ░███    ░███ ░███████████
 ░███░░███    ░███░░░░░███  ░███░░░░░███  ░███ ░░░  ░███  ░███░░░░░███  ░███    ░███ ░███░░░░░███
 ░███ ░░███   ░███    ░███  ░███    ░███  ░███      ░███  ░███    ░███  ░███    ███  ░███    ░███
 █████ ░░████ █████   █████ █████   █████ █████     █████ █████   █████ ██████████   █████   █████
░░░░░   ░░░░ ░░░░░   ░░░░░ ░░░░░   ░░░░░ ░░░░░     ░░░░░ ░░░░░   ░░░░░ ░░░░░░░░░░   ░░░░░   ░░░░░
------------------------------------------------------------------------------------------------------
Karmada is installed successfully.

Register Kubernetes cluster to Karmada control plane.

Register cluster with 'Push' mode

Step 1: Use "kubectl karmada join" command to register the cluster to Karmada control plane. --cluster-kubeconfig is kubeconfig of the member cluster.
(In karmada)~# MEMBER_CLUSTER_NAME=$(cat ~/.kube/config  | grep current-context | sed 's/: /\n/g'| sed '1d')
(In karmada)~# kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config  join ${MEMBER_CLUSTER_NAME} --cluster-kubeconfig=$HOME/.kube/config

Step 2: Show members of karmada
(In karmada)~# kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get clusters


Register cluster with 'Pull' mode

Step 1: Use "kubectl karmada register" command to register the cluster to Karmada control plane. "--cluster-name" is set to cluster of current-context by default.
(In member cluster)~# kubectl karmada register 172.18.0.2:32443 --token 2dd1pt.xec194hxpjhz4v8j --discovery-token-ca-cert-hash sha256:6937d47f9b852e53e0fb0c035a6166992afc30f9332e1b1d2ee638664b4d6aca

Step 2: Show members of karmada
(In karmada)~# kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get clusters
```


## 集群注册

Karmada 支持 Push 和 Pull 两种模式来管理成员集群。 Push 和 Pull 模式的主要区别在于部署清单时访问成员集群的方式。

### PUSH Mode

Karmada 控制平面将直接访问成员集群的 kube-apiserver 以获取集群状态并部署清单。

#### 注册

```bash
kubectl karmada join kind-cluster01 --kubeconfig=/etc/karmada/karmada-apiserver.config --cluster-kubeconfig=/root/.kube/config --cluster-context=kind-cluster01
cluster(kind-cluster01) is joined successfully
kubectl karmada join kind-cluster02 --kubeconfig=/etc/karmada/karmada-apiserver.config --cluster-kubeconfig=/root/.kube/config --cluster-context=kind-cluster02
cluster(kind-cluster02) is joined successfully
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get clusters
NAME             VERSION   MODE   READY   AGE
kind-cluster01   v1.27.3   Push   True    16s
kind-cluster02   v1.27.3   Push   True    5s
```


### Pull mode

Karmada 控制平面不会访问成员集群，而是将其委托给名为 Karmada-agent 的额外组件。

每个 karmada-agent 服务于一个集群并负责：

* 将集群注册到 Karmada（创建 Cluster 对象）
* 维护集群状态并向 Karmada 报告（更新集群对象的状态）
* 从 Karmada 执行空间（命名空间、karmada-es-<集群名称>）监视清单，并将监视的资源部署到代理服务的集群。

## 多集群调度
> https://karmada.io/zh/docs/userguide/scheduling/resource-propagating#fieldselector

提供 [PropagationPolicy](https://github.com/karmada-io/karmada/blob/master/pkg/apis/policy/v1alpha1/propagation_types.go#L13) 和 [ClusterPropagationPolicy](https://github.com/karmada-io/karmada/blob/master/pkg/apis/policy/v1alpha1/propagation_types.go#L292) API 来传播资源。关于两个API之间的差异，请参见[此处](https://karmada.io/zh/docs/faq/#what-is-the-difference-between-propagationpolicy-and-clusterpropagationpolicy)

### 部署最简单的多集群Deployment

#### 创建传播策略

```bash
cat <<EOF > propagationpolicy.yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: propagaetion-nginx # The default namespace is `default`.
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx # If no namespace is specified, the namespace is inherited from the parent object scope.
  placement:
    clusterAffinity:
      clusterNames:
        - kind-cluster01
EOF

kubectl --kubeconfig /etc/karmada/karmada-apiserver.config apply -f propagationpolicy.yaml
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config create deployment nginx --image nginx

kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get deployment
NAME    CLUSTER          READY   UP-TO-DATE   AVAILABLE   AGE     ADOPTION
nginx   kind-cluster01   1/1     1            1           3m36s   Y

kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get pods
NAME                     CLUSTER          READY   STATUS    RESTARTS   AGE
nginx-77b4fdf86c-zxjp2   kind-cluster01   1/1     Running   0          3m36s

kubectl --context kind-cluster01 get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-77b4fdf86c-zxjp2   1/1     Running   0          2m36s

kubectl --context kind-cluster02 get pods
No resources found in default namespace.
```

#### 更新传播策略

可以通过应用新的 YAML 文件来更新 propagationPolicy。此 YAML 文件将部署传播到 kind-cluster02 集群。

```bash
cat <<EOF > propagationpolicy-update.yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: propagaetion-nginx
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames: # Modify the selected cluster to propagate the Deployment.
        - kind-cluster02
EOF

kubectl --kubeconfig /etc/karmada/karmada-apiserver.config apply -f propagationpolicy-update.yaml

kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get deployment
NAME    CLUSTER          READY   UP-TO-DATE   AVAILABLE   AGE    ADOPTION
nginx   kind-cluster02   1/1     1            1           112s   Y

kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get pods
NAME                     CLUSTER          READY   STATUS    RESTARTS   AGE
nginx-77b4fdf86c-jk5qm   kind-cluster02   1/1     Running   0          99s

kubectl --context kind-cluster02 get pods 
NAME                     READY   STATUS    RESTARTS   AGE
nginx-77b4fdf86c-jk5qm   1/1     Running   0          2m13s
```

### 将部署部署到指定的一组目标集群中

PropagationPolicy 的 `.spec.placement.clusterAffinity` 字段表示对特定集群集合的调度限制，没有该限制，任何集群都可以成为调度候选者。

它有四个字段可以设置：
* LabelSelector
* FieldSelector
* ClusterNames
* ExcludeClusters

#### LabelSelector
> 支持matchLabels和matchExpressions， 区别可以参考[支持基于集合需求的资源](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/#%E6%94%AF%E6%8C%81%E5%9F%BA%E4%BA%8E%E9%9B%86%E5%90%88%E9%9C%80%E6%B1%82%E7%9A%84%E8%B5%84%E6%BA%90)

##### 配置标签

```bash
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config label cluster kind-cluster01 app=cluster01
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config label cluster kind-cluster02 app=cluster02
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config label cluster kind-cluster01 deploy-type=kind
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config label cluster kind-cluster02 deploy-type=kind
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config get cluster --show-labels
NAME             VERSION   MODE   READY   AGE   LABELS
kind-cluster01   v1.27.3   Push   True    10h   app=cluster01,deploy-type=kind
kind-cluster02   v1.27.3   Push   True    10h   app=cluster02,deploy-type=kind
```

##### matchLabels
```bash
cat <<EOF > propagationpolicy-labelSelector-matchLabels.yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: propagaetion-nginx
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      labelSelector:
        matchLabels:
          app: cluster01
EOF
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config apply -f propagationpolicy-labelSelector-matchLabels.yaml
propagationpolicy.policy.karmada.io/propagaetion-nginx configured
```

查看集群deployment部署情况
```bash
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get deployments.apps
NAME    CLUSTER          READY   UP-TO-DATE   AVAILABLE   AGE   ADOPTION
nginx   kind-cluster01   2/2     2            2           13s   Y
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get pods            
NAME                     CLUSTER          READY   STATUS    RESTARTS   AGE
nginx-77b4fdf86c-s7qzm   kind-cluster01   1/1     Running   0          48s
nginx-77b4fdf86c-v28n5   kind-cluster01   1/1     Running   0          48s
```

##### matchExpressions
```bash
cat <<EOF > propagationpolicy-labelSelector-matchExpressions.yaml 
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: propagaetion-nginx
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - cluster01
          - cluster02
EOF
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config apply -f propagationpolicy-labelSelector-matchExpressions.yaml
propagationpolicy.policy.karmada.io/propagaetion-nginx configured
```

查看集群deployment部署情况
```bash
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get deployments.apps
NAME    CLUSTER          READY   UP-TO-DATE   AVAILABLE   AGE     ADOPTION
nginx   kind-cluster01   2/2     2            2           3m34s   Y
nginx   kind-cluster02   2/2     2            2           23s     Y
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get pods
NAME                     CLUSTER          READY   STATUS    RESTARTS   AGE
nginx-77b4fdf86c-s7qzm   kind-cluster01   1/1     Running   0          3m40s
nginx-77b4fdf86c-v28n5   kind-cluster01   1/1     Running   0          3m40s
nginx-77b4fdf86c-dvzfx   kind-cluster02   1/1     Running   0          29s
nginx-77b4fdf86c-m6brd   kind-cluster02   1/1     Running   0          29s
```

#### FieldSelector
> 支持matchLabels和matchExpressions， 区别可以参考[支持基于集合需求的资源](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/#%E6%94%AF%E6%8C%81%E5%9F%BA%E4%BA%8E%E9%9B%86%E5%90%88%E9%9C%80%E6%B1%82%E7%9A%84%E8%B5%84%E6%BA%90)

##### 配置Field

> FieldSelector 支持三个值`provider`, `region`, `zone`, 分别对应cluster对象的`.spec.provider`, `.spec.region`, `.spec.zone`的选择

```bash
bectl --kubeconfig /etc/karmada/karmada-apiserver.config patch clusters.cluster.karmada.io kind-cluster01 kind-cluster02 -p '{"spec": {"provider": "kind"} }' 
cluster.cluster.karmada.io/kind-cluster01 patched
cluster.cluster.karmada.io/kind-cluster02 patched

kubectl --kubeconfig /etc/karmada/karmada-apiserver.config patch clusters.cluster.karmada.io kind-cluster01 kind-cluster02 -p '{"spec": {"region": "local"} }'
cluster.cluster.karmada.io/kind-cluster01 patched
cluster.cluster.karmada.io/kind-cluster02 patched

kubectl --kubeconfig /etc/karmada/karmada-apiserver.config patch clusters.cluster.karmada.io kind-cluster01 -p '{"spec": {"zone": "cluster01"} }'
cluster.cluster.karmada.io/kind-cluster01 patched
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config patch clusters.cluster.karmada.io kind-cluster02 -p '{"spec": {"zone": "cluster02"} }'
cluster.cluster.karmada.io/kind-cluster02 patched
```

#####  matchExpressions

```bash
cat <<EOF > propagationpolicy-FieldSelector-matchExpressions.yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: propagaetion-nginx
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      fieldSelector:
        matchExpressions:
        - key: provider
          operator: In
          values:
          - kind
        - key: region
          operator: In
          values:
          - local
        - key: zone
          operator: In
          values:
          - cluster02
EOF
kubectl --kubeconfig /etc/karmada/karmada-apiserver.config apply -f propagationpolicy-FieldSelector-matchExpressions.yaml
propagationpolicy.policy.karmada.io/propagaetion-nginx configured
```
如果在`fieldSelector`中指定了多个`matchExpression`，则簇必须匹配所有`matchExpression`。

查看集群deployment部署情况

```bash
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get deployment
NAME    CLUSTER          READY   UP-TO-DATE   AVAILABLE   AGE   ADOPTION
nginx   kind-cluster02   2/2     2            2           12m   Y
kubectl karmada --kubeconfig /etc/karmada/karmada-apiserver.config get pods
NAME                     CLUSTER          READY   STATUS    RESTARTS   AGE
nginx-77b4fdf86c-dvzfx   kind-cluster02   1/1     Running   0          13m
nginx-77b4fdf86c-m6brd   kind-cluster02   1/1     Running   0          13m
```

### 多个集群亲和性组


