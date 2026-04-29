这个项目的「核心模块、迁移、网关、前后端架构」一次讲清楚。我读了关键文件后，按真实代码顺序还原架构，方便你直接在它上面继续开发。



## 一、后端核心模块（`backend/app/`）

```
app/
├── main.py                  # FastAPI 入口：建 app、挂中间件、注册路由
├── api/
│   ├── main.py              # 顶层 APIRouter 聚合：login/users/utils/items/private
│   ├── deps.py              # 依赖注入：DB Session、当前用户、超级管理员
│   └── routes/              # 业务路由：items / login / users / utils / private
├── core/
│   ├── config.py            # pydantic-settings 全局配置（读 .env）
│   ├── db.py                # SQLAlchemy engine + 初始化首个超级管理员
│   └── security.py          # 密码哈希(Argon2/Bcrypt) + JWT 生成
├── crud.py                  # 用户/Item 的增改与认证 CRUD
├── models.py                # SQLModel 定义：表 + 输入/输出 Schema
├── alembic/                 # 数据库迁移：env.py + versions/*
├── alembic.ini              # Alembic 配置（脚本目录）
├── backend_pre_start.py     # 等数据库可用（tenacity 重试）
├── initial_data.py          # 调 init_db 创建首个超级管理员
└── utils.py                 # 邮件发送 / 重置 token 工具
```



### 各模块职责一句话总结

- `main.py`：构造 `FastAPI(...)`、若 `SENTRY_DSN` 启用 Sentry、加 `CORSMiddleware`、`include_router(api_router, prefix=settings.API_V1_STR)`。

  ```python
  # main.pyLines 17-34
  app = FastAPI(
      title=settings.PROJECT_NAME,
      openapi_url=f"{settings.API_V1_STR}/openapi.json",
      generate_unique_id_function=custom_generate_unique_id,
  )
  ```

  

- `api/main.py`：聚合各业务路由；本地环境额外挂 `private` 子路由（仅 `local` 环境）。

  ```python
  # main.pyLines 13-14
  if settings.ENVIRONMENT == "local":
      api_router.include_router(private.router)
  ```

  

- `api/deps.py`：三个核心依赖类型——

  - `SessionDep`：开 `Session(engine)` 注入到路由

  - `CurrentUser`：解 JWT、查库、校验 `is_active`

  - `get_current_active_superuser`：校验 `is_superuser`

    

- `core/config.py`：`Settings(BaseSettings)`，字段对应 `.env` 中变量；并：

  - 计算 `SQLALCHEMY_DATABASE_URI`（`postgresql+psycopg://...`）

  - 用 `_check_default_secret` 强制非 local 环境下不允许 `changethis`

    

- `core/db.py`：建 `engine`；`init_db` 用 `select` 查首个超管，没有就 `crud.create_user` 创建。

- `core/security.py`：

  - `password_hash` = `Argon2 + Bcrypt` 多哈希器（`pwdlib`），支持渐进升级旧 hash
  - `create_access_token` 用 `SECRET_KEY` HS256 签 JWT

- `crud.py`：纯函数式增删改 + `authenticate`（含抗时序攻击的 dummy hash）。

- `models.py`：典型 SQLModel 三层模型——

  - `XxxBase`（共享字段）→ `XxxCreate/Update/Public`（API DTO）→ `Xxx(table=True)`（DB 表）
  - 主键全 `uuid.UUID`；`User` 与 `Item` 一对多，删除级联

- `api/routes/*.py`：每个文件一个 `APIRouter(prefix=..., tags=[...])`，函数签名直接吃 `SessionDep, CurrentUser`，FastAPI 帮你做注入和文档生成。



## 二、数据库迁移（Alembic）

工具：Alembic + SQLModel.metadata。

### 关键文件

- `backend/alembic.ini`：只配了 `script_location = app/alembic`。

- `backend/app/alembic/env.py`：

  ```python
  # env.pyLines 22-25
  from app.models import SQLModel  # noqa
  from app.core.config import settings # noqa
  
  target_metadata = SQLModel.metadata
  ```

  - 把 `SQLModel.metadata`（包含所有 `table=True` 模型）作为「目标元数据」给 Alembic。
  - 通过 `settings.SQLALCHEMY_DATABASE_URI` 动态拼数据库 URL，不写在 ini 里，方便容器/CI 用环境变量覆盖。
  - 离线/在线两种迁移模式都开启了 `compare_type=True`，类型变化也能被检测。

- `backend/app/alembic/versions/*.py`：迁移历史。例如：

  - `e2412789c190_initialize_models.py`（初始化）
  - `d98dd8ec85a3_edit_replace_id_integers_in_all_models_.py`（主键改 UUID）
  - `9c0a54914c78_add_max_length_for_string_varchar_.py`、`fe56fa70289e_add_created_at_to_user_and_item.py`、`1a31ce608336_add_cascade_delete_relationships.py`

### 迁移触发链

容器或本地启动时都走的是 `backend/scripts/prestart.sh`：



