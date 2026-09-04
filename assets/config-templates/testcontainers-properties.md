# Testcontainers 全局配置

> **文件路径**：`src/test/resources/testcontainers.properties`

## 配置说明

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `testcontainers.reuse.enable` | 容器复用（实验性），避免重复启动 | `false` |
| `testcontainers.image.pull.policy` | 镜像拉取策略：`missing`（本地不存在时拉取）、`always` | `missing` |
| `testcontainers.ryuk.disabled` | Ryuk 资源回收，JVM 退出时自动清理容器 | `false` |

## 配置模板

```properties
# Testcontainers 全局配置
# 文件路径: src/test/resources/testcontainers.properties

# 容器复用：避免重复启动（实验性功能）
# 需配合 ~/.testcontainers.properties 中 testcontainers.reuse.enable=true
# testcontainers.reuse.enable=true

# Docker 镜像拉取策略：missing = 本地不存在时才拉取
testcontainers.image.pull.policy=missing

# Ryuk 资源回收：JVM 退出时自动清理容器
# CI 环境可关闭（需手动清理）
testcontainers.ryuk.disabled=false

# Docker 镜像替换（企业内网镜像仓库）
# testcontainers.docker.registry.mirror=https://mirror.example.com
```