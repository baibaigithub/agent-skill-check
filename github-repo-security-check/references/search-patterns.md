# GitHub 代码仓库只读静态检索模式参考（Tier 2）

本参考表用于在 Tier 2 阶段快速定位候选高危代码与配置。**检索命中仅作为线索，必须进入 Tier 3 精读上下文按类型核验（注入流查外部输入 → Sink；机密性泄露流查敏感源 → 6 类未授权出口；非数据流查直接证据链），严禁将搜索命中直接输出为漏洞。**

## 工具引擎、传参边界、对象预算与降级规范

1. **正则语法层级、传参边界与错误区分**：
   - 本文所有行内代码式均为**单层转义的 PCRE2 / ripgrep（`rg`）表达式**（即磁盘原文只有单个反斜杠如 `\s`、`\.`，**非**双层 JSON 转义串）；
   - **含引号模式的传参禁止简单套单引号**：凡表达式自身包含单引号或双引号（如第 4 节密钥匹配式含 `["']`），**严禁写成一律直接套单引号拼进 Shell 命令**！必须使用**参数数组传参**或写入**可信只读模式文件并通过 `-f <pattern_file>`** 传入；其余不含引号表达式亦优先走参数数组并带 `--` 参数终止符；
   - **默认引擎与状态码四分法**：默认采用支持 PCRE2 的 `rg`（或对象树 `git grep -n -I -P`）。执行任何检索必须严格区分以下 4 种返回状态，**严禁把引擎不支持或语法解析错误当作“无命中”**：
     1. **引擎不支持**（如未编译 PCRE2 支持 `-P`）→ 转入下述严格等价降级或将对应条款计为盲区；
     2. **语法解析错误**（非 0 且非 1 错误码，或报错输出）→ 修复调用传参或计入失败 `D`；
     3. **明确无命中**（退出码 1 且无报错/无截断）→ 计入 `C_done` 候选集核对；
     4. **输出截断或超限** → 按子树/扩展名分片补搜，未补全则计入 `D`。
   - **静态正反验证示例（以 `os\.system\(` 为例）**：正例 `os.system(user_cmd)` 命中；反例 `osXsystem(x)` 不匹配，说明 `\.` 匹配字面点号。
2. **POSIX ERE 严格等价降级规范（严禁删除 `\b` 宣称等价）**：
   - 若降级使用 `git grep -E` / `grep -E`（POSIX ERE），仅可使用经确认等价的替代式：`\s` → `[[:space:]]`，展开非捕获组 `(?:...)` → `(...)`；
   - **词边界 `\b` 处理硬规则**：**绝不允许直接删去 `\b`（如把 `\|safe\b` 改成 `\|safe`）再声称行为等价**（否则会误匹配 `|safely`）。若 ERE 方言不支持 `\b` / `\<` / `\>`, 必须改用等价边界表达（如针对 `\|safe\b` 写为 `\|safe([^[:alnum:]_]|$)`）并经过正/反例验证；无法提供等价替换的复杂 PCRE2 表达式**直接标为引擎覆盖缺口并转入 `D`/`B_scope`**。
3. **本地工作树防漏扫与逐项对账**：
   - 常规 `rg` 默认跳过隐藏文件（`.`）和 `.gitignore`/`.ignore` 规则命中的文件。本地检索必须显式禁用过滤（`--hidden --no-ignore --no-follow`），仅排除 `.git/` 元数据与经 Tier 1 核实不参与构建/安装/加载/发布的纯缓存 `A`；**严禁按目录名盲跳 `vendor/`、`dist/`、`build/`**；
   - 完成后核对 `Q_plan \ C_done` 集合缺口，任何未完成条目转入 `D`，绝不被动并入 `F`。
