## config

config/default 包含 Kustomize base 文件，用于以标准配置启动 controller。

config/目录下每个目录都包含不同的配置：

config/manager: 包含在 k8s 集群中以 pod 形式运行 controller 的 YAML 配置文件

config/rbac: 包含运行 controller 所需最小权限的配置文件

## main.go:

每组 controller 都需要一个 Scheme， Scheme 会提供 Kinds 与 Go types 之间的映射关系
我们 main.go 的功能相对来说比较简单：

为 metrics 绑定一些基本的 flags。

实例化一个 manager，用于跟踪我们运行的所有 controllers， 并设置 shared caches 和可以连接到 API server 的 k8s clients 实例，并将 Scheme 配置传入 manager。

运行我们的 manager, 而 manager 又运行所有的 controllers 和 webhook。 manager 会一直处于运行状态，直到收到正常关闭信号为止。 这样，当我们的 operator 运行在 Kubernetes 上时，我们可以通过优雅的方式终止这个 Pod。
