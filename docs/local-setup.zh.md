# ARTEX 本地运行调研报告

> 调研对象：artex-global（Windows 24H2 / PowerShell 环境）
> 目标：理清工程结构、技术栈与本地运行条件，为后续搭建环境提供依据。

---

## 1. 一句话结论

ARTEX 是一个 **Go 单体后端（可内嵌 Next.js 前端）+ PostgreSQL** 的 LLM 多 Agent 渗透测试系统。

本地跑起来的最小条件是：**Go ≥ 1.26 + PostgreSQL + 前端构建/开发服务器**；LLM Key 不是启动必需，只影响能否真正执行探索任务。

---

## 2. 当前机器环境现状

| 组件 | 状态 | 说明 |
|---|---|---|
| Node.js | ✅ v24.16.0 | Next.js 16 要求 ≥ 20.9，满足 |
| npm | ✅ 11.13.0 | |
| Go | ❌ 未安装 | `go.mod` 要求 1.26.3，CI 使用 1.26 |
| Docker | ❌ 未安装 | Docker Compose 路径暂不可用 |
| PostgreSQL | ❌ 未安装 | `psql` 不在 PATH |
| winget | ✅ v1.29.380 | 可用于安装 Go 等组件 |

仓库初始状态：不存在 `config.json`、`data/`、`web/node_modules`、`server/webui/dist`，属于全新未初始化状态。

---

## 3. 工程结构

| 目录 | 职责 |
|---|---|
| `cmd/artex/main.go` | 唯一入口：解析 `-addr/-proxy/-data`，启动 Manager、traffic 代理、HTTP 服务 |
| `server/` | `net/http` 全部 REST API、JWT 鉴权、SSE、任务/引擎/Agent 生命周期（最大包，约 148 个文件） |
| `agent/` | goals / planner / worker / mainagent 四类 Agent，基于 norma SDK；ToolSet 把双图暴露为 LLM 工具 |
| `db/` | PostgreSQL 单一数据源：资产图 + 探索图；`schema.sql` 随启动幂等执行 |
| `traffic/` | MITM 流量记录代理（:8788），SQLite 纯 Go 驱动做流量索引，存 `data/traffic/` |
| `guard/`、`intercept/` | 工具审批门与拦截规则 |
| `enrich/` | 异步 DNS / HTTP 补全 |
| `notify/` | 钉钉、飞书、企业微信、邮件等通知渠道 |
| `report/`、`evidence/`、`sidequestion/` | 报告、证据存储、侧问 |
| `llmrec/`、`llmpool/`、`mcphttp/` | LLM 调用记录、LLM 连接池、MCP 客户端 |
| `web/` | Next.js 16 前端；`next dev` 开发模式，`NEXT_EXPORT=1` 静态导出后可被 go:embed 内嵌 |
| `skills/` | 内置 3 个技能（api-recon / playwright-cli / scopesentry），运行时从磁盘加载 |
| `detections/`、`docs/`、`scripts/`、`selfupdate/` | 检测规则、文档、辅助脚本、二进制自更新 |

---

## 4. 技术栈

- **后端**：Go 1.26.3（纯 Go，`CGO_ENABLED=0`）
  - pgx v5（PostgreSQL 驱动）
  - go-mitmproxy（MITM 代理）
  - dnsx / miekg-dns（DNS）
  - golang-jwt、bcrypt
  - modernc.org/sqlite（流量索引）
- **Agent SDK**：`github.com/Autumn-27/norma v0.4.3`
  - 子包：agentcore / llm / harness / tool / permission / memory / transcript
- **前端**：Next.js 16.2.9 + React 19 + TypeScript + Tailwind 4 + shadcn/ui
  - next-intl（i18n）、zustand（状态）、@antv/g6（图可视化）
- **数据库**：PostgreSQL（**无需任何扩展**，官方使用 16-alpine，schema 兼容更老版本）
- **部署形态**：单二进制（前端静态产物通过 `-tags embedui` 内嵌）
  - 官方镜像：`ghcr.io/jiwoochris/artex-global:latest`

---

## 5. 后端启动链路（决定运行条件）

