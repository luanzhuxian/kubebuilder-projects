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

.PHONY: manifests
manifests: controller-gen ## Generate WebhookConfiguration, ClusterRole and CustomResourceDefinition objects.
$(CONTROLLER_GEN) rbac:roleName=manager-role crd:maxDescLen=0 webhook paths="./..." output:crd:artifacts:config=config/crd/bases

.PHONY: kustomize
kustomize: $(KUSTOMIZE) ## Download kustomize locally if necessary.
$(KUSTOMIZE): $(LOCALBIN)
	$(call go-install-tool,$(KUSTOMIZE),sigs.k8s.io/kustomize/kustomize/v5,$(KUSTOMIZE_VERSION))

.PHONY: controller-gen
controller-gen: $(CONTROLLER_GEN) ## Download controller-gen locally if necessary.
$(CONTROLLER_GEN): $(LOCALBIN)
	$(call go-install-tool,$(CONTROLLER_GEN),sigs.k8s.io/controller-tools/cmd/controller-gen,$(CONTROLLER_TOOLS_VERSION))

// 只安装 CRDs，为自定义资源做准备
.PHONY: install
install: manifests kustomize ## Install CRDs into the K8s cluster specified in ~/.kube/config.
$(KUSTOMIZE) build config/crd | $(KUBECTL) apply -f -

// 首先设置控制器的镜像名称
// 使用 kustomize 构建 config/default 目录中的所有配置
// 通过 kubectl 应用到集群中
// 结果：部署完整的控制器应用，包括：Deployment（控制器 Pod） ServiceAccount RBAC 权限 CRDs 其他相关资源
.PHONY: deploy
deploy: manifests kustomize ## Deploy controller to the K8s cluster specified in ~/.kube/config.
cd config/manager && $(KUSTOMIZE) edit set image controller=${IMG}
$(KUSTOMIZE) build config/default | $(KUBECTL) apply -f -
