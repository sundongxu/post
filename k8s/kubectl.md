---
title: "Kubectl wiki"
date: 2019-06-17T23:45:12+08:00
categories:
    - Knowledge
tags: 
    - kubernetes
    - k8s
    - kubectl
---

基于 kubernetes v1.14.3 版本

## 1. basic

kubectl 命令行语法:

`$ kubectl [command] [TYPE] [NAME] [flags]`

1) command: 子命令，用于操作 K8s 集群资源对象的命令，例如 `create`/`delete`/`describe`/`get`/`apply`

2) TYPE: 资源对象的名称，区分大小写，能以单数形式、复数形式或者简写形式表示

> 以下 3 种 TYPE 是等价的:
> - kubectl get pod pod1
> - kubectl get pods pod1
> - kubectl get po pod1

3) NAME: 资源对象的名称，区分大小写。如果不指定名称，则系统将返回属于 TYPE 的全部对象列表

4) flags: kubectl 子命令的可选参数

获取 kubectl 可操作的资源对象列表: `kubectl api-resources`


NAME                              | SHORTNAMES   | APIGROUP                       | NAMESPACED   | KIND
| --- | --- | --- | --- | --- |
bindings                          |              |                                | true         | Binding
componentstatuses                 | cs           |                                | false        | ComponentStatus
configmaps                        | cm           |                                | true         | ConfigMap
endpoints                         | ep           |                                | true         | Endpoints
events                            | ev           |                                | true         | Event
limitranges                       | limits       |                                | true         | LimitRange
namespaces                        | ns           |                                | false        | Namespace
nodes                             | no           |                                | false        | Node
persistentvolumeclaims            | pvc          |                                | true         | PersistentVolumeClaim
persistentvolumes                 | pv           |                                | false        | PersistentVolume
pods                              | po           |                                | true         | Pod
podtemplates                      |              |                                | true         | PodTemplate
replicationcontrollers            | rc           |                                | true         | ReplicationController
resourcequotas                    | quota        |                                | true         | ResourceQuota
secrets                           |              |                                | true         | Secret
serviceaccounts                   | sa           |                                | true         | ServiceAccount
services                          | svc          |                                | true         | Service
mutatingwebhookconfigurations     |              | admissionregistration.k8s.io   | false        | MutatingWebhookConfiguration
validatingwebhookconfigurations   |              | admissionregistration.k8s.io   | false        | ValidatingWebhookConfiguration
customresourcedefinitions         | crd,crds     | apiextensions.k8s.io           | false        | CustomResourceDefinition
apiservices                       |              | apiregistration.k8s.io         | false        | APIService
controllerrevisions               |              | apps                           | true         | ControllerRevision
daemonsets                        | ds           | apps                           | true         | DaemonSet
deployments                       | deploy       | apps                           | true         | Deployment
replicasets                       | rs           | apps                           | true         | ReplicaSet
statefulsets                      | sts          | apps                           | true         | StatefulSet
tokenreviews                      |              | authentication.k8s.io          | false        | TokenReview
localsubjectaccessreviews         |              | authorization.k8s.io           | true         | LocalSubjectAccessReview
selfsubjectaccessreviews          |              | authorization.k8s.io           | false        | SelfSubjectAccessReview
selfsubjectrulesreviews           |              | authorization.k8s.io           | false        | SelfSubjectRulesReview
subjectaccessreviews              |              | authorization.k8s.io           | false        | SubjectAccessReview
horizontalpodautoscalers          | hpa          | autoscaling                    | true         | HorizontalPodAutoscaler
cronjobs                          | cj           | batch                          | true         | CronJob
jobs                              |              | batch                          | true         | Job
certificatesigningrequests        | csr          | certificates.k8s.io            | false        | CertificateSigningRequest
stacks                            |              | compose.docker.com             | true         | Stack
leases                            |              | coordination.k8s.io            | true         | Lease
events                            | ev           | events.k8s.io                  | true         | Event
daemonsets                        | ds           | extensions                     | true         | DaemonSet
deployments                       | deploy       | extensions                     | true         | Deployment
ingresses                         | ing          | extensions                     | true         | Ingress
networkpolicies                   | netpol       | extensions                     | true         | NetworkPolicy
podsecuritypolicies               | psp          | extensions                     | false        | PodSecurityPolicy
replicasets                       | rs           | extensions                     | true         | ReplicaSet
ingresses                         | ing          | networking.k8s.io              | true         | Ingress
networkpolicies                   | netpol       | networking.k8s.io              | true         | NetworkPolicy
runtimeclasses                    |              | node.k8s.io                    | false        | RuntimeClass
poddisruptionbudgets              | pdb          | policy                         | true         | PodDisruptionBudget
podsecuritypolicies               | psp          | policy                         | false        | PodSecurityPolicy
clusterrolebindings               |              | rbac.authorization.k8s.io      | false        | ClusterRoleBinding
clusterroles                      |              | rbac.authorization.k8s.io      | false        | ClusterRole
rolebindings                      |              | rbac.authorization.k8s.io      | true         | RoleBinding
roles                             |              | rbac.authorization.k8s.io      | true         | Role
priorityclasses                   | pc           | scheduling.k8s.io              | false        | PriorityClass
csidrivers                        |              | storage.k8s.io                 | false        | CSIDriver
csinodes                          |              | storage.k8s.io                 | false        | CSINode
storageclasses                    | sc           | storage.k8s.io                 | false        | StorageClass
volumeattachments                 |              | storage.k8s.io                 | false        | VolumeAttachment

