# GitHub 代码仓库安全威胁检查清单（6 大风险域）

本清单供 `github-repo-security-check` 在 Tier 1–Tier 3 筛查时逐域对照。取证前必须完成**前移最小化与局部掩码**（**严禁向外部 API 试用疑似凭据以验证有效性**），检测副本可规范化并保留原文行号，报告严格按场景编码且禁引入裸尖括号或可用链接。核验方法按风险类型区分：
- **数据注入流**：追踪外部不可信输入 → 校验/参数化/转义 → 敏感操作 Sink；
- **机密性泄露流**：追踪本机密钥、数据库字段、会话或其他敏感数据 → 未授权接收方（覆盖 6 类出口，按数据字段、账号/租户边界、接收方与用途定性）；
- **非数据流型风险**（硬编码凭据、缺失鉴权、CI/容器/权限配置缺陷、安装构建期固定恶意命令）：核验位置与非占位符有效性线索、暴露接口与鉴权检查、触发事件与授予权限、固定命令与触发入口；
- **封装与覆盖边界**：必须从语言、框架依赖、模块导入、别名与入口追踪底层封装，无法覆盖关键语言/关键框架接口时入 `B_scope` 并阻断仓库级低风险放行。

---

## 域 1：安装期、构建期与 IDE 打开期隐蔽执行（Pre-run RCE）

重点排查“还没运行业务代码、仅克隆/打开/安装依赖就中招”的陷阱：

- **Node.js / 前端生态**：`package.json` 中的 `preinstall`、`install`、`postinstall`、`prepare`、`prepublish` 脚本是否执行网络下载、解码执行或读写敏感路径；`.npmrc` 是否篡改 registry 指向陌生域名或携带 token。
- **Python 生态**：`setup.py`（`cmdclass`、顶层副作用执行）、`pyproject.toml` 自定义 `build-backend`、`*.pth` 启动注入文件、`requirements.txt` 中的陌生 URL/git 依赖或拼写仿冒包。
- **Rust / Go / Java / C++**：`build.rs` 内的网络/命令执行、`Makefile`/`CMakeLists.txt` 隐藏下载或覆盖系统路径、`go.mod` `replace` 指向未知源、`pom.xml`/`build.gradle` 恶意插件。
- **IDE 与本地环境打开期触发链（按触发机制盘点，文件名仅为线索）**：
  - 工作区文件与扩展：`*.code-workspace`（可内嵌 `tasks`/`settings`）、`.vscode/extensions.json`（推荐仿冒或恶意扩展）；
  - 自动任务与调试：`.vscode/tasks.json`（含 `"runOn": "folderOpen"`）、`.vscode/launch.json`、JetBrains `.idea/`（如 `watcherTasks.xml`/运行配置）；
  - 解释器与工具链路径劫持：`.vscode/settings.json`（覆写 `terminal.integrated.env.*`、`python.defaultInterpreterPath`、`eslint.nodePath`、Language Server 路径指向仓内文件）；
  - 容器与目录环境加载器：`.devcontainer/devcontainer.json`（`postCreateCommand`/`initializeCommand`/`onCreateCommand`）、`.envrc`（direnv）、`mise.toml`、`.tool-versions`、`.nvmrc` 等；无法确认目标 IDE 信任策略时标为依赖环境的风险。
- **Git 层与文档安装命令**：`.gitmodules` 指向不明外部仓或含 `..` 路径穿越、`.gitattributes` 可疑 `filter=`、`README*` 诱导执行 `curl ... | sh` 或下载闭源二进制。

## 域 2：恶意后门、凭据窃取、隐蔽外联、预算受控混淆与被动资产核验

- **凭据与本地隐私窃取**：代码读取 `~/.ssh`、`~/.aws`、`~/.kube`、`~/.gcloud`、`~/.docker/config.json`、`~/.npmrc`、`.git-credentials`、`.env`、浏览器 Cookie/密码库、加密钱包文件、macOS Keychain 或全量导出 `os.environ`/`process.env`。
- **隐蔽外联与数据回传**：硬编码陌生公网 IP、短链接、Pastebin、Discord/Telegram/Webhook 外发接口；DNS 外带（将敏感数据编码进子域名查询）；未在文档披露的后台遥测/埋点上报源码、路径、主机名或环境变量。
- **后门与持久化**：反弹 Shell（`/dev/tcp`、socket 绑定 shell）、定时任务（`crontab`/LaunchAgents）、篡改 `~/.zshrc`/`~/.bashrc`/`.git/hooks/`、预留硬编码万能口令或隐藏管理后门路由。
- **反分析、混淆、LFS 指针与可配置预算解码规范**：
  - 关注无构建说明的混淆/单行压缩 JS、Base64/Hex/XOR/URL/HTML 实体/Unicode 转义及多层组合编码喂给 `eval`/`exec`/`new Function`/`subprocess`、Unicode 零宽字符/双向控制符（Trojan Source，如 U+202E）及同形字符欺骗；
  - **可配置预算下的只读分层还原**：在可配置预算内（保守默认：最大 5 层、单次与累计解码字节及时间上限）只读解码并规范化（保留原文位置、每层变换与还原产物），严禁执行还原内容；按还原后行为定性；**超出解码预算或无法可靠还原则停止展开并记入 `D`**；
  - **LFS 指针与被动资产 `P` 核验**：若可执行/运行时/随包资产仅存在 Git LFS 指针而缺真实载荷，入 `D` 阻断放行；仅当通过魔数/头验证、无执行位/非 symlink/非 gitlink、且仅被动媒体/字体用途时可归入 `P`（归档/可执行/`SVG` 不入 `P`；魔数不符/Polyglot 升级高危或阻断；缺人工白名单时逐项核验）。

