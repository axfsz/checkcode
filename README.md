# 分支差异检测服务 v1.2.0

按发布流水线 testa(main) → uat → 灰度(hotfix-prod) → 生产(prod) 定时检测 6 个 GitLab 仓库的分支差异，生成图示化报告，并提供 Web 页面查看历史报告、手动触发检测。

## 目录结构

```
branch-diff-service/
├── docker-compose.yaml
├── Dockerfile
├── .env.example           # 复制为 .env，填 GitLab Token 等
├── config/
│   └── config.json        # 仓库、分支、流向规则、定时、凭据方式
├── secrets/               # 可选：token 文件 / SSH key / CA 证书（只读挂载到 /run/secrets）
├── data/                  # 运行后自动生成
│   ├── repos/             # 仓库 bare 缓存（增量 fetch；_inventory/ 为分支清单用的全分支缓存）
│   ├── audit/             # 审计报表 <日期>_<时间>_<时间戳>/index.html、xlsx、csv；history_rewrites.jsonl
│   ├── inventory/
│   │   └── 20261028_083000_1793147400/   # branches_<目录名>.xlsx / .csv / summary.json
│   └── reports/
│       ├── 20261028_090000_1793149200/   # <日期>_<时间>_<Unix时间戳>
│       │   ├── index.html
│       │   └── summary.json
│       ├── latest.html
│       └── history.jsonl
└── app/
    ├── service.py         # 服务：定时调度 + Web + API
    ├── branch_diff_check.py   # 检测与报告生成（也可单独命令行运行）
    ├── branch_inventory.py    # 分支清单（也可单独命令行运行）
    ├── equiv.py               # 等价提交识别（cherry-pick / rebase / squash）
    ├── audit.py               # 代码提交审计（也可单独命令行运行）
    ├── htmlsheet.py           # 网页表格（标签页、筛选、排序、分页、导出）
    └── version.py             # 版本号
    └── git_auth.py        # Git 凭据配置
```

## 快速启动

```bash
cd branch-diff-service
cp .env.example .env && chmod 600 .env
vi .env                      # 填 GITLAB_USERNAME / GITLAB_TOKEN，建议同时设置 API_TOKEN

mkdir -p data
chown -R 1000:1000 data secrets      # 容器以 uid 1000 运行，见下方“运行用户”

docker compose up -d --build
docker compose logs -f
```

启动日志里会显示每个仓库的凭据自检结果：

```
[auth] OK  https://uggitlab.ugmid888.com/keepvent05/ug-transfer.git 4 个分支
[auth] ERR https://uggitlab.ugmid888.com/xg/uggame.git fatal: Authentication failed ...
```

打开 `http://<服务器>:8080`，点 **立即检测** 生成第一份报告。

## Git 凭据配置

在 `config/config.json` 的 `git_auth` 段配置，以下几种方式可以组合使用。凭据只写入容器内的临时 HOME（`/tmp/git-home`），不会写进镜像，也不会改动宿主机的 git 配置；日志和页面里出现的 token 会被替换为 `****`。

### 方式 1：Access Token（推荐）

在 GitLab 创建 Personal / Group Access Token，权限只需要 `read_repository`。

```json
"git_auth": {
  "credentials": [
    {"host": "uggitlab.ugmid888.com", "protocol": "https",
     "username_env": "GITLAB_USERNAME", "token_env": "GITLAB_TOKEN"}
  ]
}
```

`.env` 中填写：

```
GITLAB_USERNAME=ci-bot
GITLAB_TOKEN=glpat-xxxx
```

不想把 token 放在环境变量里，可以改用文件：把 token 写进 `secrets/gitlab_token`，配置改为 `"token_file": "/run/secrets/gitlab_token"`。

每个字段都支持三种写法：`xxx`（明文）、`xxx_env`（环境变量）、`xxx_file`（文件）。`credentials` 可以配置多条，对应多个 GitLab 地址。

### 方式 2：复用服务器上已有的 .git-credentials

1. 在 `docker-compose.yaml` 中取消这行注释：
   ```yaml
   - /root/.git-credentials:/run/host-git-credentials:ro
   ```
2. 在 `config.json` 中设置：
   ```json
   "git_auth": { "credentials": [], "credentials_file": "/run/host-git-credentials" }
   ```