```bash
#! /usr/bin/env bash

# prestart.shLines 1-14
set -e
set -x

python app/backend_pre_start.py     # 等 DB Ready（tenacity 5 分钟内每秒重试）
alembic upgrade head                 # 应用所有迁移
python app/initial_data.py           # 创建首个超管
```

Compose 里 `prestart` 服务的 `command: bash scripts/prestart.sh`，跑完后 `backend` 才启动（`depends_on.condition: service_completed_successfully`）。



### 你日常会用到的命令（在 `backend` 下）

```sh
uv run alembic upgrade head                        # 升级到最新
uv run alembic revision --autogenerate -m "xxx"    # 改完 models.py 后生成新迁移
uv run alembic downgrade -1                        # 回滚一格
uv run alembic history                             # 看版本树
```

> 注意：自动生成只是草稿，提交前一定要打开看，特别是改字段类型/重命名/数据搬迁。



## 三、网关与请求路由（Traefik）

项目里 Traefik 只在生产网关层介入；每个服务通过 Docker labels 声明自己的路由规则，Traefik 监听 Docker 事件自动生效。

### 三份 Compose 的分工

| 文件                   | 用途                                                         |
| :--------------------- | :----------------------------------------------------------- |
| `compose.yml`          | 共享/生产配置：服务、镜像、各自的 Traefik labels（路由规则、TLS 等） |
| `compose.override.yml` | 本地开发叠加：暴露端口到宿主、`develop.watch`、本地 traefik、mailcatcher、playwright 等 |
| `compose.traefik.yml`  | 独立的“前置 Traefik”：装在生产服务器上，单独管理 80/443 入口、Let's Encrypt 证书、Dashboard 鉴权 |

### 入口与端口

- 生产：`compose.traefik.yml` 启 Traefik，监听 `80` / `443`；`--entrypoints.http.address=:80`、`--entrypoints.https.address=:443`；`certificatesresolvers.le` 用 ACME TLS 挑战自动签证书。
- 本地：`compose.override.yml` 里也启了一个 `proxy: traefik:3.6`，绑 `80`、Dashboard 走 `8090`，命令包含 `--api.insecure=true` 用于本机调试。

### 服务怎么注册到 Traefik？看 `backend` 的标签

compose.ymlLines 118-138

```yaml
    build:
      context: .
      dockerfile: backend/Dockerfile
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik-public
      - traefik.constraint-label=traefik-public
      - traefik.http.services.${STACK_NAME?Variable not set}-backend.loadbalancer.server.port=8000
      - traefik.http.routers.${STACK_NAME?Variable not set}-backend-http.rule=Host(`api.${DOMAIN?Variable not set}`)
      - traefik.http.routers.${STACK_NAME?Variable not set}-backend-http.entrypoints=http
      - traefik.http.routers.${STACK_NAME?Variable not set}-backend-https.rule=Host(`api.${DOMAIN?Variable not set}`)
      - traefik.http.routers.${STACK_NAME?Variable not set}-backend-https.entrypoints=https
      - traefik.http.routers.${STACK_NAME?Variable not set}-backend-https.tls=true
      - traefik.http.routers.${STACK_NAME?Variable not set}-backend-https.tls.certresolver=le
      - traefik.http.routers.${STACK_NAME?Variable not set}-backend-http.middlewares=https-redirect
```

含义：

| 字段                                    | 作用                                          |
| :-------------------------------------- | :-------------------------------------------- |
| `traefik.enable=true`                   | 让 Traefik 处理这个容器                       |
| `traefik.docker.network=traefik-public` | 在哪张共享网络里能找到                        |
| `loadbalancer.server.port=8000`         | Traefik 转发到容器内的 `8000`（Uvicorn 端口） |
| `routers.*.rule=Host(api.${DOMAIN})`    | 基于域名匹配，`api.example.com` 到 `backend`  |
| `entrypoints=http/https`                | 接哪一个入口的流量                            |
| `tls=true` + `certresolver=le`          | 启用 HTTPS，证书由 Let's Encrypt 颁发         |
| `middlewares=https-redirect`            | HTTP 自动重定向到 HTTPS                       |

`frontend` 是同样套路，规则是 `Host(dashboard.${DOMAIN})`，指向容器内 `80`（Nginx）。

### 整体路由图（生产）

```
浏览器
  └── DNS 解析  api.example.com / dashboard.example.com / adminer.example.com
        ↓
    Traefik(80/443)
        ├── Host(api.*)        →  backend  : 8000   (FastAPI/Uvicorn)
        ├── Host(dashboard.*)  →  frontend : 80     (Nginx 静态)
        └── Host(adminer.*)    →  adminer  : 8080
```



### 本地开发是“另一种路由”

`compose.override.yml` 把 `backend` 直接 `ports: "8000:8000"`、`frontend: "5173:80"`，所以你不经过 Traefik 也能访问 http://localhost:8000/docs、http://localhost:5173；带域名的访问（`api.localhost.tiangolo.com` 等）则走本地那只 traefik。