## 域 3：数据注入全类风险（Injection）

排查外部不可信输入（HTTP 请求、CLI 参数、环境变量、Webhook、文件上传、数据库脏数据、外部 API/LLM 输出）未经严格参数化或白名单校验流入危险 Sink（注意追踪多行构造、模块别名、解构导入与封装调用）：

- **SQL / NoSQL 注入**：
  - 字符串拼接/格式化构造 SQL（含跨多行构造：`f"SELECT ... {user}"`、Go `fmt.Sprintf`、Rust `format!`、JS 模板字面量、Java `+` 拼接）；
  - 各语言及封装后原生查询接口（Python `cursor.execute`/SQLAlchemy `text()`/`raw()`/`extra()`；JS/TS `$queryRawUnsafe`/`sequelize.query`；Go `database/sql`、`sqlx`、`gorm` 下任意变量名的 `.Query`/`.QueryRow`/`.Exec`/`.Raw`；Java `Statement.execute*`/`createNativeQuery`/`JdbcTemplate`；Rust `sqlx::raw_sql`）未绑定参数；
  - MongoDB 等 NoSQL 直接将用户 JSON 对象传入查询条件导致 `$ne`/`$where`/`$regex` 操作符注入。
- **OS 命令注入与代码注入**：
  - Python：`os.system`、`os.popen`、`subprocess.*(..., shell=True)`、`asyncio.create_subprocess_shell`、`pty.spawn`；
  - JS/TS：`child_process.exec`、`execSync`、`execFile`/`spawn(..., {shell: true})` 及解构/别名/封装调用；
  - Go / Java / Rust：`exec.Command("sh", "-c", ...)`、`Runtime.getRuntime().exec`、`new ProcessBuilder(...)`、`std::process::Command::new(...)` 拼接 shell 或可控参数；
  - 动态执行：`eval`、`exec`、`compile`、`new Function`、`vm.runInNewContext`（非安全沙箱）。
- **模板注入（SSTI）与 XSS / DOM 注入定级前提（严格区分）**：
  - **SSTI（服务端模板注入）**：**仅当外部不可信输入进入“模板源码本身”或“模板表达式求值位置”**（如 `render_template_string(user_input)`、`jinja2.Template(user_input).render(...)`、动态拼接模板语法字符串再编译渲染）时判定为 SSTI；
  - **XSS / DOM 注入（独立判断）**：**若模板源码为固定文件/常量，外部输入仅作为模板变量传入**（即使关闭了 `autoescape=False`、使用 Jinja `|safe`、Go `text/template` 输出 HTML、或前端 `innerHTML`/`outerHTML`/`document.write`/`dangerouslySetInnerHTML`/`v-html`、`javascript:` 协议），**不得直接判为 SSTI，而应按输出上下文（HTML/属性/脚本/URL）独立评估 XSS 风险**。
- **路径穿越与解压穿越（Path Traversal / Zip Slip）**：
  - 用户输入拼接到文件路径（`os.path.join`、`path.join`、`filepath.Join`、`Paths.get`、`Path::join`）后读写或删除文件，但未校验 `realpath`/`resolve` 仍位于预期根目录内；
  - 解压 `zip`/`tar` 归档（`tarfile.extractall`、`zipfile.extractall`、`ZipInputStream` 等）未校验成员路径是否含 `../` 或绝对路径/符号链接。
- **不安全反序列化、XXE 与原型链污染**：
  - Python `pickle.loads`、`marshal.loads`、`shelve`、`jsonpickle`、`yaml.load`/`yaml.unsafe_load`、`torch.load`（未设 `weights_only=True`）；
  - Java `ObjectInputStream.readObject`、`ObjectMapper.enableDefaultTyping`、PHP `unserialize`、Node `node-serialize`、Ruby `Marshal.load`；
  - XML 解析未禁用外部实体（XXE）；
  - JS/TS 递归深拷贝或 `merge` 未拦截 `__proto__`、`constructor`、`prototype` 导致原型链污染。
