# app_redis-cluster

Redis Cluster 离线交付仓库。

这个仓库不是只放一个 Redis Helm chart，而是把下面几件事一起打包成了 `.run` 安装器：

- 镜像准备
- Helm 安装
- 监控接入
- 多架构离线交付

整体范式和 MySQL、MinIO、Milvus、RabbitMQ、MongoDB 保持一致，目标是让一个没有背景信息的新接手同学，或者一个普通 AI，也能按 README 独立完成安装和验证。

## 这套安装器是怎么设计的

普通使用者可以把它理解成一个“Redis Cluster 离线安装器”，核心只有 4 个动作：

- `install`
- `status`
- `uninstall`
- `help`

其中 `install` 会自动完成：

1. 解包 `.run` 里的 chart、镜像元数据和镜像 tar
2. 按目标仓库地址准备 Redis、exporter、辅助镜像
3. 检查集群里是否支持 `ServiceMonitor`
4. 生成最终 Helm 参数
5. 执行 `helm upgrade --install`
6. 输出 Pod、Service、PVC、ServiceMonitor 状态

也就是说，使用者一般不需要手动做：

- `docker load`
- `docker tag`
- `docker push`
- `helm dependency build`
- `kubectl apply ServiceMonitor`

## 默认部署契约

默认参数如下：

- namespace: `aict`
- release name: `redis-cluster`
- total nodes: `6`
- replicas per master: `1`
- 默认拓扑：`3 master + 3 replica`
- Redis password: `Redis@Passw0rd`
- storage class: `nfs`
- storage size: `10Gi`
- metrics: `true`
- ServiceMonitor: `true`
- wait timeout: `10m`
- target image repo: `sealos.hub:5000/kube4`

这是一套“标准 6 节点 Redis Cluster + 默认开启监控”的交付方案。

## 默认拓扑

默认安装：

```bash
./redis-cluster-installer-amd64.run install -y
```

会部署：

- 1 个 Redis Cluster StatefulSet
- 共 `6` 个 Redis Pod
- 默认形成 `3 主 3 从`
- 1 个 headless Service：`redis-cluster-headless`
- 1 个集群访问 Service：`redis-cluster`
- 1 个 metrics Service：`redis-cluster-metrics`
- 6 个 PVC
- 1 个 `ServiceMonitor`

默认不会部署：

- 外部 LoadBalancer
- 额外的业务 sidecar

## 默认访问地址、端口和密码

### 集群内访问

默认 Service 契约：

- 集群入口：`redis-cluster.aict.svc.cluster.local:6379`
- headless Service：`redis-cluster-headless.aict.svc.cluster.local`
- Redis Cluster bus port：`16379`
- metrics Service：`redis-cluster-metrics.aict.svc.cluster.local:9121`

通常业务系统或测试客户端直接使用：

```text
redis-cluster.aict.svc.cluster.local:6379
```

然后配合 cluster 模式连接。

### 默认密码

默认 Redis 密码：

- `Redis@Passw0rd`

生产环境建议在首次安装时显式改掉，不要长期沿用仓库默认值。

### 典型连接方式

集群内客户端示例：

```bash
redis-cli -c -h redis-cluster.aict.svc.cluster.local -p 6379 -a 'Redis@Passw0rd'
```

## 默认资源需求

当前 chart 的资源主要依赖 Bitnami `resourcesPreset`：

- Redis 主容器：`nano`
- exporter：`nano`
- volumePermissions init 容器：`nano`

Bitnami `nano` 预设大致是：

- request: `100m CPU / 128Mi memory / 50Mi ephemeral-storage`
- limit: `150m CPU / 192Mi memory / 2Gi ephemeral-storage`

### 单个 Redis Pod

默认监控开启时，每个 Pod 大致会有：

- Redis 主容器：`100m / 128Mi`
- exporter：`100m / 128Mi`

所以单 Pod 持续资源大致是：

- request: `200m CPU / 256Mi memory`
- limit: `300m CPU / 384Mi memory`

### 默认 6 节点总资源

| 项目 | 单 Pod | 6 节点合计 |
| --- | --- | --- |
| CPU request | `200m` | `1200m` |
| Memory request | `256Mi` | `1536Mi` |
| CPU limit | `300m` | `1800m` |
| Memory limit | `384Mi` | `2304Mi` |

### 存储需求

默认每个节点：

- `10Gi`

默认总存储需求：

- `60Gi`

如果后面你把总节点数调高，存储需求也应该按节点数线性上升。

## 监控设计

Redis 的监控默认就是开启的：

- `metrics.enabled=true`
- `metrics.serviceMonitor.enabled=true`

