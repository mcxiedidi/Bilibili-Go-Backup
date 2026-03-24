# 代码架构文档 (Architecture Documentation)

## 概述 (Overview)

本项目是 Bilibili 基于 Go 语言的后端微服务代码仓库（Monorepo），采用 **Kratos 框架**，使用 **Bazel** 构建系统。仓库包含 **56+ 独立服务**，涵盖视频、用户、社区、直播等核心业务领域，共享统一的基础库 (`library/`)。

## 目录结构 (Directory Structure)

```
.
├── app/                  # 应用层 - 所有业务服务代码
│   ├── admin/            #   管理后台服务 (后台管理 API)
│   ├── interface/        #   对外网关服务 (HTTP/gRPC 对外接口)
│   ├── service/          #   内部 RPC 服务 (核心业务逻辑)
│   ├── job/              #   异步任务服务 (后台队列消费者)
│   ├── infra/            #   基础设施服务 (配置、发现、消息队列等)
│   ├── common/           #   共享业务代码
│   └── tool/             #   开发工具 (Kratos CLI, protoc 插件等)
├── library/              # 基础库 - 所有服务共享的基础组件
├── vendor/               # 第三方依赖 (vendored)
├── build/                # 构建脚本和 CI 配置
├── WORKSPACE             # Bazel 工作空间配置
├── BUILD.bazel           # 根 Bazel 构建文件
├── Makefile              # Make 构建入口
└── .gitlab-ci.yml        # GitLab CI/CD 流水线配置
```

## 架构模式 (Architecture Pattern)

### 分层微服务架构

本项目采用 **Monorepo 微服务架构**，每个业务领域按以下四层组织：

```
                    ┌─────────────────────────┐
                    │     外部客户端/浏览器      │
                    └──────────┬──────────────┘
                               │
                    ┌──────────▼──────────────┐
                    │   Interface (网关层)      │
                    │   HTTP/gRPC 对外接口      │
                    └──────────┬──────────────┘
                               │ RPC 调用
                    ┌──────────▼──────────────┐
                    │   Service (服务层)        │
                    │   核心业务逻辑 / 内部 RPC  │
                    └──────────┬──────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼────┐  ┌───────▼───────┐  ┌─────▼─────────┐
    │   Job (任务层) │  │ Admin (管理层) │  │ Infra (基础层)  │
    │   异步队列消费  │  │  后台管理接口   │  │  配置/发现/消息  │
    └──────────────┘  └───────────────┘  └───────────────┘
```

### 各层职责

| 层级 | 目录 | 职责 | 通信方式 |
|------|------|------|----------|
| **Interface** | `app/interface/` | 对外暴露 HTTP/gRPC API，聚合多个 Service | HTTP, gRPC |
| **Service** | `app/service/` | 核心业务逻辑，提供内部 RPC 接口 | gRPC, Gorpc |
| **Job** | `app/job/` | 消费消息队列，执行异步任务 | Databus 消费 |
| **Admin** | `app/admin/` | 内部管理后台，提供运营管理功能 | HTTP |
| **Infra** | `app/infra/` | 基础设施服务（配置中心、服务发现等） | 各协议 |

## 基础库 (`library/`)

所有服务共享的基础组件，提供统一的基础设施抽象：

### 网络通信

| 包 | 路径 | 说明 |
|----|------|------|
| **net/http** | `library/net/http/` | HTTP 框架，支持中间件 (BM - Blademaster) |
| **net/rpc** | `library/net/rpc/` | RPC 框架，支持 gRPC 和 Gorpc (Warden) |
| **net/trace** | `library/net/trace/` | 分布式链路追踪 (OpenTracing 兼容) |
| **net/metadata** | `library/net/metadata/` | 请求元数据传递 |
| **net/ip** | `library/net/ip/` | IP 地址解析工具 |

### 数据存储

| 包 | 路径 | 说明 |
|----|------|------|
| **database/sql** | `library/database/sql/` | SQL 数据库访问层 (MySQL) |
| **database/hbase** | `library/database/hbase.v2/` | HBase NoSQL 数据库客户端 |
| **database/tidb** | `library/database/tidb/` | TiDB 分布式数据库客户端 |
| **database/elastic** | `library/database/elastic/` | Elasticsearch 搜索引擎客户端 |
| **database/bfs** | `library/database/bfs/` | BFS 文件存储系统客户端 |
| **database/orm** | `library/database/orm/` | ORM 对象关系映射 |

### 缓存

| 包 | 路径 | 说明 |
|----|------|------|
| **cache/redis** | `library/cache/redis/` | Redis 缓存客户端 |
| **cache/memcache** | `library/cache/memcache/` | Memcache 缓存客户端 |

### 消息队列

| 包 | 路径 | 说明 |
|----|------|------|
| **queue/databus** | `library/queue/databus/` | Databus 消息队列客户端 (事件流) |

### 配置与服务发现