- **AI / LLM 提示词注入与工具调用分发注入（即使无攻击话术亦须排查两条链）**：
  - **链 A（外部输入 → 高权限消息构造）**：检查外部网页、文档、检索片段或用户输入是否未加隔离边界直接拼接进系统提示词/高权限消息角色（`system` / `system_instruction` / `messages`）；
  - **链 B（模型输出 → 有副作用工具分发）**：检查模型返回文本或 `tool_calls` 是否未经严格处理直接进入数据库、Shell、文件写删、邮件或外发工具；定性时必须严格区分**① 仅格式型 Schema 校验 vs 语义与范围白名单校验、② 工具实际生效权限（只读 vs 写删/外发）、③ 目标范围约束（固定单目录/白名单域名 vs 任意路径/URL）**。
  - *注：测试夹具、WAF/安全审计特征库中的模拟攻击字符串（无触发/加载路径）应记录排除依据，不判为投毒或注入。*

## 域 4：数据泄露、硬编码密钥与 6 类机密性出口（Confidentiality）

- **硬编码凭据与私钥**（非数据流型直接取证）：源码、配置文件、默认模板、前端代码中硬编码云厂商 Key（`AKIA...`、`AIza...`）、API Token（`sk-...`、`ghp_...`、`xoxb-...`）、`-----BEGIN PRIVATE KEY-----`、数据库密码或固定 JWT Secret（核验位置与非占位符线索；禁止外试验证）。
- **机密性数据流 6 类出口（追踪“本机密钥/数据库字段/会话/PII/内部配置 → 未授权接收方”，按数据字段、账号/租户边界、接收方与用途定性）**：
  1. **日志输出**：将密码、Token、Cookie、Session、密钥、数据库敏字段或 PII 写入明文日志或调试控制台；
  2. **前端与 API 响应 / 堆栈暴露**：前端前缀变量（`NEXT_PUBLIC_*`/`VITE_*`/`REACT_APP_*`）误曝服务端密钥；接口返回多余数据库内部字段（缺字段裁剪/DTO 过滤）、跨租户记录或完整异常堆栈与内部路径；
  3. **导出与文件转储**：CSV/JSON/Excel 导出、备份接口或未校验权限的静态目录/文件下载暴露 `.env`、`.git`、全表数据或跨账号快照；
  4. **遥测与 APM 回传**：未披露或超范围上报源码片段、环境变量、请求头/Body 凭据或客户 PII 到第三方统计/异常监控；
  5. **模型上下文投喂**：将跨租户记录、内部密钥或无关本地敏感文件无授权/无脱敏拼入发往外部 LLM 或共享会话的上下文；
  6. **外部 HTTP 请求与工具入参（含官方域名与封装客户端）**：向外部接口、Webhook 或工具参数发送敏感数据时，**即使目标为固定官方 API 域名，仍须核对是否超范围多传本地密钥、无关业务字段或跨租户数据**。
- **不安全传输与弱加密**：
  - 明文 `http://` 传输凭据或敏感业务数据；
  - 关闭 TLS 证书校验（`verify=False`、`rejectUnauthorized: false`、`NODE_TLS_REJECT_UNAUTHORIZED=0`、`InsecureSkipVerify: true`、`curl -k`）；
  - 使用 MD5/SHA1/无盐哈希存储密码，或使用非密码学安全随机数（`Math.random`、`random.random`）生成 Token/Session/重置码。

## 域 5：SSRF、CORS/CSRF、认证授权与访问控制缺陷