默认监控对象：

- redis-exporter sidecar
- metrics Service：`redis-cluster-metrics`
- `ServiceMonitor`

默认会自动带平台统一标签：

- `monitoring.archinfra.io/stack=default`

如果集群里没有 `ServiceMonitor` CRD，安装器会自动降级：

- 保留 exporter
- 不创建 `ServiceMonitor`

不会因为监控 CRD 缺失而导致整个 Redis Cluster 安装失败。

## 和其他组件的依赖关系

### Redis 不依赖谁

Redis Cluster 默认不依赖这些组件启动：

- MySQL
- Nacos
- MinIO
- RabbitMQ
- MongoDB
- Milvus

### 谁常常会消费 Redis

在系统集成场景里，Redis 常常被这些系统作为缓存、会话或配置加速组件消费：

- 业务 API 服务
- AI 应用服务
- 通过 `cmict-share.yaml` 导入到 Nacos 的下游业务配置

也就是说，Redis 更多是“被连接使用”，而不是“启动时依赖别人”。

### 和 Prometheus 的关系

如果你的 Prometheus Stack 按我们统一规则做了按标签发现，那么 Redis 安装后会自动接入，因为默认 `ServiceMonitor` 已经带了：

- `monitoring.archinfra.io/stack=default`

## 常见使用场景

### 场景 1：按默认参数安装

```bash
./redis-cluster-installer-amd64.run install -y
```

### 场景 2：显式指定密码和存储类

```bash
./redis-cluster-installer-amd64.run install \
  --storage-class nfs \
  --password 'StrongRedis@2026' \
  -y
```

### 场景 3：关闭监控

```bash
./redis-cluster-installer-amd64.run install \
  --disable-metrics \
  --disable-servicemonitor \
  -y
```

### 场景 4：目标仓库已有镜像

```bash
./redis-cluster-installer-amd64.run install \
  --registry sealos.hub:5000/kube4 \
  --skip-image-prepare \
  -y
```

### 场景 5：透传 Helm 参数

如果安装器没有显式暴露某个 Helm 参数，可以通过 `--` 透传：

```bash
./redis-cluster-installer-amd64.run install -y -- \
  --set cluster.externalAccess.enabled=true \
  --set cluster.externalAccess.service.type=LoadBalancer
```

## 给 AI 或自动化系统的执行规则

如果后续把安装包放到服务器上，希望 AI 参考文档自动部署 Redis，这些规则建议直接写给它。

### 默认优先策略

如果没有额外约束，优先使用：

- `namespace=aict`
- `nodes=6`
- `replicas=1`
- 显式传入密码
- 开启 metrics
- 开启 `ServiceMonitor`

### 成功标准

可以把这些作为成功信号：

- `6` 个 Redis Pod 全部 `Running`
- `redis-cluster` Service 存在
- `redis-cluster-headless` Service 存在
- `redis-cluster-metrics` Service 存在
- PVC 全部绑定成功
- 如果集群支持 `ServiceMonitor`，则 `redis-cluster` 对应的 `ServiceMonitor` 存在

### 失败信号

- Pod 长时间 `Pending`
- Redis Pod `CrashLoopBackOff`
- PVC 未绑定
- Cluster 初始化未完成
- exporter 存在但 metrics Service 不通

## 常见排障命令

```bash
./redis-cluster-installer-amd64.run status -n aict
kubectl get pods,svc,pvc -n aict
kubectl get servicemonitor -A | grep redis-cluster
kubectl logs -n aict redis-cluster-0 --tail=200
```

验证集群连接：

```bash
kubectl run --namespace aict redis-cluster-client --rm --tty -i --restart='Never' \
  --env REDIS_PASSWORD='Redis@Passw0rd' \
  --image sealos.hub:5000/kube4/redis-cluster:8.2.3-debian-12-r0 -- bash

redis-cli -c -h redis-cluster -a "$REDIS_PASSWORD"
```

## 仓库结构

- `build.sh`
  构建多架构 `.run` 安装包
- `install.sh`
  安装器入口，负责解包、镜像准备、Helm 安装和状态输出
- `images/image.json`
  按架构定义镜像来源和目标内网镜像
- `charts/redis-cluster`
  vendored Redis Cluster Helm chart
- `.github/workflows/build-offline-installer.yml`
  GitHub Actions 构建和 release 发布流程

## 构建与发布

本地构建需要：

- `bash`
- `docker`
- `jq`

示例：

```bash
./build.sh --arch amd64
./build.sh --arch arm64
./build.sh --arch all
```

GitHub Actions 负责：

- `main/master` 多架构构建
- `v*` tag 发布 release
