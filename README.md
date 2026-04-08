# app_redis-cluster

Redis Cluster 离线安装项目，面向内网和弱网环境，提供：

- `amd64` / `arm64` 多架构离线安装包
- 内置 Redis Cluster、`os-shell`、`redis-exporter` 镜像载荷
- 默认启用 exporter
- 默认创建 ServiceMonitor
- 通过 GitHub Actions 自动构建和发布 release

## 目录说明

- `build.sh`: 构建多架构 `.run` 离线安装包
- `install.sh`: 自解压安装器模板
- `images/image.json`: 多架构镜像定义
- `charts/redis-cluster`: Redis Cluster Helm Chart
- `.github/workflows/build-offline-installer.yml`: GitHub Actions 构建与 release

## 本地构建

需要：

- `bash`
- `docker`
- `jq`

示例：

```bash
./build.sh --arch amd64
./build.sh --arch arm64
./build.sh --arch all
```

构建产物位于 `dist/`：

- `redis-cluster-installer-amd64.run`
- `redis-cluster-installer-amd64.run.sha256`
- `redis-cluster-installer-arm64.run`
- `redis-cluster-installer-arm64.run.sha256`

## 安装器用法

查看帮助：

```bash
./redis-cluster-installer-amd64.run --help
./redis-cluster-installer-amd64.run help
```

基础安装：

```bash
./redis-cluster-installer-amd64.run install \
  --namespace aict \
  --storage-class nfs \
  --password 'Redis@123' \
  -y
```

启用 exporter 和 ServiceMonitor：

```bash
./redis-cluster-installer-amd64.run install \
  --namespace aict \
  --storage-class nfs \
  -y
```

如果目标仓库已经有镜像，可跳过镜像准备：

```bash
./redis-cluster-installer-amd64.run install \
  --registry sealos.hub:5000/kube4 \
  --skip-image-prepare \
  -y
```

查看状态：

```bash
./redis-cluster-installer-amd64.run status -n aict
```

卸载：

```bash
./redis-cluster-installer-amd64.run uninstall -n aict -y
```

## 监控说明

项目已经把监控链路收敛到安装器参数：

- 默认会打开 `redis-exporter` sidecar、metrics service 和 ServiceMonitor
- 需要关闭时再显式传 `--disable-metrics` 或 `--disable-servicemonitor`
- 如果集群中不存在 `servicemonitors.monitoring.coreos.com`，安装器会自动降级并给出告警

Chart 中对应资源：

- `charts/redis-cluster/templates/metrics-svc.yaml`
- `charts/redis-cluster/templates/metrics-prometheus.yaml`

## GitHub Actions 发版

仓库推送到 `main` 后：

- 自动构建 `amd64` / `arm64` 安装包
- 自动上传 Actions artifacts

推送 `v*` tag 后：

- 自动把两种架构的 `.run` 包和 `.sha256` 发布到 GitHub Release

推荐发版流程：

```bash
git push origin main
git tag v0.1.0
git push origin v0.1.0
```
