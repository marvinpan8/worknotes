# k0s集群动态配置开关

参考官网：https://docs.k0sproject.io/v1.33.2+k0s.0/dynamic-configuration/

​	k0s controller 命令附带启用集群级组件动态配置的选项。这涵盖了除 **etcd** 和 **Kubernetes api-server** 之外的所有组件。此选项允许直接通过 Kubernetes API 进行 k0s 配置，而无需使用配置文件进行所有集群配置。

## --enable-dynamic-config

​	使用 `k0s controller`  或  `k0s install controller` 命令时加上  **--enable-dynamic-config**  标识将打开每一个控制器的动态配置开关。若一个集群中有2种类型的控制器将会发生冲突。

## 动态与静态配置

​	现有默认的启用方法就是静态配置。k0s 进程会从给定的 YAML 文件读取配置（如果用户未提供，则使用默认配置），并相应地配置每个组件。**这意味着，对于任何配置更改，集群管理员都必须重新启动集群上的所有控制器，并在每个控制器节点上进行匹配的配置。**
​	
​	在动态配置模式下，集群创建时启动的第一个控制器将使用给定的**配置 YAML 作为引导配置，并将其存储在 Kubernetes API 中。**所有其他控制器将在 API 上查找该配置，并将其作为配置除 etcd 和 kube-apiserver 之外所有组件的真实来源。**集群初始引导后，所有控制器的真实来源都是 Kubernetes API 中的配置对象。**


## 控制节点配置

[	在k0s 配置选项](https://docs.k0sproject.io/v1.33.2+k0s.0/configuration/)中，有些选项是集群范围的，有些选项是集群中每个控制节点**独有的**。以下列表是特定于控制节点的，并且**只能通过本地文件进行配置**：

- `spec.api`- 这是本地 Kubernetes API 服务器的启动配置
- `spec.storage`- 这是本地存储（etcd 或 sqlite）的启动配置
- `spec.network.controlPlaneLoadBalancing`- 这是[控制平面负载平衡](https://docs.k0sproject.io/v1.33.2+k0s.0/cplb/)的启动配置

对于 HA高可用控制平面，所有控制器都需要以上配置，否则它们将无法运行存储和 Kubernetes API 服务器。

## 动态配置位置

​	集群配置存储在名为`clusterconfig` 的 Kubernetes API 自定义资源中。目前只有一个名为`k0s` 的实例。您可以使用以下命令编辑该配置：

```properties
k0s config edit
```

## 动态配置探测执行

​	动态配置使用典型的操作符模式进行操作。**k0s 控制器会检测对象何时发生变化，并协调配置变更**，使其反映到不同组件的配置方式中。因此，假设您要更改 kube-router CNI 网络的 MTU 设置，您可以修改配置，使其包含以下内容：


```properties
kuberouter:
  mtu: 1350
  autoMTU: false
```

这将改变 kube-router 相关的配置映射，从而使 kube-router **对新 pod 使用不同的 MTU 设置。**

## 不可变更的配置

​	除 `spec.api` 和 `spec.storage`以外，配置对象与 YAML文件都是一对一映射的。

与任何 Kubernetes 集群一样，有的配置是无法**即时**更改的，具体如下：

- `network.podCIDR`
- `network.serviceCIDR`
- `network.provider`
- `network.controlPlaneLoadBalancing`

在使用 `k0s install`命令手动安装控制平面节点时，**所有这些不可更改的选项都必须在配置文件中定义**。这是因为这些字段可以在动态配置协调器初始化之前使用。k0sctl 和 k0smotron 都可以在无需用户干预的情况下处理此问题。

## 动态配置变更事件

动态配置执行器将检测到所有变更事件，查看命令如下：

```properties
k0s config status
# ------------------------------------------------
LAST SEEN   TYPE      REASON                OBJECT              MESSAGE
64s         Warning   FailedReconciling     clusterconfig/k0s   failed to validate config: [invalid pod CIDR invalid ip address]
59s         Normal    SuccessfulReconcile   clusterconfig/k0s   Successfully reconciler cluster config
69s         Warning   FailedReconciling     clusterconfig/k0s   cannot change CNI provider from kuberouter to calico
```