4. **非检出对象库（`--no-checkout`）预算预过滤与有界跨行条件**：
   - **先筛路径再分批检索（`-I` 与 `--depth=1` 非大小上限）**：执行 Tier 2 前先由 `git -C [temp_dir] ls-tree -r -l -z [commit_sha] -- :(literal)[subpath]` 获取全部条目，**按对象类型（仅普通 blob）、字节大小与可配置预算先过滤允许搜索的字面路径集合 `Q_plan`**（超限/二进制/LFS 指针/symlink/gitlink 先入 `D`），再通过 `--` + `:(literal)[path]` 分批执行 `git -C [temp_dir] grep -n -I -P`；
   - **明确跨行条件（不单靠 `rg -U`）**：单行正则会漏掉跨多行拼接或换行传参。有界跨行构造在 `rg -U`（工作树）或具备 PCRE2 多行能力时须显式写出**有界跨行正则条件**（例如 `(?:\r?\n|\s)+` 或 `[\s\S]{0,200}` 限制跨越长度防止回溯爆炸）；在仅单行 `git grep` 对象流场景下，通过第 3、5 节按语言分组的 **API 入口/导入/封装构造器单行线索**捕获候选，并在 Tier 3 用 `cat-file -p` 精读多行上下文；
   - **语言与框架范围限制**：主要覆盖 **Python、JS/TS、Go、Java、Rust** 及常见配置；若目标仓核心语言或关键框架接口（含未覆盖封装层）不在本表覆盖内，必须计入 `B_scope` 并阻断仓库级低风险结论。

---

## 1. 安装/构建/IDE 打开期入口与锁文件（Tier 1 优先 + Tier 2 辅助）

重点定位文件与触发链：
- 包清单与构建：`package.json`、`.npmrc`、`setup.py`、`pyproject.toml`、`requirements*.txt`、`*.pth`、`Makefile`、`CMakeLists.txt`、`build.rs`、`go.mod`、`pom.xml`、`build.gradle*`、`Gemfile`、`composer.json`
- **依赖锁文件**（小文件精读；超单次 256 KB 块的文本锁文件用分段读取或可信只读解析器**逐条全量核验**来源/版本/integrity，缺尾段/不可解析入 `D`；二进制锁文件如 `bun.lockb` 无可靠解析器时入 `D`；**应有但缺失锁文件入 `B_scope` 不改变 `N`**）：
  - `package-lock.json`、`pnpm-lock.yaml`、`yarn.lock`、`bun.lock`、`bun.lockb`、`poetry.lock`、`uv.lock`、`Pipfile.lock`、`Cargo.lock`、`go.sum`、`Gemfile.lock`、`composer.lock`、`gradle.lockfile`
- IDE 与开发环境打开期触发链：`*.code-workspace`、`.vscode/extensions.json`、`.vscode/tasks.json`、`.vscode/settings.json`、`.vscode/launch.json`、`.idea/*`、`.devcontainer/devcontainer.json`、`.envrc`、`mise.toml`、`.tool-versions`、`.nvmrc`
- Git、LFS 与 CI/容器：`.gitmodules`、`.gitattributes`、`.github/workflows/*`、`Dockerfile*`、`docker-compose*`
- 伴生 Agent 文件：`SKILL.md`、`.agents/`、`.claude/`、`.cursor/`、`.cursorrules`、`GEMINI.md`、`AGENTS.md`、`mcp_config.json`、`.mcp.json`、`hooks.json`

关键检索词：
- `preinstall|postinstall|prepare|folderOpen|postCreateCommand|initializeCommand|onCreateCommand`
- `recommendations|watcherTasks|interpreterPath|defaultInterpreterPath|nodePath`
- `resolved|tarball|git\+(https?|ssh)|registry=|replace\s+.*=>|hasInstallScript`
- `version https://git-lfs\.github\.com/spec/v1`（LFS 指针识别线索）
- `curl\s+.*\|\s*(sh|bash|zsh|python)|wget\s+.*\|\s*(sh|bash)`

## 2. 恶意后门、凭据窃取与预算受控混淆还原

- **敏感凭据路径与环境变量导出**：
  - `\.ssh|\.aws|\.kube|\.gcloud|\.docker/config|\.npmrc|\.git-credentials|Keychain|security\s+find-`
  - `os\.environ|process\.env|printenv|pbpaste|xclip`
- **外发与 Webhook**：
  - `discord\.com/api/webhooks|api\.telegram\.org|pastebin\.com|ngrok|webhook\.site|requestbin`
  - `/dev/tcp|mkfifo|bash\s+-i|nc\s+-e|crontab|LaunchAgents|\.bashrc|\.zshrc`