| 包 | 路径 | 说明 |
|----|------|------|
| **conf** | `library/conf/` | 配置管理 (配置客户端) |
| **conf/dsn** | `library/conf/dsn/` | DSN 连接字符串解析 |
| **conf/env** | `library/conf/env/` | 环境变量配置 |
| **conf/paladin** | `library/conf/paladin/` | Paladin 配置中心客户端 |
| **naming** | `library/naming/` | 服务注册与发现接口 |
| **naming/discovery** | `library/naming/discovery/` | Discovery 服务发现客户端 |
| **naming/livezk** | `library/naming/livezk/` | ZooKeeper 服务发现 (直播) |

### 可观测性

| 包 | 路径 | 说明 |
|----|------|------|
| **log** | `library/log/` | 结构化日志框架 (多 handler 支持) |
| **log/infoc** | `library/log/infoc/` | 信息采集日志 |
| **log/anticheat** | `library/log/anticheat/` | 反作弊日志 |
| **stat/prom** | `library/stat/prom/` | Prometheus 指标采集 |
| **stat/statsd** | `library/stat/statsd/` | StatsD 指标上报 |
| **stat/counter** | `library/stat/counter/` | 计数器统计 |
| **stat/summary** | `library/stat/summary/` | 摘要统计 |
| **net/trace** | `library/net/trace/` | 分布式链路追踪 |

### 流量控制

| 包 | 路径 | 说明 |
|----|------|------|
| **rate** | `library/rate/` | 限流接口定义 |
| **rate/limit** | `library/rate/limit/` | 令牌桶限流算法 |
| **rate/vegas** | `library/rate/vegas/` | Vegas 自适应限流算法 |

### 并发与数据结构

| 包 | 路径 | 说明 |
|----|------|------|
| **sync/errgroup** | `library/sync/errgroup/` | 错误组 (并发任务管理) |
| **sync/pipeline** | `library/sync/pipeline/` | 管道模式 (数据流处理) |
| **container/pool** | `library/container/pool/` | 对象池 |
| **container/queue** | `library/container/queue/` | 线程安全队列 |

### 错误码

| 包 | 路径 | 说明 |
|----|------|------|
| **ecode** | `library/ecode/` | 统一错误码定义和状态管理 |

错误码按业务域分类：
- `common_ecode.go` — 通用错误 (登录、权限、参数等)
- `main_ecode.go` — 主站业务错误 (稿件、会员等)
- `bbq_ecode.go` — BBQ 服务错误
- `live_ecode.go` — 直播业务错误
- `ep_ecode.go` — EP 基础设施错误
- `open_ecode.go` — 开放平台错误

## 核心业务服务

### Service 层 (内部 RPC 服务)

| 服务 | 路径 | 说明 |
|------|------|------|
| archive | `app/service/main/archive/` | 稿件/视频管理 |
| account | `app/service/main/account/` | 账号服务 |
| relation | `app/service/main/relation/` | 用户关系 (关注/粉丝) |
| member | `app/service/main/member/` | 会员信息 |
| tag | `app/service/main/tag/` | 标签管理 |
| favorite | `app/service/main/favorite/` | 收藏服务 |
| history | `app/service/main/history/` | 观看历史 |
| reply | `app/service/main/reply/` | 评论服务 |
| dm | `app/service/main/dm/` | 弹幕服务 |
| filter | `app/service/main/filter/` | 内容过滤 |
| spy | `app/service/main/spy/` | 反作弊服务 |
| coin | `app/service/main/coin/` | 投币服务 |
| thumbup | `app/service/main/thumbup/` | 点赞服务 |
| location | `app/service/main/location/` | 地理位置 |
| push | `app/service/main/push/` | 推送服务 |
| seq-server | `app/service/main/seq-server/` | 序列号生成 |
| dapper | `app/service/main/dapper/` | 分布式追踪 |
| videoup | `app/service/main/videoup/` | 视频上传 |

### Interface 层 (对外网关)

| 服务 | 路径 | 说明 |
|------|------|------|
| app-view | `app/interface/main/app-view/` | App 视频播放页 API |
| app-feed | `app/interface/main/app-feed/` | App 信息流 API |
| app-channel | `app/interface/main/app-channel/` | App 频道 API |
| web | `app/interface/main/web/` | Web 端 API |
| creative | `app/interface/main/creative/` | 创作中心 API |
| account | `app/interface/main/account/` | 账号接口 |
| dm/dm2 | `app/interface/main/dm/`, `dm2/` | 弹幕接口 |
| reply | `app/interface/main/reply/` | 评论接口 |
| history | `app/interface/main/history/` | 历史记录接口 |
| credit | `app/interface/main/credit/` | 风纪委员会接口 |

### Job 层 (异步任务)

