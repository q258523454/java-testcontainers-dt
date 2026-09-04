# 测试资源目录结构规范

本文件定义了 `src/test/resources/` 目录的结构规范，指导开发者清晰、一致地组织 DT 测试资源。

---

## 一、顶层结构

```
src/test/resources/
├── application.properties              # 测试配置
├── testcontainers.properties           # Testcontainers 全局配置
├── data/                               # 中间件测试数据
│   ├── mysql/                          #   MySQL 测试数据
│   ├── css/                            #   Elasticsearch 测试数据
│   ├── redis/                          #   Redis 测试数据
│   ├── mongo/                          #   MongoDB 测试数据
│   └── kafka/                          #   Kafka 测试数据
└── wiremock/                           # WireMock 回放数据
    ├── __files/                        #   响应体文件目录
    └── mappings/                       #   Stub 映射配置目录
```

---

## 二、中间件测试数据（data/{mysql|css}/）

`mysql/` 与 `css/` 目录结构完全一致，统一遵循 `fixtures / migrations / seeds / testcontainers` 四层组织。以下以 `{中间件}` 代指二者。

### 2.1 通用目录结构

```
data/{中间件}/
├── fixtures/                               # 测试夹具
│   ├── global/                             #   全局夹具（跨测试共享，变更频率低）
│   ├── modules/                            #   模块夹具（按业务模块复用，变更频率中）
│   └── tests/                              #   用例夹具（特定测试专用，变更频率高）
├── migrations/                             # 迁移脚本（与生产环境严格一致）
│   ├── v1/                                 #   v1 版本迁移
│   └── v2/                                 #   v2 版本迁移
├── seeds/                                  # 种子数据（开发/测试环境粗粒度数据）
└── testcontainers/                         # Testcontainers 容器化测试
    └── image/                              #   自定义镜像构建资源
        ├── README.md                       #     镜像构建指南
        ├── Dockerfile                      #     镜像定义
        ├── build.sh                        #     构建脚本（WSL 环境执行）
        ├── init/                           #     初始化脚本目录
        └── tokenizer/                      #     分词器插件目录（仅 css）
```

### 2.2 fixtures（测试夹具）

> 在测试运行前后需要被执行的代码片段，用于设置测试环境和数据准备。可根据不同测试用例组装数据（全局 + 模块 + 特定用例）。

| 子目录 | 用途 | 变更频率 |
|---|---|---|
| `global/` | 跨多个测试共享的全局基础数据（建表/建索引 + 基础数据） | 低 |
| `modules/` | 按业务模块组织的可复用数据场景 | 中 |
| `tests/` | 按测试类组织的特定测试用例专用数据 | 高 |

**文件命名规范**（mysql / css / redis / mongo / kafka 等中间件通用）：

| 文件命名 | 用途 |
|---|---|
| `init-schema.sql` | 初始化表结构 |
| `cleanup-schema.sql` | 清理表结构 |
| `init-{entity}-data.sql` | 初始化 {实体} 数据 |
| `cleanup-{entity}-data.sql` | 清理 {实体} 数据 |

> `{entity}` 为实体名，使用小写英文 + 连字符，如 `init-user-data.sql`、`cleanup-contact-data.sql`。css / redis 同理，文件后缀为 `.json`。

### 2.3 migrations（迁移脚本）

> 完全模拟生产环境数据库演进过程，与生产环境保持严格一致。

- 按 `v1/`、`v2/` 版本目录组织，每个版本对应一次生产环境结构变更
- 脚本内容须与生产环境 DBA 执行的迁移脚本完全一致

### 2.4 seeds（种子数据）

> 开发/测试环境需要的种子数据（非生产数据），方便替换不同环境专用数据。

- 用于本地开发、联调等非生产场景的快速数据填充
- 与 `fixtures/` 的区别：seeds 是"环境级"粗粒度数据，fixtures 是"用例级"细粒度数据

### 2.5 testcontainers（容器化测试）

| 文件/目录 | 用途 |
|---|---|
| `image/Dockerfile` | 镜像定义文件 |
| `image/build.sh` | 镜像构建脚本（WSL 环境执行） |
| `image/README.md` | 镜像构建与验证指南 |
| `image/init/` | 初始化脚本目录（各中间件内容不同，见 2.6 节） |
| `image/tokenizer/` | 分词器插件目录（仅 css，存放 ES 分词器 zip 包） |

### 2.6 各中间件差异

两者目录结构相同，仅数据格式和 `image/init/` 内容不同：

| 中间件 | fixtures 数据格式 | migrations 数据格式 | seeds 数据格式 | image/init/ 内容 |
|---|---|---|---|---|
| `mysql/` | `.sql` | `.sql` | `.sql` | SQL 初始化脚本 + 数据导出脚本 |
| `css/`（Elasticsearch） | `.json` | `.json` | `.json` | 索引初始化脚本 + 数据导出脚本 |
| `mongo/` | `.json` | `.json` | `.json` | mongo 初始化脚本 + 数据导出脚本 |
| `kafka/` | `.json` | `.json` | `.json` | kafka 初始化脚本 + 数据导出脚本 |


---

## 三、WireMock 目录（wiremock/）

WireMock 采用标准的 `__files` + `mappings` 双目录结构，符合 WireMock 官方约定。建议按业务场景组织目录结构：