1. 定位配置文件：优先 CWD 下的 `config.json`（可用 `ARTEX_CONFIG` 覆盖）
2. `NewManager`：
   - 解析 DSN（优先级：`ARTEX_PG_DSN` > `config.json`）
   - **自动建库**（账号需有权限），随后执行 schema、seed 内置 Agent 与拦截规则
   - PostgreSQL 连接失败 → 直接退出
3. 启动 traffic 代理 :8788（**失败只打日志，不阻塞启动**），创建 `data/traffic/` 与 MITM CA
4. 生成/加载 `data/jwt.key`；无密码时**打印一次性 setup token**（或由 `ARTEX_ADMIN_PASSWORD` 直接初始化）
5. 监听 :8787；无 `embedui` 标签时前端页面返回 404 提示，需另起 `next dev`

---

## 6. 本地运行条件清单

### 6.1 硬性必需

| 条件 | 说明 |
|---|---|
| Go ≥ 1.26 | `go.mod` 为 1.26.3，CI 使用 1.26 |
| Node.js ≥ 20 + npm | CI 使用 22，本机 24 可用 |
| PostgreSQL | 推荐 16，无扩展依赖；账号需能建库或预先建好 `artex` 库 |
| Go module 网络 | `norma` 上游仓库已不可达，依赖 GOPROXY 缓存；`go.sum` 已有校验和，必要时用 `GOPROXY=https://goproxy.cn,direct` |
| npm registry 网络 | `web/` 依赖安装 |
| 配置来源 | `config.json` 的 `database` 字段，或环境变量 `ARTEX_PG_DSN` |
| 端口 | 8787（前端+API）、8788（流量代理）；dev 模式前端 5173/3000 |
| 目录 | `data/`（jwt.key、traffic 索引、CA）、`skills/`（已存在） |

### 6.2 非必需（影响功能）

| 条件 | 说明 |
|---|---|
| LLM Key | `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `ARTEX_LLM_PROVIDER|MODEL|BASE_URL`，或登录后在 UI「LLM 设置」配置；不配也能启动，只是探索引擎空闲 |
| 首次初始化 | `/setup` 页面 + 启动日志里的 `[auth] Setup token`，或 `ARTEX_ADMIN_PASSWORD`（8 位以上） |
| 外部工具 | curl、nmap、ripgrep、python 等（Agent 执行真实命令时才需要） |
| Playwright MCP | 默认禁用，本地不装也能跑 |

### 6.3 主要环境变量速查

| 变量 | 用途 |
|---|---|
| `ARTEX_PG_DSN` | PostgreSQL 连接串，优先级最高 |
| `ARTEX_CONFIG` | 指定 config.json 路径 |
| `ARTEX_ADMIN_PASSWORD` | 首次启动直接初始化管理员密码（8 位以上） |
| `ARTEX_SETUP_TOKEN` | 固定首次初始化令牌，不再随机打印 |
| `ARTEX_CORS_ORIGINS` | 跨域白名单，逗号分隔（默认 `http://localhost:5173`、`http://127.0.0.1:5173`） |
| `ARTEX_LLM_PROVIDER` | `anthropic` / `openai` |
| `ARTEX_LLM_MODEL` | 模型 ID |
| `ARTEX_LLM_BASE_URL` | 兼容 OpenAI 的自定义端点 |
| `ARTEX_LLM_PROXY` | LLM 请求代理（http/https/socks5） |
| `ARTEX_LANG` | 服务端默认输出语言（en/ko/zh/es/ja） |
| `ARTEX_SKILL_DIR` | 技能根目录，默认 `BaseDir()/skills` |

---

## 7. Windows 上的 4 个坑

1. **POSIX 脚本不可直接用**
   `start.sh` / `dev.sh` / `build.sh` 是 sh 脚本；Windows 用 `start.bat` 或直接运行 `artex.exe`。

2. **`build:static` 脚本语法**
   `"build:static": "NEXT_EXPORT=1 next build"` 是 POSIX 语法，PowerShell 中应改为：

   ```powershell
   cd web
   $env:NEXT_EXPORT="1"; npx next build
   ```