宿主机文件属于 root 且权限是 600 时，容器内的 uid 1000 读不到。可以复制一份到 `secrets/git-credentials`，`chown 1000` 之后再指向它。

### 方式 3：SSH key

仓库地址需要改成 `git@uggitlab.ugmid888.com:keepvent05/ug-transfer.git` 这种形式。

```json
"git_auth": {
  "ssh": {
    "key_file": "/run/secrets/gitlab_ssh_key",
    "known_hosts_file": "/run/secrets/known_hosts",
    "strict_host_key_checking": "accept-new",
    "port": 22
  }
}
```

key 会被复制一份并设置为 600 权限，所以挂载进来的文件权限不对也没关系。

### 其他选项

| 配置 | 用途 |
|---|---|
| `ca_file` | 自建 GitLab 使用私有 CA 时，放 `secrets/gitlab-ca.crt`，填 `/run/secrets/gitlab-ca.crt` |
| `ssl_verify: false` | 自签证书且没有 CA 文件时临时使用 |
| `extra_config` | 任意 git config，例如 `{"http.proxy": "http://proxy:3128"}` |

如果 `git_auth` 整段都不配置，但设置了环境变量 `GIT_TOKEN`（以及可选的 `GIT_USERNAME`），会自动为 repos 中出现的所有 https 主机生成凭据。

修改凭据后，点页面上的 **检查 Git 凭据**（等同于 `POST /api/check-auth`），会对每个仓库执行 `git ls-remote` 来验证。

## 检测逻辑：发布流水线

```
testa(main) ──→ uat(uat) ──→ 灰度 groxy(hotfix-prod) ──→ 生产(prod)
```

在 `config.json` 的 `pipeline` 中按上游到下游的顺序配置。脚本会根据这个顺序自动生成三类检查。

| 类型 | 检查内容 | 级别 | 含义 |
|---|---|---|---|
| **逆向异常** | uat→testa、灰度→uat、生产→灰度 | 风险 | 下游有、直接上游没有：在下游环境直接改了代码，或者跳过了上游环境。需要同步回上游 |
| **未回合主线** | 灰度→testa、生产→testa | 风险 | 灰度或生产上的修复没进 main，后续发版可能被覆盖丢失 |
| **正向待晋级** | testa→uat、uat→灰度、灰度→生产 | 提示 | 正常的发布排队。停留超过 `pending_stale_days` 天时升级为警告 |

### 在途提交追踪

在途提交是指 main 上有、生产还没有的提交。报告会逐个列出每个提交当前走到了哪个环境，并显示在该环境停留了多少天。

- 停留天数从服务第一次看到该提交进入这个环境时开始计算，记录保存在 `data/reports/.state/` 下。
- 在 testa 的停留天数按提交时间估算。
- 刚部署时，uat、灰度、生产这几个环境的停留时间都从 0 开始累计，所以停留超时提醒会在运行几天之后才开始生效。

### 停留超时阈值

```json
"pipeline_rules": {
  "pending_stale_days": {"uat": 30, "hotfix-prod": 14, "prod": 7},
  "level_override": {"uat->main": "warn"}
}
```

- `pending_stale_days`：表示“等待进入某个环境”的天数阈值。例如 `"prod": 7` 的意思是，已经在灰度停留超过 7 天、还没上生产的提交会触发警告。
- `level_override`：用来调整某条检查的级别，key 的格式是 `源分支->目标分支`。

### 其他说明

- **统计口径**：merge 提交不计入。cherry-pick 过的等价提交视为已包含，所以生产修复用 cherry-pick 回合到 main 后，不会再报“未回合主线”。
- **缺分支的仓库**：缺失的环境会被跳过，上下游直接相连。例如仓库没有 hotfix-prod，就直接比较 uat 和生产。
- **拉取范围**：只拉取这 4 个分支。旧版本缓存的其他分支会在首次运行时自动清理，并执行一次 gc，所以这次运行会稍慢一些。

## 代码提交审计（v1.2 新增）

审计指定时间范围内各仓库环境分支（testa / uat / 灰度 / 生产）上的提交。

### 使用方式

