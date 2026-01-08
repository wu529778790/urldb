# urlDB - 网盘资源管理系统

一个全栈网盘资源管理系统，支持多个云存储平台的资源管理、自动验证和批量转存操作。

## 技术栈

**后端：** Go 1.23+、Gin Web 框架、GORM ORM、SQLite、JWT 认证、robfig/cron 定时任务

**前端：** Nuxt.js 3、Vue 3、TypeScript、Tailwind CSS、Pinia 状态管理、Naive UI 组件库

**基础设施：** Docker、Docker Compose、Nginx 反向代理

## 支持的网盘平台

| 平台 | 录入 | 转存+分享 |
|------|-------|----------|
| 百度网盘 | ✅ | ❌ |
| 阿里云盘 | ✅ | ❌ |
| 夸克网盘 | ✅ | ✅ |
| 天翼云盘 | ✅ | ❌ |
| 迅雷云盘 | ✅ | ✅ |
| 123云盘 | ✅ | ❌ |
| 115网盘 | ✅ | ❌ |
| UC网盘 | ✅ | ❌ |

## 核心功能

- **资源管理** - 支持批量添加、验证资源有效性
- **自动转存** - 支持夸克网盘、迅雷云盘的自动转存和分享
- **多账号管理** - 同平台支持多账号轮换使用
- **任务系统** - 异步任务处理，支持暂停/继续/重试
- **定时任务** - 自动处理待验证资源
- **API 支持** - 公开 API 用于批量操作和搜索

## 快速开始

### 环境要求

- Go 1.23+
- Node.js 18+
- Docker & Docker Compose (可选)

### 开发环境运行

**后端：**
```bash
# 运行开发服务器
go run main.go

# 运行测试
go test ./...

# 格式化代码
gofmt -w .
```

**前端：**
```bash
cd web
pnpm install
pnpm dev      # 开发服务器
pnpm build    # 生产构建
```

### Docker 部署

```bash
# 使用脚本构建
./scripts/docker-build.sh build 1.3.0

# 使用 compose 运行
docker-compose up -d
```

## 配置

### 环境变量 (.env)

```bash
# 数据库配置
DB_PATH=data/urldb.db

# 服务器配置
PORT=8080
TIMEZONE=Asia/Shanghai

# JWT 配置
JWT_SECRET=your-secret-key

# 上传配置
MAX_UPLOAD_SIZE=50MB
```

### 系统配置

系统启动后，可通过管理界面配置：
- 自动处理待验证资源 (`auto_process_ready_resources`)
- 转存任务相关设置
- 平台账号管理

## 架构设计

### 后端结构

```
main.go                    # 应用入口
├── db/                    # 数据库层
│   ├── entity/           # GORM 模型
│   ├── repo/             # 仓储模式 (RepositoryManager)
│   ├── dto/              # 数据传输对象
│   ├── converter/        # 实体转换器
│   └── migrations/       # 数据库迁移
├── handlers/             # HTTP 处理器
├── middleware/           # 中间件 (认证、CORS、日志)
├── scheduler/            # 定时任务调度器
├── task/                 # 异步任务处理器
└── utils/                # 工具函数
```

### 核心架构模式

1. **仓储模式** - 通过 `RepositoryManager` 统一管理所有数据库访问
2. **任务队列系统** - 内存任务管理器，支持异步转存操作
3. **调度器模式** - 单例全局调度器管理定时任务
4. **多账号轮换** - 同平台多账号自动选择

### API 结构

- **公开 API** (`/api/public/*`) - 基于令牌的批量操作
- **认证** (`/api/auth/*`) - 登录、注册
- **管理后台** (`/api/*`) - JWT 认证的 CRUD 操作
  - 资源管理：`/api/resources/*`
  - 分类管理：`/api/categories/*`
  - 平台管理：`/api/pans/*`, `/api/cks/*`
  - 标签管理：`/api/tags/*`
  - 待处理资源：`/api/ready-resources/*`
  - 任务管理：`/api/tasks/*`
  - 系统配置：`/api/system/*`

## 工作流程

### 转存任务流程

1. 创建任务 → `task` 表和 `task_item` 表
2. `TaskManager` 启动后台处理
3. `TransferProcessor` 执行转存：
   - 检查资源存在性
   - 选择匹配账号
   - 调用网盘 API
   - 创建资源记录
4. 实时更新进度和状态

### 调度器工作流程

1. 读取 `auto_process_ready_resources` 配置
2. 启动 `ReadyResourceScheduler`
3. 定期检查待处理资源
4. 自动执行转存操作

## 版本管理

```bash
./scripts/version.sh show    # 显示当前版本
./scripts/version.sh patch   # 1.3.0 -> 1.3.1
./scripts/version.sh minor   # 1.3.0 -> 1.4.0
./scripts/version.sh major   # 1.3.0 -> 2.0.0
```

## 生产构建

```bash
# 使用构建脚本
./scripts/build.sh
./scripts/build.sh build-linux

# 手动构建
VERSION=$(cat VERSION)
GIT_COMMIT=$(git rev-parse --short HEAD)
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo \
  -ldflags "-X 'github.com/ctwj/urldb/utils.Version=${VERSION}' \
            -X 'github.com/ctwj/urldb/utils.GitCommit=${GIT_COMMIT}'" \
  -o main .
```

## 数据库

- 使用 SQLite 数据库
- 启动时自动运行迁移
- 自动插入默认数据（分类、平台、系统配置、管理员用户）

## 代码风格

- **Go**：使用 `gofmt` 格式化
- **TypeScript**：ESLint + Prettier
- **提交信息**：`类型(范围): 简短描述`（feat、fix、docs、style、refactor、test、chore）

## 许可证

本项目采用 [GPL v3](LICENSE) 许可证。