- **SSRF（服务端请求伪造）**：用户可控 URL、Host 或重定向目标直接传给服务端 HTTP 客户端（含封装后的自定义 Client/Factory：Python `requests`/`httpx`/`urllib`/`aiohttp`；JS/TS `fetch`/`axios`/`got`/`undici`/`ofetch`；Go `http.Get`/`http.Client.Do`；Java `HttpURLConnection`/`HttpClient`/`RestTemplate`/`WebClient`；Rust `reqwest`/`ureq`/`hyper`），且未限制协议（仅 `http`/`https`）、未拦截回环/内网网段（`127.0.0.1`、`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`[::1]`）及云元数据地址（`169.254.169.254`、`metadata.google.internal`）。
- **CORS 定级三类区分与 CSRF 独立评估（严格区分）**：
  - **情形 1（`Access-Control-Allow-Origin: *` 与 `Access-Control-Allow-Credentials: true` 组合）**：按 Fetch/CORS 规范浏览器会拒绝带凭据请求的响应读取；**不得仅凭这一组合配置即判为“认证数据可被跨域读取”**（标为配置错误或低危规范风险）；
  - **情形 2（公开 `Access-Control-Allow-Origin: *` 且无凭据）**：属公开无凭据暴露，仅当该接口本应限内网/私有（如内部管理面板/敏感非公开数据）时才按未授权暴露定级；
  - **情形 3（动态反射任意或弱校验 `Origin` 且同时设 `Access-Control-Allow-Credentials: true`）**：可导致第三方网站在携带用户 Cookie/凭据下跨域读取认证响应，定为**高危**；
  - **CSRF 独立判断**：无论 CORS 如何配置，**状态变更接口均须独立核验是否存在 CSRF 防护**（Anti-CSRF Token、`SameSite` 属性、自定义头/Origin+Referer 校验等）。
- **认证、会话与访问控制**：缺失鉴权的敏感接口、默认弱密码且未强制修改、JWT 允许 `alg: none` 或用 `decode` 代替 `verify`、Session Cookie 缺少 `HttpOnly`/`Secure`/`SameSite`、水平/垂直越权（IDOR：未校验资源与账号/租户归属权）、默认监听 `0.0.0.0` 暴露调试或管理端口。

## 域 6：第三方依赖供应链、CI/CD 与容器部署配置

- **依赖供应链与锁文件核验**：
  - 对小锁文件全量精读、对超过单次读取上限的大文本锁文件须通过分段读取或可信只读解析器**逐条核验全部条目**（指向不明私有源/任意第三方 Git 提交、完整性哈希缺失或异常、`hasInstallScript` 等），缺失尾段/不可解析/无可靠二进制锁文件解析器时入 `D`；
  - 仓库**应有但缺失对应锁文件**时计入范围盲区 `B_scope`（不改变 `N`）；排查依赖未锁定、引入已知高危废弃库或拼写仿冒包。
- **CI/CD 注入（`.github/workflows/`）**：
  - 使用 `pull_request_target` 并检出 PR 代码后执行构建/脚本（可窃取仓库 Secrets）；
  - 将 `${{ github.event.issue.title }}`、`${{ github.event.pull_request.body }}`、`${{ github.head_ref }}` 等不可信上下文直接拼入 `run:` Shell 命令导致 CI 命令注入；
  - 第三方 Action 未固定到完整 commit SHA 且授予过大 `permissions: write-all`。
- **容器与部署配置**：`Dockerfile` 使用 `curl | sh`、以 `root` 运行对外服务、镜像内打包 `.env`/私钥；`docker-compose.yml` 开启 `privileged: true`、`network_mode: host` 或挂载 `/var/run/docker.sock`、宿主机根目录 `/`。

---

## 误报区分矩阵（真实风险 vs 常见误报）

| 线索现象 | 倾向真实风险（需报告） | 倾向误报或降级（核实后剔除或降为低危） |
| --- | --- | --- |
| 命中 `eval` / `exec` / `subprocess` / 编码块 | 参数来自外部输入，或分层还原后指向隐蔽执行/外联（超预算/不可还原列 `D`） | 参数为静态常量或无害静态文案，且分层还原无危险行为 |
| 命中 SQL 拼接或原生/封装查询接口 | 拼接变量可追溯到外部输入且未参数化 | 仅拼接内部白名单表名/列名枚举，值均走参数化占位符 |
| 命中 `sk-...` / `password =` | 位于生产代码或真实配置中且具备真实有效格式特征 | 位于 `tests/`、`.env.example`、`docs/` 的明确占位符（如 `sk-your-key-here`） |
| 命中 HTTP 客户端（含官方域名与封装客户端） | `url` 可控未校验（SSRF），**或即使固定官方域名但超范围多传本地密钥/跨租户/多余敏感字段** | 目标固定合法且仅传实现业务必需的最小脱敏字段，无未授权泄露 |
| 命中模板渲染与 `autoescape=False`/`\|safe`/`innerHTML` | 外部输入进入模板源码/表达式（判 SSTI），或固定模板输出未转义入危险 HTML 上下文（独立判 XSS） | **固定模板 + 仅传变量时不误判 SSTI**；严格转义/净化后无 XSS 风险 |
| 命中 CORS `*` 与 `Credentials: true` / 反射 `Origin` | 动态反射任意/弱校验 `Origin` + `Credentials: true`（判高危可读），或私有无凭据接口开放 `*`，或状态变更缺 CSRF 防护 | `*` + `Credentials: true` 按规范浏览器拒读响应，不误判为认证数据可被跨域读取 |
| 命中图片/字体二进制文件 | 魔数不符（Polyglot）、带执行位、含主动内容（SVG）或无法核清（入 `D`/升级） | 魔数/属性/引用用途均核实为被动静态资源（归入 `P`） |