- **页面**：在“代码提交审计”里选日期范围和仓库，点“开始审计”。
- **接口**：
  ```bash
  POST /api/audit
  {"since": "2026-10-01", "until": "2026-10-28", "repos": ["xg/uggame"]}
  ```
- **命令行**：
  ```bash
  docker exec branch-diff-service python /app/audit.py --since 2026-10-01 --until 2026-10-28
  ```
- **定时**：默认每天 08:10 审计最近 7 天。

结果保存在 `data/audit/<日期>_<时间>_<时间戳>/` 目录下，包含：

- `index.html`：网页版报表，所有结果都在一个页面里。
- `audit_*.xlsx`：Excel 文件。
- `audit_commits_*.csv`：提交明细。

### 网页报表

页面分成多个标签页，与 Excel 的 sheet 一一对应。每张表都可以：搜索、按列筛选、点击表头排序、分页、导出当前结果为 CSV。

| 标签页 | 内容 |
|---|---|
| 概览 | 关键指标、审计发现分布（点击可跳到对应明细）、每日提交、提交时段热力图、提交最多的人、按仓库汇总 |
| 审计发现 | 级别、规则、仓库、提交、人员、时间、环境、详情 |
| 提交人统计 | 每个人的提交数、合并数、增删行数、高中低发现数、非工作时间提交数、涉及仓库与环境 |
| 提交明细 | 作者、作者时间、提交人、提交时间、作者≠提交人、所在环境、直接提交到哪、文件数、增删行、文件列表 |
| 合并记录 | 合并人、时间、合入环境、来源分支、MR 编号、带入的提交数 |
| 推送记录 | GitLab 推送事件：推送人、分支、操作（推送 / 新建 / 删除）、是否强制推送（需要 API 权限） |
| 历史改写 | 服务拉取时发现的强制推送，以及丢失的提交 |

“分支清单”也改成了同样的网页形式，包含概览、负责人统计、全部分支，以及每个项目一个标签页。

### 审计规则

| 级别 | 规则 | 说明 |
|---|---|---|
| 高 | 疑似密钥 | 扫描新增代码中的私钥、AWS Key、GitLab / GitHub / Slack / Telegram Token、JWT、带密码的连接串、明文密码。报告中只显示打码后的值 |
| 高 | 强制推送 | GitLab 推送事件中，旧位置不是新位置的祖先 |
| 高 | 历史改写 | 服务每次拉取都会比对分支位置，发现改写就记录到 `data/audit/history_rewrites.jsonl`，并尽量匹配出推送人 |
| 高 | 删除环境分支 | GitLab 推送事件 |
| 中 | 改动敏感文件 | 按 `sensitive_paths` 匹配：生产配置、密钥文件、Dockerfile、CI、k8s、helm、部署、数据库迁移等 |
| 中 | 非工作时间提交 | 进入 `off_hours_branches`（默认灰度、生产）的提交或合并，时间不在 `work_hours` 内 |
| 中 | 直接提交到环境分支 | `direct_commit_branches`（默认生产）的第一父链上的非合并提交 |
| 中 | 邮箱域名不符 | `allowed_email_domains` 不为空时检查 |
| 低 | 超大提交 | 改动行数超过 `large_commit_lines`，或文件数超过 `large_commit_files` |
| 低 | 缺少需求单号 | `ticket_pattern` 不为空时检查，例如 `"[A-Z]+-\\d+"` |
| 低 | 时间异常 | 提交时间在未来，或者作者时间晚于提交时间 |

时间范围按**提交时间**（committer date）筛选。cherry-pick 和 rebase 会产生新的提交时间，所以会计入操作发生的那一天。

```json
"audit": {
  "enabled": true, "cron": ["10 8 * * *"], "days": 7,
  "branches": [],
  "direct_commit_branches": ["prod"],
  "off_hours_branches": ["hotfix-prod", "prod"],
  "work_hours": {"start": "09:00", "end": "21:00", "weekdays": [1, 2, 3, 4, 5]},
  "large_commit_lines": 2000, "large_commit_files": 50,
  "sensitive_paths": ["*.env", "*application-prod*", "*.pem", "Dockerfile", ".gitlab-ci.yml", "*/k8s/*", "..."],
  "secret_scan": true, "secret_scan_exclude": ["*.min.js", "*lock.json", "..."],
  "ticket_pattern": "", "allowed_email_domains": [],
  "use_gitlab_api": "auto", "api_token_env": "", "push_events_max_pages": 50
}
```

