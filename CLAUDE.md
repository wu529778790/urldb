# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

urlDB 是一个全栈网盘资源管理系统，支持多个云存储平台（百度网盘、阿里云盘、夸克网盘、天翼云盘、迅雷云盘、123云盘、115网盘、UC网盘）。主要功能包括资源有效性自动验证、批量转存操作、多账号管理等。

## 技术栈

**后端：** Go 1.23+、Gin Web 框架、GORM ORM、SQLite、JWT 认证、robfig/cron 定时任务

**前端：** Nuxt.js 3、Vue 3、TypeScript、Tailwind CSS、Pinia 状态管理、Naive UI 组件库

**基础设施：** Docker、Docker Compose、Nginx 反向代理

## 常用命令

### 后端开发

```bash
# 运行开发服务器
go run main.go

# 查看版本
./main version

# 运行测试
go test ./...
go test -cover ./...

# 格式化代码
gofmt -w .
```

### 前端开发

```bash
cd web
pnpm install
pnpm dev      # 开发服务器
pnpm build    # 生产环境构建
```

### 生产环境构建

```bash
# 使用构建脚本（推荐 - 编译 Linux 版本）
./scripts/build.sh
./scripts/build.sh build-linux

# 手动构建（带版本信息）
VERSION=$(cat VERSION)
GIT_COMMIT=$(git rev-parse --short HEAD)
GIT_BRANCH=$(git branch --show-current)
BUILD_TIME=$(date '+%Y-%m-%d %H:%M:%S')

CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo \
  -ldflags "-X 'github.com/ctwj/urldb/utils.Version=${VERSION}' \
            -X 'github.com/ctwj/urldb/utils.BuildTime=${BUILD_TIME}' \
            -X 'github.com/ctwj/urldb/utils.GitCommit=${GIT_COMMIT}' \
            -X 'github.com/ctwj/urldb/utils.GitBranch=${GIT_BRANCH}'" \
  -o main .
```

### Docker 操作

```bash
# 使用脚本构建
./scripts/docker-build.sh build 1.3.0

# 手动构建
docker build --target backend -t ctwj/urldb-backend:1.3.0 .
docker build --target frontend -t ctwj/urldb-frontend:1.3.0 .

# 使用 compose 运行
docker-compose up -d
```

### 版本管理

```bash
./scripts/version.sh show    # 显示当前版本
./scripts/version.sh patch   # 1.3.0 -> 1.3.1
./scripts/version.sh minor   # 1.3.0 -> 1.4.0
./scripts/version.sh major   # 1.3.0 -> 2.0.0
```

## 架构设计

### 后端结构

后端采用分层架构，关注点分离明确：

- **`main.go`** - 应用入口，路由设置，中间件初始化，任务管理器和调度器初始化
- **`db/`** - 数据库层
  - `entity/` - GORM 模型（User、Category、Pan、Cks、Tag、Resource、ResourceTag、ReadyResource、SearchStat、SystemConfig、ResourceView、Task、TaskItem、File、APIAccessLog）
  - `repo/` - 仓储模式实现（RepositoryManager 统一管理所有仓储）
  - `dto/` - API 响应的数据传输对象
  - `converter/` - 实体与 DTO 之间的转换器
  - `migrations/` - 数据库迁移脚本
- **`handlers/`** - HTTP 请求处理器（20+ 个处理器）
- **`middleware/`** - 认证中间件、日志记录、CORS、公开 API 令牌验证
- **`scheduler/`** - 定时任务管理（待处理资源自动处理）
- **`task/`** - 异步任务处理器（转存任务）
- **`utils/`** - 共享工具（日志、版本、时区、时间处理）

### 前端结构 (`web/`)

- **`components/`** - Vue 组件（自动导入）
- **`pages/`** - 基于文件的路由
- **`stores/`** - Pinia 状态管理（resource、systemConfig、task、user）
- **`composables/`** - 可复用的组合式函数（useApi、useConfigChangeDetection、useSeo、useTimeFormat、useVersion）
- **`middleware/`** - Nuxt 路由守卫中间件
- **`layouts/`** - Vue 布局组件
- **`server/`** - 服务端代码（Nitro 插件、API 路由）

### 请求流程

```
客户端请求
    ↓
Nginx (端口 3030)
    ↓
├─→ /api/* → 后端 Gin (端口 8080)
│       ↓
│   中间件 (认证/CORS/日志)
│       ↓
│   处理器 → 仓储 (GORM) → SQLite
│
└─→ /* → 前端 Nuxt (端口 3000)
```

### 核心架构模式

1. **仓储模式** - 所有数据库访问通过 `db/repo/` 中的仓储抽象，使用 `RepositoryManager` 统一管理
2. **DTO 模式** - 数据传输对象与实体分离，用于 API 响应
3. **任务队列系统** - 内存任务管理器，使用后台 goroutine 处理异步操作（转存）
   - `TaskManager` - 管理任务生命周期（启动、暂停、停止、恢复）
   - `TaskProcessor` - 任务处理器接口，目前有 `TransferProcessor`
   - 任务状态存储在 `task` 和 `task_item` 表中