- **解码、规范化与混淆执行**（在可配置字节/时间/最大 5 层预算内只读还原并重追踪行为；超预算即停入 `D`）：
  - `base64\.b64decode|atob\(|Buffer\.from\([^)]*base64|base64\s+-d|bytes\.fromhex|codecs\.decode|decodeURIComponent|unescape\(|html\.unescape`
  - `eval\(|exec\(|new\s+Function\(|Function\(|__import__\(|importlib\.import_module`
  - 零宽字符 / Bidi 视觉欺骗字符 / 混合脚本同形字符（U+200B、U+200C、U+200D、U+FEFF、U+202A–U+202E、U+2066–U+2069）

## 3. 按语言分组的高危数据注入 Sink（含封装追踪与 AI/LLM 注入）

### 3.1 SQL / NoSQL 注入与查询构造（去除变量名依赖 + 有界跨行条件）
- **通用有界跨行拼接线索**（支持多行时使用 `[\s\S]{0,200}`，单行流改配下方语言与导入线索）：
  - `(SELECT|INSERT|UPDATE|DELETE)[\s\S]{0,200}(\+|\{|%s|\$\{)`
- **Python**：`\.execute\(|executemany\(|text\(|extra\(|\.raw\(|cursor\(`
- **JS/TS**：`queryRawUnsafe|executeRawUnsafe|sequelize\.query\(|\$where|\$regex|\.query\(|\.execute\(`
- **Go（不依赖 `db`/`tx` 变量名，先查包导入再查方法）**：
  - 导入：`database/sql|github\.com/jmoiron/sqlx|gorm\.io/gorm|github\.com/jackc/pgx`
  - 方法与拼接：`fmt\.Sprintf|\.(Query|QueryRow|Exec)(Context)?\(|\.(Raw|Where|Order|Group)\(`
- **Java**：`createStatement\(|executeQuery\(|executeUpdate\(|createNativeQuery\(|JdbcTemplate\.(query|update|execute)`
- **Rust**：`sqlx::raw_sql|sqlx::query\(|diesel::sql_query|format!\s*\(`

### 3.2 OS 命令注入与进程启动（补查解构、别名与封装模块）
- **Python**：`os\.system\(|os\.popen\(|subprocess|create_subprocess_shell|pty\.spawn\(`
- **JS/TS**：`child_process|node:child_process|execSync\(|execFile(Sync)?\(|spawn(Sync)?\(`
- **Go**：`os/exec|exec\.Command(Context)?\(`
- **Java**：`ProcessBuilder|Runtime\.getRuntime\(\)\.exec`
- **Rust**：`std::process::Command|Command::new\(`

### 3.3 模板注入（SSTI）vs XSS、路径穿越、反序列化与原型链
- **SSTI 候选（须核验外部输入进入模板源码或表达式）**：`render_template_string|jinja2\.Template\(|Environment\(.*from_string|ejs\.render\(|handlebars\.compile\(|Handlebars\.compile\(`
- **XSS / DOM 上下文候选（固定模板加变量时按 XSS 独立评估，不直接判 SSTI）**：`autoescape\s*=\s*False|text/template|dangerouslySetInnerHTML|innerHTML|outerHTML|document\.write|v-html|\|safe\b`
- **路径穿越与解压**：`extractall\(|send_file\(|sendFile\(|os\.path\.join\(|path\.join\(|filepath\.Join\(|Paths\.get\(|ZipInputStream`
- **反序列化 / XXE / 原型链**：`pickle\.loads?|marshal\.loads?|shelve|jsonpickle|yaml\.(load|unsafe_load)\(|torch\.load\(|ObjectInputStream|readObject\(|enableDefaultTyping|unserialize\(|Marshal\.load`
- **XXE 与原型链**：`resolve_entities\s*=\s*True|FEATURE_SECURE_PROCESSING|__proto__|constructor\.prototype`

### 3.4 AI / LLM 提示词与工具调用分发注入线索（不依赖攻击话术）
> 命中后必须进 Tier 3 精读上下文，区分**① 仅格式型 Schema 校验 vs 语义/白名单校验、② 实际权限（只读 vs 写删/外发）、③ 目标范围约束**。
- **外部输入 → 高权限消息/提示词构造**：
  - `(system_prompt|system_instruction|system_message|SystemMessage\(|SystemMessagePromptTemplate)`
  - `role["']?\s*[:=]\s*["'](system|developer)["']`（含 `["']` 引号，须经参数数组或可信 `-f` 文件传入！）
  - `(\.messages\.(append|extend)|chat\.completions\.create|responses\.create|messages\.create|generate_content)`