| 服务 | 路径 | 说明 |
|------|------|------|
| videoup | `app/job/main/videoup/` | 视频上传后处理 |
| relation | `app/job/main/relation/` | 关系变更异步处理 |
| member | `app/job/main/member/` | 会员异步任务 |
| archive | `app/job/main/archive/` | 稿件异步处理 |
| dm/dm2 | `app/job/main/dm/`, `dm2/` | 弹幕异步处理 |
| reply | `app/job/main/reply/` | 评论异步处理 |
| push | `app/job/main/push/` | 推送异步发送 |
| spy | `app/job/main/spy/` | 反作弊异步分析 |

### Infra 层 (基础设施服务)

| 服务 | 路径 | 说明 |
|------|------|------|
| config | `app/infra/config/` | 配置中心服务 |
| discovery | `app/infra/discovery/` | 服务发现与注册中心 |
| databus | `app/infra/databus/` | 消息队列服务 |
| canal | `app/infra/canal/` | 数据库 Binlog 订阅服务 |
| notify | `app/infra/notify/` | 通知服务 |

## 单个服务内部结构

每个服务遵循统一的目录结构：

```
app/service/main/{service-name}/
├── cmd/                  # 入口文件
│   └── main.go           #   服务启动入口
├── api/                  # API 定义
│   ├── api.proto         #   Protobuf 接口定义
│   ├── api.pb.go         #   生成的 Go 代码
│   └── client.go         #   RPC 客户端
├── conf/                 # 配置
│   └── conf.go           #   配置结构体定义
├── dao/                  # 数据访问层 (Data Access Object)
│   ├── dao.go            #   DAO 初始化
│   ├── mysql.go          #   MySQL 数据访问
│   ├── redis.go          #   Redis 数据访问
│   └── mc.go             #   Memcache 数据访问
├── model/                # 数据模型
│   └── model.go          #   业务实体定义
├── server/               # 服务器
│   ├── http/             #   HTTP 服务 (BM 路由)
│   │   └── http.go
│   └── grpc/             #   gRPC 服务
│       └── server.go
├── service/              # 业务逻辑层
│   └── service.go        #   业务逻辑实现
├── CHANGELOG.md          # 版本变更日志
├── CONTRIBUTORS.md       # 贡献者/审核权限
└── README.md             # 服务说明文档
```

### 代码分层说明

```
  HTTP/gRPC 请求
        │
        ▼
  ┌─────────────┐
  │   server/    │  路由层 - 请求路由和参数绑定
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  service/    │  业务逻辑层 - 核心业务处理
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │    dao/      │  数据访问层 - 数据库/缓存/RPC 调用
  └──────┬──────┘
         │
    ┌────┼────┐
    ▼    ▼    ▼
  MySQL Redis Memcache ...
```

## 构建系统

### Bazel 构建

项目使用 **Bazel** 作为主构建系统：

```bash
# 构建特定服务
bazel build //app/service/main/archive/cmd:cmd

# 运行测试
bazel test //app/service/main/archive/...

# 使用 Kratos 工具构建
kratos build
```

### 关键构建文件

| 文件 | 说明 |
|------|------|
| `WORKSPACE` | Bazel 工作空间，定义 Go SDK 和外部依赖 |
| `BUILD.bazel` | 各目录的构建规则 |
| `.gitlab-ci.yml` | CI/CD 流水线 (编译、lint、测试) |
| `build/` | 构建辅助脚本 |

### CI/CD 流程

```
MR (Merge Request)
  │
  ├── COMPILE  ── Bazel 增量编译 + Master 合并检查
  ├── LINT     ── Gometalinter 代码检查
  ├── TEST     ── 单元测试执行
  └── SAGA     ── 工作流一致性校验
```

## 开发工具

| 工具 | 路径 | 说明 |
|------|------|------|
| **kratos** | `app/tool/kratos/` | 项目脚手架 CLI (初始化、构建、升级) |
| **protoc-gen-bm** | `app/tool/protoc-gen-bm/` | Protobuf → BM HTTP 代码生成器 |
| **warden** | `app/tool/warden/` | Warden RPC 中间件工具 |
| **gorpc** | `app/tool/gorpc/` | Gorpc 代码生成工具 |
| **liverpc** | `app/tool/liverpc/` | 直播 RPC 框架工具 |
| **saga** | `app/admin/ep/saga/` | 代码审查和合并自动化工具 |
| **merlin** | `app/admin/ep/merlin/` | 开发环境管理工具 |

## 技术栈总结

| 类别 | 技术选型 |
|------|----------|
| **编程语言** | Go 1.11.4+ |
| **框架** | Kratos (自研) |
| **构建系统** | Bazel |
| **RPC 框架** | gRPC, Gorpc (Warden) |
| **HTTP 框架** | BM (Blademaster) |
| **数据库** | MySQL, TiDB, HBase |
| **缓存** | Redis, Memcache |
| **搜索引擎** | Elasticsearch |
| **消息队列** | Databus |
| **服务发现** | Discovery, ZooKeeper |
| **配置中心** | Paladin |
| **链路追踪** | Dapper (OpenTracing) |
| **监控指标** | Prometheus, StatsD |
| **CI/CD** | GitLab CI |
| **接口定义** | Protocol Buffers |