4. **调度器模式** - 单例全局调度器管理定时任务
   - `GlobalScheduler` - 全局调度器（单例）
   - `ReadyResourceScheduler` - 待处理资源自动处理调度器
   - 通过系统配置动态启停
5. **中间件管道** - 认证中间件验证 JWT 令牌，公开 API 中间件验证自定义令牌
6. **多账号轮换** - 同平台支持多账号，转存时自动选择可用账号

### 多账号平台支持

- 平台凭证存储在 `db/entity/pan.go` 和 `db/entity/cks.go`
- 当存在多个账号时，转存和分享操作使用账号轮换
- 完整转存+分享支持：夸克网盘、迅雷云盘
- 仅录入支持：百度网盘、阿里云盘、天翼云盘、123云盘、115网盘、UC网盘

### 任务处理系统

任务通过管理界面创建并异步处理：

1. 在 `task` 表中创建任务，类型为 `transfer`，状态为 `pending`/`running`/`paused`/`completed`/`failed`/`partial_success`
2. 在 `task_item` 表中为每个资源创建任务项，包含输入数据（标题、URL、分类、标签等）
3. `TaskManager` 启动后台 goroutine，使用 `TransferProcessor` 处理任务项
4. 处理器执行转存操作，调用网盘 API，创建资源记录
5. 状态实时写回数据库，支持进度跟踪和服务器重启恢复

**关键特性：**
- 服务器重启后自动恢复运行中的任务
- 支持任务暂停/继续/停止
- 详细的进度统计和输出数据
- 失败任务可重试

### 定时任务

- **待处理资源调度器** - 处理待验证的资源并进行自动转存
- 可通过系统配置启用/禁用 (`auto_process_ready_resources`)
- 调度器状态通过 `GlobalScheduler` 管理

## 重要配置文件

- **`.env`** - 数据库连接、服务器端口、时区、上传设置
- **`VERSION`** - 当前版本号（使用 `scripts/version.sh` 更新）
- **`web/nuxt.config.ts`** - 前端配置，包含开发环境的 API 代理设置
- **`docker-compose.yml`** - 多服务编排（后端、前端、Nginx）
- **`nginx/nginx.conf`** 和 **`nginx/conf.d/default.conf`** - 反向代理路由配置

## 代码风格

- **Go**：遵循标准 Go 规范，使用 `gofmt` 格式化
- **TypeScript**：使用 TypeScript 严格模式，遵循 ESLint 规则，Prettier 格式化
- **组件命名**：组件使用 PascalCase，变量使用 camelCase
- **中文提交信息**：使用格式 `类型(范围): 简短描述`，类型包括：feat、fix、docs、style、refactor、test、chore

## 数据库

- 使用 SQLite 数据库
- 启动时自动运行迁移（由 `SKIP_MIGRATION` 环境变量控制）
- 自动插入默认数据：分类、平台、系统配置、管理员用户
- 通过 `RepositoryManager` 统一管理所有数据库访问

## API 结构

- **公开 API** (`/api/public/*`) - 基于令牌的端点，用于批量资源操作和搜索
- **认证** (`/api/auth/*`) - 登录、注册、个人信息
- **管理后台** (`/api/*`) - 需要 JWT 认证的受保护端点，用于所有 CRUD 操作
  - 资源管理：`/api/resources/*`
  - 分类管理：`/api/categories/*`
  - 平台管理：`/api/pans/*`, `/api/cks/*`
  - 标签管理：`/api/tags/*`
  - 待处理资源：`/api/ready-resources/*`
  - 任务管理：`/api/tasks/*`
  - 统计信息：`/api/stats/*`, `/api/search-stats/*`
  - 系统配置：`/api/system/*`
  - 文件管理：`/api/files/*`
  - API 访问日志：`/api/api-access-logs/*`

## 核心工作流程

### 转存任务流程

1. 用户在管理界面创建转存任务，指定资源列表、目标账号
2. 系统在 `task` 表创建任务，在 `task_item` 表创建任务项
3. `TaskManager.StartTask()` 启动后台 goroutine
4. `TransferProcessor.Process()` 处理每个任务项：
   - 检查资源是否已存在
   - 选择匹配的账号（根据 URL 类型）
   - 调用网盘 API 执行转存
   - 创建资源记录并保存转存链接
   - 添加标签关联
5. 更新任务进度和状态

### 资源管理流程

1. 资源可通过 API 批量添加或手动创建
2. 系统验证资源有效性（通过访问链接）
3. 有效资源可自动转存（通过调度器）
4. 转存成功后生成分享链接并存储

### 调度器工作流程

1. 系统启动时读取 `auto_process_ready_resources` 配置
2. 如果启用，启动 `ReadyResourceScheduler`
3. 定期检查 `ready_resources` 表中的待处理资源
4. 自动执行转存操作
5. 更新资源状态和错误信息