`branches` 为空时，审计流水线里的全部环境分支。推送记录和推送人需要 Token 具有 `read_api` 权限，否则这部分为空，其余审计不受影响。

## 如何判断“已同步”（等价提交识别）

不能只比较 commit hash。merge 合并不会改变 hash，但 cherry-pick、rebase、squash 都会生成新的 hash。比如同一个修复 cherry-pick 到 uat 之后，作者时间不变，提交时间和 hash 都会变。

git 自带的 `--cherry-pick` 是用 patch-id 判断的，而 patch-id 会把改动上下 3 行代码也算进去。分支分叉久了，同一个改动周围的代码不一样，patch-id 就对不上，导致误报。

服务按下面的顺序判断：A 分支上的一个提交，在 B 分支上是否已经有“等价提交”。

| 顺序 | 判定依据 | 适用场景 |
|---|---|---|
| 1 | **cherry-pick 标记**：提交说明里有 `(cherry picked from commit X)`，且 X 在 B 上；或者反过来 | `cherry-pick -x` |
| 2 | **改动内容一致**：只比较新增和删除的行，不看上下文，忽略空白 | 普通 cherry-pick、rebase |
| 3 | **作者 + 作者时间 + 标题一致**：作者时间精确到秒 | cherry-pick 或 rebase 时解决过冲突、内容略有不同 |
| 4 | **代码已包含**：该提交新增的有效代码行都已存在于 B 分支对应的文件中 | squash 合并、手工重新提交 |

结果的展示方式：

- 报告的每条流向都会写明“hash 不同的提交 N 个，其中 X 个已同步（按各依据分别计数）”。
- 卡片底部“已判定为等价”的部分可以展开，逐条列出依据和对应的提交，方便开发核对。
- 每个提交的指纹缓存在 `data/repos/_keys/` 下，所以后续运行很快。

每种依据都可以在配置里单独关闭：

```json
"equivalence": {
  "cherry_pick_trailer": true,
  "diff_content": true,
  "author_time_subject": true,
  "content_check": true,
  "content_check_max_commits": 3000,
  "content_check_max_files": 80,
  "content_check_max_lines": 5000
}
```

`exclude_cherry_picked: false` 会关闭全部等价识别，只按 hash 比对。

### 提交核对

开发对某个提交有疑问时，可以自己查：

- **页面**：在“提交核对”里选择仓库，输入 hash。
- **接口**：`GET /api/explain?repo=uggame&sha=798e02297d`
- **命令行**：
  ```bash
  docker exec branch-diff-service python /app/branch_diff_check.py --explain uggame 798e02297d
  ```

输出示例：

```
xg/uggame  798e02297d  fix: 申请详情接口契约展示问答与奖金 UG-3013
作者 xxx  作者时间 2026-10-21 18:10  提交时间 2026-10-23 19:01
  testa  main         包含（改动内容一致）  对应提交 d5ff9c8bf2 2026-10-21 18:10
  uat    uat          包含（同一提交）
  灰度   hotfix-prod  不包含
  生产   prod         不包含
```

## 分支清单（全部分支 + 负责人）

为每个项目列出全部分支，推断每个分支的负责人，输出成 Excel 和 CSV。

### 使用方式
- 页面上点击 **生成分支清单**
- 调用接口：`POST /api/inventory`
- 定时生成：默认每周一 08:30
- 生成的文件保存在 `data/inventory/<日期>_<时间>_<时间戳>/` 目录下，页面上可以直接下载

### Excel 内容

| Sheet | 内容 |
|---|---|
| 汇总 | 每个项目的分支总数、各分类数量、数据来源、创建人覆盖率 |
| 负责人统计 | 每个人名下的分支数（活跃 / 不活跃 / 过期 / 可删除），以及涉及哪些项目 |
| 全部分支 | 所有项目的分支放在一张表里，带筛选 |
| 每个项目一个 sheet | 分支、分类、负责人、创建人、MR 作者 / 状态、最后提交人 / 时间 / 说明、是否已合并到 testa / 生产、是否受保护、链接 |

### 负责人推断

Git 本身没有“分支所有人”的概念，按下面的优先级推断：