3. **dev 前端端口 / CORS**
   后端 CORS 默认只放行 `localhost:5173`，而 `npm run dev` 未指定端口（Next 默认 3000）。
   - 普通 API 走 rewrites 代理，同源无 CORS 问题；
   - 但 SSE 事件流在 dev 下直连 :8787，跑在 3000 会被 CORS 拦。
   - 解决：`npx next dev -p 5173`，或设置 `ARTEX_CORS_ORIGINS=http://localhost:3000`。

4. **Agent 工具执行（最重要的功能限制）**
   worker 的 Bash 工具和交互式 shell 依赖 Unix shell 与 pty（norma 依赖 `creack/pty`），官方运行环境是 Linux 容器（镜像内预装 nmap/curl/playwright）。
   - Windows 原生：可启动、登录、建任务、看 UI、连数据库；
   - 真实命令执行大概率不可用；
   - 完整功能建议 Docker Desktop 或 WSL2。

---

## 8. 四条可选运行路径（成本从低到高）

### 路径 A：纯前端 Mock（最快看 UI）

```powershell
cd web
npm ci
$env:NEXT_PUBLIC_MOCK="1"; npm run dev
```

- 需要：仅 Node
- 能验证：完整 UI（假数据，无后端/DB）
- 默认地址：http://localhost:3000

### 路径 B：开发全栈

```powershell
# 1) 准备 config.json（或直接设环境变量）
# 2) 启动后端
go run ./cmd/artex -addr 127.0.0.1:8787 -proxy 127.0.0.1:8788

# 3) 另开终端启动前端
cd web
npm ci
npx next dev -p 5173
```

- 需要：Go、PostgreSQL
- 能验证：API、登录、数据库、任务编排（工具执行受限）
- 前端地址：http://localhost:5173

### 路径 C：单二进制（最接近发布形态）

```powershell
# 1) 前端静态导出
cd web
npm ci
$env:NEXT_EXPORT="1"; npx next build

# 2) 同步到内嵌目录
cd ..
New-Item -ItemType Directory -Force server/webui/dist | Out-Null
Copy-Item -Recurse -Force web/out/* server/webui/dist/

# 3) 编译（-tags embedui 才内嵌前端）
go build -tags embedui -o artex.exe ./cmd/artex

# 4) 启动
./artex.exe -addr 127.0.0.1:8787 -proxy 127.0.0.1:8788
```

- 需要：Go、Node、PostgreSQL
- 能验证：生产形态，:8787 同源无 CORS 问题
- 地址：http://127.0.0.1:8787

### 路径 D：Docker Compose（官方推荐，功能最完整）

```powershell
Copy-Item .env.example .env
# 编辑 .env，至少设置 POSTGRES_PASSWORD
docker compose up -d
```

- 需要：Docker Desktop
- 自动拉起 `ghcr.io/jiwoochris/artex-global:latest` + `postgres:16-alpine`
- 含 nmap/playwright 等工具链，Agent 功能完整
- 地址：http://localhost:8787
- 查看设置令牌：`docker compose logs artex | findstr "[auth]"`

---

## 9. 首次启动与验证

1. 打开 `http://127.0.0.1:8787`
2. 首次进入 `/setup`，输入启动日志中的 `[auth] ... Setup token`
3. 设置管理员密码（用户名固定为 `ARTEX`，密码 8 位以上）
4. 健康检查：

   ```powershell
   Invoke-RestMethod http://127.0.0.1:8787/api/health
   # 期望返回：{"ok":true,"service":"artex","version":"dev"}
   ```

5. 在「系统配置 → LLM 设置」中填写模型参数后，方可创建并运行任务

---

## 10. 建议

| 目标 | 建议路径 |
|---|---|
| 先看界面 | A（Mock），10 分钟内可完成 |
| 改代码调试 | B（开发全栈），接受工具执行受限 |
| 完整功能验证（含 Agent 执行） | D（Docker Desktop）最省事 |
| 模拟正式部署 | C（单二进制） |

**推荐顺序**：先 A 验证前端可用 → 再装 Go + PostgreSQL 走 B/C → 需要完整 Agent 能力时再引入 Docker/WSL2。