```
wiremock/
├── searchperson/                         # 按业务场景组织
│   ├── mappings/                         #   Stub 映射配置
│   │   ├── combine-search-zhangsan-success.json
│   │   └── combine-search-default-success.json
│   └── __files/                          #   响应体文件
│       ├── combine-search-zhangsan-success.json
│       └── combine-search-default-success.json
├── payment/                              # 另一个业务场景
│   ├── mappings/
│   └── __files/
```

### 3.1 目录约定

| 约定项 | 规则 |
|---|---|
| `__files/` | 存放 HTTP 响应体文件（JSON、XML 等大体积数据），被 `mappings/` 中的配置引用 |
| `mappings/` | 存放 Stub 映射配置，定义 URL 匹配规则、请求条件及响应策略 |
| 场景目录命名 | `{业务场景}`，使用小写英文，如 `searchperson/`、`searchrobot/` |
| 数据文件命名 | `{接口/操作}-{场景/结果}.json`，使用小写英文 + 连字符 |

---

## 四、完整目录树示例

```
src/test/resources/
├── application.properties
├── testcontainers.properties
├── data/
│   ├── mysql/
│   │   ├── fixtures/
│   │   │   ├── global/
│   │   │   │   ├── init-schema.sql
│   │   │   │   ├── cleanup-schema.sql
│   │   │   │   ├── init-{entity}-data.sql
│   │   │   │   └── cleanup-{entity}-data.sql
│   │   │   ├── modules/
│   │   │   └── tests/
│   │   ├── migrations/
│   │   │   ├── v1/
│   │   │   └── v2/
│   │   ├── seeds/
│   │   └── testcontainers/
│   │       └── image/
│   │           ├── README.md
│   │           ├── Dockerfile
│   │           ├── build.sh
│   │           ├── init/
│   ├── css/
│   │   ├── fixtures/
│   │   │   ├── global/
│   │   │   │   ├── init-schema.json
│   │   │   │   ├── cleanup-schema.json
│   │   │   │   ├── init-{entity}-data.json
│   │   │   │   └── cleanup-{entity}-data.json
│   │   │   ├── modules/
│   │   │   └── tests/
│   │   ├── migrations/
│   │   │   ├── v1/
│   │   │   └── v2/
│   │   ├── seeds/
│   │   └── testcontainers/
│   │       └── image/
│   │           ├── README.md
│   │           ├── Dockerfile
│   │           ├── build.sh
│   │           ├── init/
│   │           └── tokenizer/
│   ├── redis/
│   │   └── (目录结构同 mysql/css)
└── wiremock/
    ├── __files/
    │   └── {场景名}/
    │       └── {接口/操作}-{场景/结果}.json
    └── mappings/
        └── {场景名}/
            └── {接口/操作}-{场景/结果}.json
```

---

## 五、新增文件规范

| 场景 | 放置位置 | 命名规范 |
|---|---|---|
| 初始化表结构 | `data/{中间件}/fixtures/{层级}/` | `init-schema.sql` / `init-schema.json` |
| 清理表结构 | `data/{中间件}/fixtures/{层级}/` | `cleanup-schema.sql` / `cleanup-schema.json` |
| 初始化实体数据 | `data/{中间件}/fixtures/{层级}/` | `init-{entity}-data.sql` / `init-{entity}-data.json` |
| 清理实体数据 | `data/{中间件}/fixtures/{层级}/` | `cleanup-{entity}-data.sql` / `cleanup-{entity}-data.json` |
| 迁移脚本 | `data/{中间件}/migrations/v{N}/` | 与生产环境迁移脚本同名 |
| 种子数据 | `data/{中间件}/seeds/` | `{环境名}_{用途}.sql` 或 `{用途}.json` |
| WireMock 场景 | `wiremock/mappings/{场景名}/` | `{接口/操作}-{场景/结果}.json` |
| WireMock 响应体 | `wiremock/__files/{场景名}/` | `{接口/操作}-{场景/结果}.json` |

> `{中间件}` 代指 `mysql`、`css` 中的任一目录；`{层级}` 代指 `global`、`modules`、`tests`；`{entity}` 为实体名，使用小写英文 + 连字符。

### 新增 WireMock 场景

1. 在 `wiremock/mappings/{场景名}/` 下创建映射配置文件
2. 如有独立响应体文件，放置在 `wiremock/__files/{场景名}/` 下，由映射配置引用

### 新增中间件测试数据

如需新增其他中间件（如 PostgreSQL、Oracle 等），在 `data/` 下创建以中间件名命名的目录，并遵循统一的 `fixtures / migrations / seeds / testcontainers` 四层结构。

> **已支持的中间件**：MySQL、Redis、Elasticsearch（css）、MongoDB、Kafka

---

## 六、注意事项

1. **路径前缀为 `data/`**：中间件测试数据统一放在 `data/` 子目录下，引用路径为 `classpath:data/{中间件}/...`
2. **迁移脚本与生产一致**：`migrations/` 下的脚本必须与生产环境 DBA 执行的脚本完全一致，不得自行修改
3. **fixtures 三层结构**：`global/` → `modules/` → `tests/` 按变更频率从低到高组织，新增数据时根据复用范围选择正确层级
4. **中间件目录结构统一**：`mysql/`、`css/` 子目录结构完全一致，仅数据格式不同（见 2.6 节）