- **模型输出 → 有副作用工具调用与执行分发**：
  - `(tool_calls|function_call|tool_use|tool_choice|ToolCall)`
  - `(execute_tool|run_tool|call_tool|invoke_tool|dispatch_tool|mcp.*call_tool|tools_by_name\[)`

## 4. 数据泄露、硬编码密钥与 6 类机密性出口线索

- **硬编码密钥特征**（禁止外试验证有效性）：
  - `AKIA[0-9A-Z]{16}|AIza[0-9A-Za-z\-_]{35}|ghp_[0-9A-Za-z]{36}|github_pat_|sk-[0-9A-Za-z]{20,}|xox[baprs]-|BEGIN\s+(RSA\s+|EC\s+|OPENSSH\s+)?PRIVATE\s+KEY`
  - `(api_key|secret_key|access_token|jwt_secret|db_password)\s*[:=]\s*["'][^"']{8,}`（含引号，**须用参数数组或 `-f` 模式文件传入，勿直接套单引号**）
- **6 类机密性出口线索**（追踪敏感源 → 未授权接收方，核查字段/租户/接收方/用途）：
  - **1) 日志输出**：`(logger|logging|log)\.(debug|info|warn|warning|error)\(|console\.(log|info|warn|error)\(|fmt\.Print(f|ln)?\(|println!\(`
  - **2) 响应/堆栈与前端暴露**：`(NEXT_PUBLIC_|VITE_|REACT_APP_)[A-Z0-9_]*(SECRET|KEY|TOKEN|PASSWORD|PRIVATE)|traceback\.format_exc|res\.(json|send)\(|jsonify\(`
  - **3) 导出与转储**：`(csv\.writer|to_csv|to_excel|Content-Disposition.*attachment|send_from_directory|serveStatic)`
  - **4) 遥测与 APM**：`(sentry_sdk|opentelemetry|posthog|mixpanel|amplitude|telemetry|analytics)\.(capture|track|init|send)`
  - **5) 模型上下文投喂**：参见 §3.4 消息构造线索（核查是否拼入密钥、跨租户记录或无关隐私）
  - **6) 外部 HTTP/工具入参**：参见 §5 客户端线索（**含固定官方域名亦须核查超范围多传**）
- **跳过 TLS 与弱加密**：
  - `verify\s*=\s*False|rejectUnauthorized\s*:\s*false|NODE_TLS_REJECT_UNAUTHORIZED|InsecureSkipVerify\s*:\s*true|curl\s+-[a-zA-Z]*k`
  - `md5\(|sha1\(|Math\.random\(`（核查是否用于密码或安全令牌）

## 5. 按语言分组的 SSRF、封装客户端与认证/CORS/CSRF

- **Python HTTP 与封装客户端**：`requests\.(get|post|put|delete|request|Session)|httpx\.(get|post|Client|AsyncClient)|urllib\.request|aiohttp\.ClientSession`
- **JS/TS HTTP 与封装客户端**：`axios(\.(get|post|request|create))?|fetch\(|got(\.extend)?\(|undici\.|ofetch|http\.request\(|https\.request\(`
- **Go HTTP 与封装客户端**：`http\.(Get|Post|NewRequest|Client|DefaultClient)|resty\.New\(`
- **Java HTTP 与封装客户端**：`HttpURLConnection|java\.net\.http\.HttpClient|RestTemplate|WebClient|OkHttpClient`
- **Rust HTTP 与封装客户端**：`reqwest::|ureq::|hyper::Client`
- **认证、CORS 与网络配置**（CORS 须区分 `*`+凭据浏览器拒读 / 无凭据公开 / 反射 `Origin`+凭据高危，CSRF 独立评估）：
  - `verify_signature.*False|algorithms.*none|jwt\.decode\(`
  - `Access-Control-Allow-Origin|allow_origins|supports_credentials|Access-Control-Allow-Credentials|cors\(`
  - `0\.0\.0\.0|debug\s*=\s*True|privileged\s*:\s*true|/var/run/docker\.sock`

## 6. GitHub Actions CI 注入线索

- 仅针对 `.github/workflows/*.yml` 和 `*.yaml` 检查：
  - `pull_request_target`
  - `\$\{\{\s*github\.event\.(issue|pull_request|comment|review|discussion|head_commit)\.`
  - `\$\{\{\s*github\.head_ref\s*\}\}`
  - `permissions:\s*write-all`
