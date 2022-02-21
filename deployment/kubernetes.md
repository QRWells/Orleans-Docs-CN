---
title: Kubernetes托管
---

Kubernetes是托管Orleans应用程序的一个流行选择。
Orleans将在Kubernetes中运行，不需要特定的配置，它也可以利用托管平台可以提供的额外信息。

[`Microsoft.Orleans.Hosting.Kubernetes`](https://www.nuget.org/packages/Microsoft.Orleans.Hosting.Kubernetes)包增加了在Kubernetes集群中托管Orleans应用程序的集成。该包提供了一个扩展方法，`ISiloBuilder.UseKubernetesHosting`，它可以执行以下操作：

* `SiloOptions.SiloName`设置为Pod名称
* `EndpointOptions.AdvertisedIPAddress`设置为Pod的IP
* `EndpointOptions.SiloListeningEndpoint`与`EndpointOptions.GatewayListeningEndpoint`配置为监听任意地址，各自的端口号通过`SiloPort`与`GatewayPort`配置。如果没有显式指定端口号，则默认为`11111`和`30000`
* `ClusterOptions.ServiceId`设置为名为`orleans/serviceId`的pod标签的值。
* `ClusterOptions.ClusterId`设置为名为`orleans/clusterId`的pod标签的值。
* 在启动过程的初期，Silo将探测Kubernetes，以发现哪些Silo没有相应的pod，并将这些Silo标记为死亡。
* 同样的过程将在运行时发生在所有Silo的一个子集上，以消除Kubernetes的API服务器的负载。默认情况下，集群中的2个Silo将用于监视Kubernetes。

注意，Kubernetes托管包并不使用Kubernetes进行集群化。对于集群化，仍然需要一个单独的集群提供者。关于配置集群的更多信息，请参阅[服务器配置](../host/configuration_guide/server_configuration.md)。

这种功能对服务的部署方式提出了一些要求：

* Silo名称必须匹配Pod名称.
* Pod必须有`orleans/serviceId`和`orleans/clusterId`标签，与Silo的`ServiceId`和`ClusterId`对应。上述方法将把这些标签传播到Orleans环境变量的相应选项中。
* Pods必须设置以下环境变量：`POD_NAME`、`POD_NAMESPACE`、`POD_IP`、`ORLEANS_SERVICE_ID`、`ORLEANS_CLUSTER_ID`。

下面的例子展示了如何正确配置这些标签和环境变量：

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dictionary-app
  labels:
    orleans/serviceId: dictionary-app
spec:
  selector:
    matchLabels:
      orleans/serviceId: dictionary-app
  replicas: 3
  template:
    metadata:
      labels:
        # This label is used to identify the service to Orleans
        orleans/serviceId: dictionary-app

        # This label is used to identify an instance of a cluster to Orleans.
        # Typically, this will be the same value as the previous label, or any 
        # fixed value.
        # In cases where you are not using rolling deployments (for example,
        # blue/green deployments),
        # this value can allow for distinct clusters which do not communicate
        # directly with each others,
        # but which still share the same storage and other resources.
        orleans/clusterId: dictionary-app
    spec:
      containers:
        - name: main
          image: my-registry.azurecr.io/my-image
          imagePullPolicy: Always
          ports:
          # Define the ports which Orleans uses
          - containerPort: 11111
          - containerPort: 30000
          env:
          # The Azure Storage connection string for clustering is injected as an
          # environment variable
          # It must be created separately using a command such as:
          # > kubectl create secret generic az-storage-acct `
          #     --from-file=key=./az-storage-acct.txt
          - name: STORAGE_CONNECTION_STRING
            valueFrom:
              secretKeyRef:
                name: az-storage-acct
                key: key
          # Configure settings to let Orleans know which cluster it belongs to
          # and which pod it is running in
          - name: ORLEANS_SERVICE_ID
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['orleans/serviceId']
          - name: ORLEANS_CLUSTER_ID
            valueFrom:
              fieldRef:
                fieldPath: metadata.labels['orleans/clusterId']
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
          - name: DOTNET_SHUTDOWNTIMEOUTSECONDS
            value: "120"
          request:
            # Set resource requests
      terminationGracePeriodSeconds: 180
      imagePullSecrets:
        - name: my-image-pull-secret
  minReadySeconds: 60
  strategy:
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

对于支持RBAC的集群，Kubernetes服务账户的pod可能也需要被授予所需的权限：

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader
rules:
- apiGroups: [ "" ]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pod-reader-binding
subjects:
- kind: ServiceAccount
  name: default
  apiGroup: ''
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: ''
```

## 存活、就绪和启动探测器

Kubernetes能够探测Pod以确定服务的健康情况。更多信息，参见Kubernetes文档的[配置存活、就绪和启动探测器](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)。

Orleans使用集群成员协议来及时检测和恢复进程或网络故障。
每个节点监控其他节点的一个子集，定期发送探测器。
如果一个节点未能对来自其他多个节点的连续探测作出反应，那么它将被强行从集群中移除。
一旦一个故障的节点得知自己被移除，它就会立即终止工作。
Kubernetes将重新启动被终止的进程，它将试图重新加入集群。

Kubernetes的探测器可以帮助确定一个pod中的进程是否在执行，而不是停留在僵尸状态。探测器不会验证pod间的连接性或响应性，也不会执行任何应用层面的功能检查。
如果一个pod未能响应一个有效性探测器，那么Kubernetes最终可能会终止该pod并重新调度它。
因此，Kubernetes的探测器和Orleans的探测器是互补的。

推荐的方法是在Kubernetes中配置存活探测器，它只执行简单的本地检查，以确保应用程序按预期执行。
这些探测器的作用是在出现完全冻结的情况下终止进程，例如由于运行时故障或其他小概率事件。

## 资源配额

Kubernetes与操作系统一起工作，实现[资源配额](https://kubernetes.io/zh/docs/concepts/policy/resource-quotas/)。
这允许CPU和内存保留和/或限制被强制执行。
对于一个为交互式负载服务的主要应用，我们建议除非必要，否则不要实施限制性的上限。
需要注意的是，请求和限制在其含义和实施方式上有很大不同。
在设置请求或限制之前，要花时间详细了解它们是如何实现和执行的。
例如，Kubernetes、Linux内核和你的监控系统之间可能不会统一测量内存。CPU配额可能不会以你期望的方式执行。

## 故障诊断

### Pods崩溃，报错信息为：`KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT must be defined`

异常信息全文： 
``` 
Unhandled exception. k8s.Exceptions.KubeConfigException: unable to load in-cluster configuration, KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT must be defined
at k8s.KubernetesClientConfiguration.InClusterConfig()
```

* 检查`KUBERNETES_SERVICE_HOST`和`KUBERNETES_SERVICE_PORT`环境变量是否已在你的Pod内设置。你可以通过执行以下命令来检查`kubectl exec -it <pod_name> /bin/bash -c env`。
* 确保在你的Kubernetes`deployment.yaml`上将`automountServiceAccountToken`设置为**true**。欲了解更多信息，请参阅[为Pod配置服务账户](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-service-account/)。