## 四、前端架构（`frontend/`）

技术栈（来自 `package.json` 与目录约定）：React + TypeScript + Vite + TanStack Router + TanStack Query + Tailwind + shadcn/ui + 自动生成的 API 客户端。

### 应用入口

main.tsxLines 16-34

```tsx
OpenAPI.BASE = import.meta.env.VITE_API_URL
OpenAPI.TOKEN = async () => {
  return localStorage.getItem("access_token") || ""
}

const handleApiError = (error: Error) => {
  if (error instanceof ApiError && [401, 403].includes(error.status)) {
    localStorage.removeItem("access_token")
    window.location.href = "/login"
  }
}
const queryClient = new QueryClient({...})
```

要点：

- `VITE_API_URL`（开发取 `http://localhost:8000`，生产 `https://api.${DOMAIN}`）= 后端 base URL
- 把 JWT 存 `localStorage.access_token`，自动注入 `Authorization`
- 401/403 全局 → 清 token 跳 `/login`

### 路由

文件结构即路由（TanStack Router 文件式路由）：

```
src/routes/
├── __root.tsx              # 根布局
├── _layout.tsx             # 登录后的总体框架（侧栏 + 内容）
├── _layout/index.tsx       # 首页 (/)
├── _layout/items.tsx       # /items
├── _layout/admin.tsx       # /admin（管理员）
├── _layout/settings.tsx    # /settings
├── login.tsx
├── signup.tsx
├── recover-password.tsx
└── reset-password.tsx
```

构建时由插件生成 `src/routeTree.gen.ts`，`main.tsx` 里 `createRouter({ routeTree })` 加载。

### 客户端代码生成

`src/client/` 下的代码不是手写的，是基于后端的 OpenAPI 自动生成（项目里有 `openapi-ts.config.ts` 与脚本 `scripts/generate-client.sh`）。改了后端的 Pydantic/SQLModel/路由 → 跑一次生成 → 前端立刻拿到强类型 API 客户端。这是这个模板最大的爽点之一。

### 数据流

`组件` → `useQuery/useMutation` → `client/`（自动生成的函数）→ `OpenAPI.BASE + Authorization` → 网关/直连 backend。`useAuth.ts` 处理登录、登出、当前用户缓存。

### 组件分层

`components/ui` 是 shadcn 原子组件；`components/{Items, Admin, UserSettings, Common, Sidebar, Pending}` 是业务组件；页面在 `routes/` 里组合它们。



## 五、把这套放到「全栈框架」的位置看

你的直觉是对的——这是一个生产级骨架，常被拿来当起点：

它给你的“成品脚手架”包括：

- 后端：FastAPI + SQLModel + Alembic + Argon2/Bcrypt + JWT + Sentry + 邮件重置流 + pytest + 严格 mypy/ruff
- 前端：React + Vite + TanStack Router/Query + shadcn/ui + Playwright E2E + 自动生成 API 客户端（强类型从后端 OpenAPI 一键产出）
- 基础设施：多份 Compose（开发/生产/Traefik/Mailcatcher/Adminer）+ Traefik 网关 + ACME 证书
- DevEx：`uv` workspace + `prek`（pre-commit）+ `prestart.sh` 串好「等库 → 迁移 → 初始数据」+ Compose Watch 热同步

在它上面做二次开发的常见路径：

| 你想做的事              | 改哪里                                                       |
| :---------------------- | :----------------------------------------------------------- |
| 加业务表 / 字段         | `app/models.py` → `alembic revision --autogenerate -m "..."` → 检查迁移 → `upgrade head` |
| 加 API                  | 在 `app/api/routes/` 加文件，`api/main.py` 里 `include_router` |
| 加权限                  | 改 `api/deps.py`（按角色注入）+ 路由里 `Depends(...)`        |
| 加业务前端页面          | `frontend/src/routes/_layout/xxx.tsx` + 用生成的 client 调接口 |
| 改完后端同步前端 client | `bash scripts/generate-client.sh`（基于 OpenAPI）            |
| 上生产                  | `compose.traefik.yml` 起前置 Traefik；`compose.yml` 部署服务；改 `.env` 的 `DOMAIN`、`SECRET_KEY` 等 |

几个值得注意的小坑：

- `core/config.py` 里写死 `env_file="../.env"`：意味着期望在 `backend/` 下运行进程时读到根目录 `.env`；你换工作目录或部署时要保证这个相对路径成立，或干脆使用环境变量。
- `_enforce_non_default_secrets`：非 local 环境 `SECRET_KEY/POSTGRES_PASSWORD/FIRST_SUPERUSER_PASSWORD` 还等于 `changethis` 会直接 raise，避免上线裸奔。
- 自动生成的 Alembic 迁移不会处理“重命名列/数据迁移”，要手动改。
- 加新的 SQLModel 表时，必须能被 import到（通常通过 `app/models.py`），否则 `SQLModel.metadata` 拿不到，autogenerate 就不会写它。