1. **分支创建人**：GitLab 推送事件里“创建分支”这条记录的推送人。
2. **MR 作者**：从这个分支发起的最近一个 MR 的作者。
3. **最后提交人**：分支最新一次提交的作者。

三列都会保留，`负责人来源` 这一列会标明这个负责人是按哪一种方式取到的。

### 分类规则

| 分类 | 判断条件 |
|---|---|
| 环境分支 | 流水线里的分支、受保护分支、默认分支 |
| 已合并可删除 | 已合并到 main，或者对应的 MR 状态是已合并 |
| 活跃 | 最后提交在 30 天内 |
| 不活跃 | 最后提交在 30 到 90 天之间 |
| 过期 | 最后提交超过 90 天 |

天数阈值可以在配置里调整。

### GitLab API

- 默认复用 git 凭据里的 token 调用 API，需要 `read_api` 权限（或 `api` 权限）。
- 如果 git 凭据用的是密码，或者 token 只有 `read_repository` 权限，API 会调用失败。这时会自动退回只用 git 数据：没有创建人、MR、受保护这几项，负责人取最后提交人。汇总页的“备注”列会写明原因。
- 可以通过 `inventory.api_token_env` 单独指定一个 API Token，例如在 `.env` 里加 `GITLAB_API_TOKEN=glpat-...`，然后配置 `"api_token_env": "GITLAB_API_TOKEN"`。
- GitLab 默认只保留一段时间内的推送事件，比较早创建的分支查不到创建人，会自动退回到 MR 作者或最后提交人。`events_max_pages` 用来控制最多往前翻多少页事件。

### 拉取方式

分支清单使用独立的缓存目录 `data/repos/_inventory/`，以 partial clone 方式拉取全部分支：只拉提交信息，不拉文件内容，所以分支多也很快。

```json
"inventory": {
  "enabled": true,
  "cron": ["30 8 * * 1"],
  "active_days": 30,
  "stale_days": 90,
  "merged_targets": ["main", "prod"],
  "use_gitlab_api": "auto",
  "api_token_env": "",
  "creator_lookup": true,
  "events_max_pages": 100,
  "mr_max_pages": 100,
  "partial_clone": true
}
```

## 定时计划

在 `config.json` 中配置，使用标准 5 段 cron 表达式，时区由 `.env` 里的 `TZ` 决定：

```json
"service": {
  "cron": ["0 9 * * 1-5", "0 17 * * 1-5"],
  "run_on_start": false,
  "auth_check_on_start": true
}
```

- 支持 `*`、`*/n`、`a-b`、`a-b/n`、逗号列表，以及 `@hourly`、`@daily`、`@weekly`、`@monthly`。
- 也可以用环境变量 `SCHEDULE_CRON="0 9 * * 1-5;0 17 * * 1-5"` 覆盖，多个表达式用分号分隔。
- 上一次检测还没结束时，本次定时会跳过。

每次检测都会重新读取 `config.json`，所以改仓库、分支、规则、凭据、定时之后都不需要重启。只有端口改动需要重启。

## 报告目录命名

由 `report_dir_name_format` 控制，默认 `{date}_{time}_{ts}`：

| 占位符 | 示例 |
|---|---|
| `{date}` | `20261028` |
| `{time}` | `090000` |
| `{ts}` | `1793149200`（Unix 秒级时间戳） |

默认格式生成的目录名例如 `20261028_090000_1793149200`。如果只要日期加时间戳，可以改成 `"{date}_{ts}"`。

超过 `report_retention_days`（默认 30 天）的报告目录会自动清理。

## Web 页面与 API

| 地址 | 说明 |
|---|---|
| `GET /` | 状态、配置、历史报告列表 |
| `GET /latest` | 跳转到最新报告 |
| `GET /reports/<目录名>/index.html` | 报告页面 |
| `GET /reports/<目录名>/summary.json` | 完整结构化数据 |
| `GET /api/status` | 是否在运行、上次结果、下次计划时间 |
| `GET /api/runs` | 历史报告列表 |
| `POST /api/run` | 立即检测，body 可选 `{"fetch": true, "notify": true}` |
| `POST /api/check-auth` | 凭据自检 |
| `POST /api/inventory` | 生成分支清单 |
| `GET /api/inventories` | 分支清单列表 |
| `GET /api/explain?repo=&sha=` | 核对某个提交在各环境分支上是否存在 |
| `POST /api/audit` | 代码提交审计，body：`since`、`until`、`repos` |
| `GET /api/audits` | 审计报表列表 |
| `GET /audit/<目录名>/index.html` | 审计报表（网页） |
| `GET /api/version` | 版本号与更新记录 |
| `GET /inventory/<目录名>/<文件>` | 下载分支清单 Excel / CSV |
| `GET /healthz` | 健康检查（compose 已配置 healthcheck） |