在一个命令行可以同时对多个资源对象进行操作，以多个 TYPE 和 NAME 的组合表示，示例:

* 获取多个 Pod 的信息: `kubectl get pods pod1 pod2`
* 获取多个 Pod 的信息: `kubectl get pod/pod1 pod/pod2`
* 同时应用多个 yaml 文件，以多个 `-f file` 参数表示: `kubectl get pod -f pod1.yaml -f pod2.yaml`

kubectl 的子命令非常丰富，可以直接输入 `kubectl` 获取子命令的基本使用帮助:

command | help
| --- | --- |
Basic Commands (Beginner): |
  create         | Create a resource from a file or from stdin.
  expose         | 使用 replication controller, service, deployment 或者 pod 并暴露它作为一个 新的
|  |  |
Kubernetes Service| 
  run            | 在集群中运行一个指定的镜像
  set            | 为 objects 设置一个指定的特征
|  |  |
Basic Commands (Intermediate): |
  explain        | 查看资源的文档
  get            | 显示一个或更多 resources
  edit           | 在服务器上编辑一个资源
  delete         | Delete resources by filenames, stdin, resources and names, or by resources and label selector
|  |  |
Deploy Commands: |
  rollout        | Manage the rollout of a resource
  scale          | 为 Deployment, ReplicaSet, Replication Controller 或者 Job 设置一个新的副本数量
  autoscale      | 自动调整一个 Deployment, ReplicaSet, 或者 ReplicationController 的副本数量
|  |  |
Cluster Management Commands: |
  certificate    | 修改 certificate 资源.
  cluster-info   | 显示集群信息
  top            | Display Resource (CPU/Memory/Storage) usage.
  cordon         | 标记 node 为 unschedulable
  uncordon       | 标记 node 为 schedulable
  drain          | Drain node in preparation for maintenance
  taint          | 更新一个或者多个 node 上的 taints
|  |  |
Troubleshooting and Debugging Commands: |
  describe       | 显示一个指定 resource 或者 group 的 resources 详情
  logs           | 输出容器在 pod 中的日志
  attach         | Attach 到一个运行中的 container
  exec           | 在一个 container 中执行一个命令
  port-forward   | Forward one or more local ports to a pod
  proxy          | 运行一个 proxy 到 Kubernetes API server
  cp             | 复制 files 和 directories 到 containers 和从容器中复制 files 和 directories.
  auth           | Inspect authorization
|  |  |
Advanced Commands: |
  diff           | Diff live version against would-be applied version
  apply          | 通过文件名或标准输入流(stdin)对资源进行配置
  patch          | 使用 strategic merge patch 更新一个资源的 field(s)
  replace        | 通过 filename 或者 stdin替换一个资源
  wait           | Experimental: Wait for a specific condition on one or many resources.
  convert        | 在不同的 API versions 转换配置文件
  kustomize      | Build a kustomization target from a directory or a remote url.
|  |  |
Settings Commands: |
  label          | 更新在这个资源上的 labels
  annotate       | 更新一个资源的注解
  completion     | Output shell completion code for the specified shell (bash or zsh)
|  |  |
Other Commands: |
  api-resources  | Print the supported API resources on the server
  api-versions   | Print the supported API versions on the server, in the form of "group/version"
  config         | 修改 kubeconfig 文件
  plugin         | Provides utilities for interacting with plugins.
  version        | 输出 client 和 server 的版本信息


kubectl 公共命令行参数列表帮助可以通过 `kubectl options` 进行查看

kubectl 子命令的特定 flags 可以通过 `kubectl [command] --help` 进行查看

kubectl 输出格式通过 `-o` 参数指定:

```
kubectl [command] [TYPE] [NAME] -o=<output_format>
```

根据不同子命令的输出结果，可选的输出格式如表:

| 输出格式 | 说明 |
| --- | --- |
| -o=custom-columns=(spec) | 根据自定义列名进行输出，以逗号分隔 |
| -o=custom-columns-file=(filename) | 从文件中获取自定义列名进行输出 |
| -o=json | 以 JSON 格式显示结果 |
| -o=jsonpath=(template) | 输出 jsonpath 表达式定义的字段信息 |
| -o=jsonpath-file=(filename) | 输出 jsonpath 表达式定义的字段信息，来源于文件 |
| -o=name | 仅输出资源对象的名称 |
| -o=wide | 输出额外信息。对于 Pod，输出 Pod 所在的 Node 名 |
| -o=yaml | 以 yaml 形式显示结果 |

常用的输出格式示例如下:

1) 显示 Pod 更多信息:
`kubectl get pod <pod-name> -o wide`

2) 以 yaml 格式显示 Pod 详细信息: 
`kubectl get pod <pod-name> -o yaml`

3) 以自定义列名显示 Pod 信息: 
`kubectl get pod <pod-name> -o=custom-columns=NAME:.metadata.name,RSRC:.metadata.recoursceVersion`

4) 将输出结果按某个字段排序，通过 `--sort-by` 参数以 jsonpath 表达式进行指定: `kubectl [command] [TYPE] [NAME] --sort-by=<jsonpath_exp>`
> 例如按照名字进行排序: kubectl get pods --sort-by=.metadata.name 