设置了 `API_TOKEN` 之后，POST 接口需要在请求头里带 `X-API-Token: <token>` 或 `Authorization: Bearer <token>`。页面上第一次点击按钮时会提示输入 token。

在 Jenkins 或其他系统里触发检测：

```bash
curl -X POST -H "X-API-Token: $API_TOKEN" http://branch-diff:8080/api/run
```

## 报告里的图表库

构建镜像时会下载 ECharts 到镜像内（`/static/echarts.min.js`），所以内网访问报告也能显示图表。

- 构建机访问不了 cdnjs 时，在 `docker-compose.yaml` 的 `build.args.ECHARTS_URL` 里换成内网地址。下载失败不会导致构建失败。
- 单独下载 `index.html` 文件打开时（比如从 Telegram 收到的报告），会自动回退到 CDN 加载图表库。

## Telegram 推送

1. 在 `config.json` 中设置 `notify.telegram.enabled: true`。
2. 在 `.env` 中填 `BRANCH_DIFF_TG_TOKEN` 和 `BRANCH_DIFF_TG_CHAT_ID`。
3. 把 `notify.report_base_url` 设为 `http://<服务器>:8080/reports`，推送消息里就会带上报告链接。

## 运行用户

容器以 `1000:1000` 运行，可以在 `.env` 里用 `PUID` / `PGID` 修改。`data/` 和 `secrets/` 两个目录需要对这个用户可读写（`secrets/` 只需可读）：

```bash
chown -R 1000:1000 data secrets && chmod 600 secrets/*
```

## 不用 Docker 时

```bash
cd app
CONFIG_PATH=../config/config.json GITLAB_USERNAME=... GITLAB_TOKEN=... python3 service.py   # 常驻服务
CONFIG_PATH=../config/config.json python3 branch_diff_check.py --fail-on-risk                # 单次检测
```

单次运行时，需要把 `config.json` 里的 `cache_dir` 和 `report_dir` 改成本机路径。

## 常见问题

**`Authentication failed`**
token 错误、过期，或者没有该仓库的权限。用页面上的 **检查 Git 凭据** 逐个仓库确认。

**`SSL certificate problem`**
配置 `ca_file`，或者临时设置 `ssl_verify: false`。

**`Permission denied`（写 /data 时）**
`data/` 目录的属主和容器用户不一致，参照“运行用户”一节处理。

**启动报 `环境变量 GITLAB_TOKEN 未设置`**
检查 `.env` 是否存在，以及变量名是否和 `config.json` 里的 `token_env` 一致。

**在容器里手动执行 git 命令**
镜像已经默认设置了 `HOME` 和 `GIT_CONFIG_GLOBAL`，`docker exec branch-diff-service git ls-remote ...` 会直接使用服务配置的凭据。前提是服务已经启动过一次，凭据文件已经生成。

## 版本记录

| 版本 | 内容 |
|---|---|
| 1.2.0 | 新增代码提交审计：提交人、时间、文件明细、合并记录、GitLab 推送记录、强制推送检测、密钥与敏感文件扫描；审计报表和分支清单都改为网页形式，并保留 Excel 下载 |
| 1.1.0 | 按发布流水线检测；分支清单与负责人；cherry-pick / rebase / squash 等价提交识别；提交核对 |
| 1.0.0 | 多仓库分支差异检测、图示化报告、定时任务与 Web 服务 |

**从 1.1 升级**：覆盖 `app/`、`Dockerfile`、`docker-compose.yaml`、`README.md`，在 `config.json` 中加入 `audit` 段，然后执行 `docker compose up -d --build`。镜像标签改为 `branch-diff-service:1.2.0`。
# checkcode
