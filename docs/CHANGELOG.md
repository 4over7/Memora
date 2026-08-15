# Changelog

All notable changes to Memora are documented here.

## [1.2.2] — 2026-08-16

> 这一版是一次全仓审计的产物，前后二十一轮交叉复审。把「审批判定」和「记忆采集」
> 两条链路从头看了一遍，逐条拿真实数据验证 —— 结果是：三个工具的采集其实一直是
> 坏的、审批的审计只覆盖了 13% 的判定，而**默认档位在特定配置下会替你拍板**。
> 都补上了，顺带把审批的打扰减了一大截。
>
> 安全修复占了这一版的大头。判定层的每一处改动都做了两件事：把修复改回去看测试
> 变不变红（约 60 条，凡是「改回去测试照样绿」的都补测试或如实标注测不了），
> 以及在 7311 条真实命令上量放行方向的净变化。

### 🗂 三个工具的采集其实一直没在工作

这三个都坏得很安静：不报错、不留日志，界面上就是「这个工具没有数据」，
和「你没用过它」长得一模一样。

- **Cursor**：对话早就搬到了 `state.vscdb` 里的另一张表（`cursorDiskKV`），
  采集还在查旧表 `ItemTable`。本机实测：旧表 0 条，新表 10 个会话、125 条消息。
  现在两张表都查，老版本 Cursor 继续支持。
- **Antigravity**：知识条目按 topic 组织、不带项目路径，而入库前有一道
  「项目路径为空就跳过」的门 —— 每一条都在写库前被丢掉。**更糟的是它还报告成功**：
  链路说「导入 5 个会话、零错误」，库里一条都没有。现在知识条目会按**引用文件
  是否真实存在**归到具体项目（拿引用里的相对路径去候选项目根下逐个试探，
  全部命中且只有一个候选做到才归；分不清就如实留着不猜）。
- **OpenCode**：只采了带文字的消息，工具调用一条没进库。同一个问题「这个 agent
  当时做了什么」，在 Claude Code 里 40% 的消息能回答、Codex 里 33%，OpenCode 是 0%。
  现在与另外两个一致。

顺带修了两个只在真机上才暴露的问题：改了解析逻辑**不会自动回填**已入库的会话
（有一道「源端没更新就整段跳过」的闸），以及 Cursor 的库快照改用 SQLite 的备份
接口 —— WAL 模式下直接复制文件拿到的是不一致快照。

### 📋 自动放行的决定终于进审计

Memora 替你放行的那些命令 —— 恰恰是你没看见的那些 —— 此前在审批历史里
**完全不存在**：只有人点过的决定才被记录。本机 97480 条风险判定里有 84213 条
（**86%**）查不到对应的决定。现在自动决定走与人工决定同一条记录路径。

同时修正了来源归属：Codex 的判定此前一律被记成 Claude Code，按工具筛选审计
会直接得出错误结论。

### 🔒 审批判定的加固

拿本机六万多条真实命令回放，堵掉一批能**跳过审批直接执行**的路径。这些不是
「多问一次」的成本，是「你根本不知道发生过」：

- 带引号的路径逃逸（`./'..'/../x` 在判定眼里是项目内，shell 眼里是项目外）
- 分离书写的递归删除标志、参数展开生成的 `--force`
- 大括号展开：`./{,../}f` 是**一个** token，bash 展开成两条路径 —— 一条在项目内、一条在外
- 白名单构建工具写项目外（`npx prettier --write ~/Documents` 会静默重写整个目录）、
  指定攻击者包（`--package` 的两种写法）、scoped 包名冒充白名单工具

网关体检也补了一处会**误报健康**的缺口：某些正则写法在 Rust 眼里合法、在
Claude Code 的 JavaScript 引擎里直接语法错误 —— matcher 用不了、hook 不触发，
而界面显示一切正常。

### 🚨 默认档位会替你拍板（重要）

**风险引擎选在「AI 建议 + 人工决定」（默认档）、同时开着飞书托管时，
低风险命令被直接自动批准了。** 你选的是「我要拍板」，系统替你拍了。

原因是同一条判据靠两个调用点各自记得：本地那条路自己判了档位，托管那条没判，
而共用的判定函数当时并不检查档位那一位 —— 它只判「这份 AI 结论够不够安全」。
现在档位判据收进唯一入口，新增调用点不会再漏。

同批还加了两道「结论自洽」的门：AI 说「破坏性」却同时说「低风险、建议放行」时
不再自动放行；置信度低于 0.7 也不放行。**两条都是零新增打扰** —— 4709 条真实
判定里，前者出现 0 次，后者最低值是 0.8。另外，AI 返回认不出的风险类别时改为
「判不出」而不是兜底成「执行类」（兜底会让一份编造类别的结论看着很完整）。

### 🔐 只读命令白名单里的执行入口

第二层「静默放行」认的是命令名，不看它能干什么。而这些命令自己带着口子：

- `rg --pre=<程序>` / `--hostname-bin=<程序>` —— **为每个文件执行指定程序**
- `ag --pager <程序>` —— 用指定程序当分页器
- `less '+!<shell 命令>'` —— `!` 是 less 的 shell 逃逸命令；
  **macOS 的 `more` 就是 less**，`-p` 同样是 less 命令
- `less -o <文件>` 写日志、`-k` 读可执行配置
- `uniq <输入> <输出>` —— 第二个**位置参数**是输出文件，直接覆盖（实测确认）
- `file -C` 写出 magic.mgc、`diff -l` 交给外部 `pr`、`nm --plugin` 加载共享库
- `env PATH=<我控制的目录> cat f` —— `cat` 命中白名单，跑的却是别的二进制；
  `LESSOPEN` / `RIPGREP_CONFIG_PATH` 同理（配置文件每一行都进命令行参数）

这类问题按构造是无穷的（每个命令行工具都有选项），所以最后一轮改了打法：不再
每轮挤出两三个，而是把白名单**全部 61 个成员逐个对照 man page 扫一遍**。
`ll` / `where` 直接移出去了 —— 它们在 macOS 上通常是别名或第三方程序，
「命令名叫这个」证明不了它只读。

同时**没有**按 `-o` 这类模式一刀切：`hexdump -o` / `strings -o` / `ps -o` /
`lsof -o` / `nm -o` 全是显示格式而不是输出文件，一刀切会把它们全误伤。

### 🛡 绕过判定链的那些快速路径

判据加对了，但有几条路根本没走到它：

- **SSH**：内层命令只比对名字，注释写着「参数不细查」—— 上面那些限制在远端
  全部失效，`sudo` 分支更是以 root 在远端执行。远端的写重定向也没查过
  （项目内外的判定是按**本地**目录算的，对远端路径毫无意义）
- **包装器**：`command` / `env` / `xargs` / `pnpm exec` 只比对目标命令名
- **信任脚本**：在解析之前就返回了，而「单条命令」的判据不认换行 ——
  「脚本一行、另一条命令另起一行」整条被信任放行

`xargs` 那条最后选了保守做法：它把标准输入的每一项追加到参数后面，**静态参数
里没有危险选项证明不了执行时没有**，所以对带危险选项的命令一律交人拍板。

### 🐚 shell 解析的语义修正

每一条都拿 bash 实测过：

- `>&<文件>` 在 bash 里是「标准输出和错误都写进该文件」，而判定把它记成了
  `&<文件>` —— 不以 `/` 开头，于是被当成项目内的相对路径放行
- 重定向目标遇到引号或转义空格会被截断（`> "/a b"` 只拿到 `/a`），
  而项目内外的判定就跑在这个截短的前缀上
- 续行只在**奇数个**反斜杠时才是续行；而且 shell 是删掉它、不是换成空格 ——
  换成空格会把跨行拼接的写目标拦腰截断
- `$(( ))` 是算术展开不是命令替换 —— 原先 `echo $((1 + 2))` 会被解析出一个
  叫 `1` 的命令，导致整条命令无谓地去问你（这是既有误拦，顺手修了）

### 🧠 喂给 AI 的判断依据

灰区判定现在会先去磁盘上查一遍，把「这个文件在不在、在不在 git 里、有没有
未提交改动」这类**测得到的事实**交给 AI，而不是让它靠命令字面猜。

配套的三条硬规矩：

1. **查不成就说「不知道」**，绝不用默认值凑 —— 工作区脏不脏这一位以前默认
   「干净」，而那个字段其实从来没人传
2. **不产出假事实**。只靠「参数里有斜杠」会把 sed 脚本、正则、命令替换的收口
   括号当成路径，报给 AI 一句「项目内、文件不存在」。真实语料实测：992 条事实
   里 608 条是这么来的，现在降到 143 条且都是语法上确实是路径的
3. **采集不能变成执行入口**，也不能拖慢审批 —— `git status` 会拉起仓库配置的
   外部程序，所以每次调用都显式关掉；三个 git 调用并发发出，实测审批延迟 18ms

喂给 AI 的内容也分了区：系统实测的事实与 agent 可控的输入（命令文本、对话片段）
明确隔开，可信区的每个值都做转义、不可信区的伪造分隔符靠随机串防住。

### ⚡ 少打扰

同一批回放显示，被推去人工审批的命令里最大一类根本不危险：`2>/dev/null`。
丢弃重定向落在项目外，判定据此拒绝放行，于是一个后缀就把纯只读命令推去问你。
另一类是 `nl`（只输出行号、没有任何写文件选项）不在只读命令集里。

两条修完，六万条真实命令里少打扰约 **1.5 万次**。

有意保留的收紧：递归删除一律审批，包括项目内的 `rm -rf build/` —— 因为
`rm -rf ./.git` 的目标也在项目内，而 git 恰恰是「项目内文件可恢复」这个前提本身。

## [1.2.1] — 2026-08-13

> 这一版只做一件事：让「审批网关」这个承诺经得起查。网关自己会不会悄悄失效、你看到的命令是不是完整的、事后能不能查清批过什么——三件都补上了。

### 🔒 网关不再会「悄悄失效」

Memora 的核心承诺是高风险命令先过人。这个承诺有个安静的失效方式：hook 还挂在配置里，但**已经拦不住了**——二进制被清理脚本删了、别的工具重写了 `settings.json`、Codex 的总开关被关掉。界面一切正常，而每条命令都在无人审批地跑过去。

- **网关体检**：现在启动时以及每轮同步（60 秒）都会验一次「拦不拦得住」，坏了就在主界面顶部挂一条播报（故意不给关闭按钮）。判据不再是「配置里出现过 memora-hook 这个词」——`memora-hook --version` 也满足那个形状却什么都不审批。现在要求：PreToolUse 条目在、命令实际会执行 `pre-tool-use` 这个事件、`--source` 对得上、二进制当前用户能执行、matcher 覆盖到要拦的工具、`type` 是 command、`timeout` 不至于小到 hook 还没拿到决定就被宿主杀掉，以及 Codex 的 `[features].codex_hooks` 总开关是开着的。
- **区分「没装过」和「装过又坏了」**：从没装过的工具不会被误报成故障；而升级时就已经坏掉的网关——最该收到警告的那种——不会再被永久静音。主动卸载（`memora-hook uninstall`）会留下凭据，播报条随之消失。

### 👁 命令看得全

- **不再掐掉命令尾巴**：Beacon 与审批中心过去都是截断末尾显示，而尾巴恰恰是追加破坏性后缀最常见的位置（`… && rm -rf ~/`）。拿本机近三万条真实命令量过：中位数约 300 字符、**65% 超出可见区**——不是边角情况，是常态。现在改成**省略中间**，头尾都留，并写明「已省略 N 字符」；保留多少按实际渲染宽度测算，不写死。
- **审批中心显示完整命令**：可滚动、可选中，不截断。
- **删掉 Allow All 按钮**：hook 协议里没有「持久授权」这回事（决定只对这一次调用生效），那个按钮到边界会塌成普通「允许」、不产生任何规则——和「允许」一模一样，却让人以为授权了「以后都这样」。真正的持久授权是 **Allow Forever**：它走 Memora 自己的指令授权记忆（你在 Agent 页确认 pattern 后才写成规则），而且因为规则留在 Memora 而不是写进 Claude Code 的权限表，每次自动放行仍然有记录、可审计、可撤销。

### 📋 事后查得清

- **审批历史 66% 的记录看不到命令**：命令的可读形式存在一个字段里，而历史只读了另一个——抽不到就把 `{}` 原样显示出来。本机九万多条历史里三分之二是这样：你知道自己批过，却不知道批的是哪条。已修，实测 **66% → 0%**。
- **长命令不再被截掉中段**：审计记录的单条上限从 500 提到 2000 字符，完整保留率 **66% → 96%**，日志体积 1.6×（已按天分片，量级可接受）。

### 🐛 其它修复

- 回执文件的路径不再用宿主给的 ID 拼接——含 `../` 的 ID 曾能把回执写到 `~/.memora/` 之外的文件上。
- 「同一条请求只许一个决定」的闸改成跨进程、跨重启有效：App 重启后不会再把已经拍过板的请求重新弹出来、也不会用第二个决定覆盖第一个。
- 三个决定入口（Beacon 点击 / 飞书 / 审批中心）现在共用同一道闸，且写决定前会确认请求还挂着——被别处解决的请求不再显示成「已处理」。
- Codex 侧超时后收到迟到的「允许」时，不再落进超时拒绝——你批准了，宿主执行的就是批准。
- 安装 / 卸载写 `settings.json`、`hooks.json`、`config.toml` 改为原子写，并保留原文件权限与软链接：写到一半不会毁掉你写在同一个文件里的其他 hook。
- Continue 从历史检查点开分支时，不再混入检查点之后的消息与提交。
- monorepo 子目录项目现在能看到自己的 commit 与提交数。

## [1.2.0] — 2026-08-11

> 审批从「问不问你」升级成「你在不在」：需要你拍板的请求先在本地弹，你不在才追到手机上。同时根治了采集链路的性能问题——同步耗时降到原来的四分之一。

### 🆕 新增

- **在场感知审批（双活通道）**：需要你拍板的请求现在先在 Beacon 弹出；本地在**宽限期**（默认 45 秒）内没人理，才把卡片镜像到飞书。你在电脑前批掉的请求根本不会发到手机上——既不刷屏，也省飞书调用额度。发卡之后 Beacon 与飞书**同时有效**，你先点哪个算哪个。宽限期在「设置 → 通知渠道 → 飞书」里可调（不等 / 15 秒 / 45 秒 / 2 分钟），设为「不等」即恢复旧的立即发卡行为。
- **事后从审批历史补标误判**：被打扰的当下常常来不及表态，随手点了「允许」，过一会儿才想起「刚才那条根本不该问我」。现在打开审批中心 →「历史」→ 点任一条，详情底部可以标「不该问我」/「早该拦住」。标记会进入 Agent 页等你确认 pattern，确认后才写成规则。
- **事件日志按天分片**：审计日志改为每天一个文件，写满即封存。

### ⚠️ 行为变更

- **命中内置 deny-list 的命令不再被自动拒绝**，改为和其它需要拍板的请求一样弹给你决定（默认倾向拒绝）。原先托管模式下这类命令直接静默拒掉——最该由人判断的一类，反而是唯一不给人看的一类。安全性不依赖「自动拒」兜底：等待超时后 Claude Code 会把该次调用交还终端二次确认，Codex 则直接拒绝，都不会静默放过。

### ⚡ 性能

- **同步耗时 3.4–4.5 秒 → 0.9 秒**。三个根因：commit 关联每轮把整个事件日志从头读一遍（2 秒 → 1 毫秒）；一批 transcript 因为增量水位记错了对象而**每轮重复解析**（每次 163 MB → 0）；被跳过的项目不记水位，同样每轮白解析一遍。
- 数据保护快照不再为几十 KB 新增内容反复重写整个事件日志。

### 🐛 修复

- **Token 统计虚高**：会话压缩时被重放进 transcript 的消息，其 token 用量会被重复累加。实测本机 10 个会话受影响，其中一个真实 22.1M 被记成 34.7M（+57%）。
- **会话消息数不准**：部分会话显示的条数与实际不符（两个方向都有），根因是增量水位与会话身份用了两把不同的钥匙。
- **审批回执可能被覆盖**：飞书上批准与 Beacon 上拒绝先后落到同一个文件时，后写的会覆盖先写的——现在同一条请求只允许写一次回执。
- **飞书上引用旧卡片回复，决定会被套用到不相干的请求**：飞书无法灰化已处理的卡片，引用一张旧卡回复时不再兜底匹配到当前请求，而是明确忽略并说明原因。
- **已执行完的命令过一会儿又弹出来问你**：排队中的请求收不到自己的「已完成」标记，现在每轮扫描都会消化全部标记。
- **飞书通道关掉再打开后，之后的决定被静默丢弃**。
- **同一条反馈重复触发时无法重试**、**审批期间应用退出后回执处理**等一批时序问题。
- 快速连点两项设置时，后写的会覆盖先写的（表现为「刚点的那项自己弹回去」）。
- 连点两条会话时界面会停在先点的那条；快速切换时间窗会显示上一个窗口的数字；切换项目时上一个项目的黑板 / 漂移结论可能出现在新项目下。

### 🔒 安全

- **URL 里的账号密码不再外泄**：`https://user:password@host` 这类写法此前未被脱敏，会随命令进入发给 LLM 供应商的请求，以及本地的反馈与规则文件。现已补上（主机名保留，便于判断风险）。
- **终端启动脚本移出 `/tmp`**：改写入当前用户私有的临时目录，避免同机其他用户预置同名文件借道执行。

### 🌍 界面

- 项目分组、信任脚本、Agent 页、风险引擎档位说明等约 150 处文案此前只有中文，现已全部支持中英切换。

### 🧪 验证

- Flutter 987 项测试、Rust workspace 全量测试、静态分析通过；Apple Silicon 与 Intel 双架构包均已完成 Apple 公证。

---

📦 提供 Apple Silicon (aarch64) 与 Intel (x86_64) 两个架构包。

## [1.1.5] — 2026-07-21

> 搜索与审批体验增强，并完成两轮安全、凭据和数据完整性加固。

### 🆕 新增

- **审批历史可查看完整命令**：审批中心「历史」列表的命令原先单行截断（长命令、`update_plan` 等 JSON 参数串看不全），现在**点击任意一条**弹详情——完整命令 / 参数（JSON 自动缩进美化、等宽、可滚动、可选中）、元数据（发起 agent / 项目 / 决定渠道 / 精确到秒的时间 / 风险等级 / 分类 / deny-list），并有「复制命令」一键复制原始命令。
- **审批历史标注发起 agent（Claude Code / Codex）**：每条历史记录标出是哪个 AI 发起的（身份色圆点 + 名字），详情里也有「发起」项。判别取自 `events.jsonl` 的 `source` 字段，**不看 tool_name**（tool_name 可能含 `mcp__codex_apps` 之类，实为 Claude Code 调的 MCP）。
- **搜索结果排序**：关键词搜索支持按相关性、最新和最旧排序；会话 ID 角标和右键菜单补充复制入口。

### 🔒 安全与凭据

- **LLM API Key 不再写入明文偏好设置**：凭据改存 macOS Keychain；兼容迁移已有明文字段，并覆盖“账户元数据已存在”与 stale Keychain 冲突场景。项目内 `.env` 被 Git 忽略，仓库不再新增明文密钥。
- **Hard Allow 收紧敏感读取**：读取真实 `.env`、SSH / 云凭据、私钥证书、完整环境变量或 Keychain 密码不再无提示自动放行；`.env.example` 等公开模板仍可正常读取。
- **单用户审批边界明确化**：未启用的飞书群聊审批入口继续保持注释；凭据修改增加并发保护，避免多个设置实例互相覆盖。

### 🐛 修复

- **会话详情页顶部 session id 角标点击复制失效**：该角标此前只有几乎不可见的小圆点可点、点短 id 文字无反应（点击被根 `SelectionArea` 的文字选择拦走）。改为整条角标都可点，点击即复制完整 session id。
- **compact 归档重复采集**：旧 transcript 归档移入独立目录，并兼容忽略历史 `-precompact-` 文件，避免旧会话被反复解析或覆盖活动会话元数据。
- **SQLite 错误不再伪装成空数据**：行解码失败从 Rust 核心、FFI 到界面完整传播；会话、搜索和同步显示明确失败，不再静默丢行或显示“暂无数据”。
- **LLM 活动账户不变量**：删除当前账户后自动切换到剩余账户，异步凭据保存完成后再刷新界面。
- **更新下载资源释放**：下载超时或中断时先关闭文件流再删除 partial DMG，避免文件描述符滞留。

### 🧪 验证

- Flutter 817 项测试和 Rust workspace 常规测试通过；静态分析、Clippy、macOS 构建及双轮代码审查通过。

### 🛠 内部

- 修复 Flutter 3.38 原生资产缓存可能让 Intel 发行包混入 arm64 `objective_c.framework`；发行构建现在强制重新合成 native assets，并继续逐文件校验目标架构。

---

📦 提供 Apple Silicon (aarch64) 与 Intel (x86_64) 两个架构包。

## [1.1.4] — 2026-06-22

> v1.1.3 数据保护瘦身的收尾与加固。

### 🐛 修复

- **数据保护快照瘦身更彻底**：`.gitignore` 进一步排除可再生的开发样本 / 评估数据（tool-use-samples / risk-annotated-bash / deepseek-eval-results）与 beacon 运行态缓存；存量仓库下次同步会自动收敛移出。核心数据（对话事件 / 会话摘要 / 反馈 / 规则）保护不变。
- **时钟回拨健壮性**：系统时钟回拨时，自动保存不再被误判持续阻断（改为 fail-open）。

### 🛠 内部

- 重建瘦身脚本加固：运行中检测、Ctrl-C / kill 自动回滚、备份防碰撞、验证 fail-safe。
- 修复 x86_64 跨架构打包偶发的 `pub.dev` socket error（cargokit 依赖解析改离线优先）。

---

## [1.1.3] — 2026-06-22

> 数据保护仓库（`~/.memora/.git`）膨胀根治：单台从 10GB+ 瘦身到 12MB。

### 🐛 修复

- **`~/.memora/.git` 无限膨胀根治**：用于数据保护的自管 git 快照仓库原先每分钟全量 `git add -A` 提交、永不 gc，且 `.gitignore` 只排除了数据库，导致 `raw/`（原始对话副本，约 2GB）、`events.jsonl`、各类 `*.log` 被反复快照，`.git` 单台撑到 10GB+（单日增长约 400MB，磁盘吃紧时持续恶化）。
  - 提交频率从每分钟降到每小时；每次提交后自动 `git gc` 回收冗余对象。
  - `.gitignore` 扩展排除 `raw/`、所有 `*.log`、`logs/`、`.git.old*`；存量仓库在升级后下次同步会自动收敛（把已追踪的大文件移出快照），无需手动操作。
  - 新增重建瘦身脚本 `scripts/rebuild-memora-git.sh`（dry-run/`--apply` + 改名备份 + 失败回滚），可一次性把现有臃肿仓库瘦身。核心数据（对话事件、会话摘要、规则、反馈）保护不变，工作区数据库全程不受影响。

### 🧪 测试

- 补充测试基建：单例 reset + reply 独立节流回归。

---

## [1.1.2] — 2026-06-14

> 飞书 API 额度优化 + 自动更新安全/健壮性加固（v1.1.1 后的修复批次）。

### 🔒 安全 & 稳定性

- **飞书托管 API 用量根治**：审批轮询从固定 2s/卡 改为按卡龄退避（新卡 5s → 老卡 5min）+ 卡片 TTL 自清理（超 26h 移出轮询），堵住长期挂起 / 泄漏卡持续消耗飞书 API 额度（原会烧穿 100 万/月）；文字回复改独立 15s 节流。
- **自动更新发布身份校验**：安装前绑定 bundle id + Team ID + Gatekeeper/公证；版本防回放降级（不安装比当前更旧的包）；helper 脚本路径全 shell 量化防注入。

### 🛠 健壮性

- 下载加连接 / 空闲超时 + 总时长 / 字节上限；当前架构无匹配安装包时引导手动下载；安装失败写标记 + 重开旧 app，下次启动提示。
- 挂载点解析支持含空格 / 中文的卷名；安装原子替换的回滚与 bundle 污染防护。

---

## [1.1.1] — 2026-06-13

> v1.1.0 设计大版本后的修复与打磨：主题模式可手动选、临时会话不再堆积空壳项目、合并相关的路由与误报修复。

### ✨ 新增

- **主题模式手动配置**：设置 → 通用 → 主题，除「跟随系统」外可强制「浅色 / 深色」，选择持久化。

### 🗂️ 项目分组

- **临时会话归一**：一次性 review / worktree checkout（`/private/tmp` 等）不再各自建项目堆积，统一归到一个「临时会话」项目，真实项目列表保持清晰。
- **合并建议「路径重叠」误报修复**：待合并项目里已有现成分组时，建议并入它而非新建（避免掏空旧组、留下同路径残组）；接受前的重叠预检不再把参与合并的组误报为冲突。

### 🔒 安全

- **自动更新校验加固**：安装前绑定 Memora 官方发布身份（bundle id `com.eastworld.memora` + Team ID + Gatekeeper / 公证评估），不再只验签名结构完整，挡掉被替换 / 任意签名的安装包；helper 脚本路径全部 shell 量化，防含特殊字符的安装路径触发命令注入。

### 🐛 修复

- **合并组路由**：合并多个子项目后，会话详情 / commit 的「续接」「分支」「git show」改用各成员真正归属的 repo，不再串到主项目。
- **commit 定位**：commit owner 映射改用完整 hash，避免合并组里两个仓库 short hash 碰撞路由到错的 repo。
- **安全**：终端打开 commit（git show）的路径 / 命令做 shell + AppleScript 转义，含空格 / 特殊字符的路径不再破坏命令。
- history 分支创建的时间戳查询按项目隔离，避免跨项目 short hash 串。

---

## [1.1.0] — 2026-06-12

> **全新设计语言「黑曜石中的琥珀光」+ Light 模式**。从 AI 随手定的紫色样式重做为「档案琥珀 × 黑曜石」的时光机界面；首页重构为「稳定流」时间轴；新增跟随 macOS 系统外观的 light / dark 双模式。

---

### 🎨 新设计语言「黑曜石中的琥珀光」

- **色彩**：主色改为档案琥珀 `#D6A24A`（被封存的时间 = 全 app 唯一的「光」），底色黑曜石分层中性 + 暖白纸感文字。建立语义化 token 体系（禁止硬编码色）。
- **字体**：SF Pro（标题 / 正文）+ SF Mono（ID / 时间戳 / 代码），Apple-native。
- **深度**：发丝线分层 + 琥珀辉光点睛，克制阴影（非 Material 投影）。

### 🌓 Light / Dark 双模式（跟随系统外观）

- 新增「暖纸」light 模式：暖米底 + 暖墨字 + 加深的琥珀墨，自动跟随 macOS 系统外观切换。
- 全 token 双模式化：改一处色值，两模式同时生效。

### 🗂️ 信息架构重构

- **审批中心提到一级导航**（原先埋在设置 4 层深），挂 pending badge 一眼可见。
- sidebar 分「全局 / 项目」两组。

### 📜 稳定流时间轴（首页重构）

- 首页从竖排列表重做为 **Time Machine 风格的稳定流**：会话按开始时间锚定、按天分组（永不重排，靠位置可找回）。
- 会话卡含**记录中**状态、**时长**、**并行**标；**commit 按归属嵌套进对话**（Context-Linked），未关联的归「其他提交」。
- 会话详情去掉左右分屏，对话占满；项目页新增悬浮「回到顶部」。

### 🧠 Blackboard 瘦头

- 「项目此刻」从占半屏的三卡块瘦成可折叠条，默认只展示「未落地想法」。

> 数据准确性原则：commit ↔ 会话关联只用 hook 精确检测，抓不到的如实留空，不做时间窗猜测。

---

## [1.0.2] — 2026-06-11

> Dashboard 模型分布跟随全局指标 + Token 口径统一（billed 摊分，标注不含缓存的实际产出）+ 分享卡片加入模型分布；**修复 Cursor 消息重复数据 bug（与 v1.0.1 的 Codex 同因，影响 Cursor 用户）**。

---

### 🔥 修复 Cursor 消息重复（数据完整性，影响所有 Cursor 用户）

与 v1.0.1 修的 Codex 同因：Cursor 此前用随机 UUID 当 message id，而 Cursor session 在源更新（`lastUpdatedAt` 变化）时会重新入库，随机 id 与已有不匹配 → 消息被反复插入累积。

修复：Cursor message id 改为确定性 `cursor-<composer>-<bubble_id>`（用源稳定的 bubble id），重新入库幂等去重。升级时通过 migration 自动清空旧 Cursor 消息 + 全量重建。Claude Code / Codex / OpenCode 不受影响。

### 🆕 Dashboard 模型分布跟随全局指标

模型分布现在跟随上方的指标切换器（对话 / 消息 / Token），与「AI 工具分布」一致；提交无模型维度，fallback 到消息。分享卡片也加入了「模型分布」板块。

### 🆕 模型分布 Token 口径统一 + 标注实际产出

模型分布的 Token 此前用 per-message token_count（不含 cache_read），与核心数字 / 工具分布的 billed（API 计费总量，含 cache_read）对不上。现统一为 **billed 按模型消息占比摊分**（与上方一致），并在下方标注 **「不含缓存 X」**（实际产出 = input + output，不含缓存重读）。能直观看出哪些模型缓存重读占比高（如某模型计费 22B、实际产出仅 1.7B，13× 是反复读缓存）。

---

## [1.0.1] — 2026-06-11

> Dashboard 模型分布跟随全局指标切换器；**修复 Codex 消息重复的严重数据 bug（实测约 5x 虚高）**——所有 Codex 用户的会话/消息/Token 统计都受影响，升级时自动清理回填。

---

### 🔥 修复 Codex 消息重复（数据完整性，影响所有 Codex 用户）

Codex 此前用随机 UUID 当 message id，而入库是增量 `INSERT OR IGNORE`。active session 的 jsonl 每增长就触发重新入库，随机 id 与已有不匹配 → 消息被当作新的反复插入 → **重复长期累积**。实测本机 Codex 消息 28507 条，去重后真实仅 4862 条（约 5x 虚高）。Dashboard 里 Codex 的消息/会话/Token 计数因此一直偏高（如 gpt-5.5 显示 7.6k，真实 1377）。

修复：Codex message id 改为确定性 `codex-<session>-<seq>`（文件名 + 顺序），重新入库幂等、去重生效。升级时通过 migration 自动清空旧 Codex 消息 + 重建 FTS + 干净重入库（顺带回填 per-message token）。Claude Code / OpenCode 等不受影响。

### 🆕 模型分布跟随全局指标

Dashboard「模型分布」现在跟随上方的指标切换器（对话 / 消息 / 提交 / Token），与「AI 工具分布」一致——切到哪个指标，每个模型就按哪个统计。提交无模型维度，fallback 到消息。

### 🆕 Codex per-message token 回填

Codex 的 `token_count` 事件是 session 累计快照，此前只用于 session 级、per-message 恒为 0。现按相邻快照 delta 归属到每回合 assistant 消息（口径对齐 Claude Code），per-message token 总和精确等于 session total，让模型分布的 Token 指标对 Codex 也准确。早期不发 token 事件的老 session 仍为 0（源数据所限，非 bug）。

---

## [1.0.0] — 2026-06-11

> **首个 1.0 里程碑**：新增「Beacon 审批中心」（待处理队列 + 历史 + 搜索 + 孤儿清理）、等待超时统一为全局配置，以及一轮深度审批安全加固（交叉 review B1-B7 + codex 6 轮独立验证，堵住约 48 个 hard-allow 绕过/边角变体 + IPC 数据丢失 + Rust 数据完整性）。25 个 commit since v0.9.4。

---

### 🆕 Beacon 审批中心

审批从「瞬时单卡」升级为「持久队列 + 历史」。设置 → 通知渠道 → Beacon → 审批中心：

- **待处理**：列出所有挂起的审批请求（错过卡片也能补处理），单个 / 批量 ✓✗。
- **孤儿清理**：来源会话（CC/Codex）已退出的请求自动标记「来源已退出」并可清理（CC 用 ppid 判活，Codex 用 age-based 兜底；判活失败保守保留，不误清活跃请求）。
- **历史**：回看全部审批历史，按命令 / 项目 / 决定（放行/拒绝）搜索过滤。

详见 `docs/wiki/approval-center.md`。

### 🆕 等待超时统一为全局配置

去掉 Beacon 硬编码的 30 秒超时。现在 Beacon 与飞书托管**共用同一个「等待超时」配置**（5min / 30min / 2h / 24h，默认 30min），选择器移到通知渠道公共区，飞书关闭时也能配。超时与托管开关解耦——单机 Beacon 模式也读用户配置，不再被 30s 强制 fallback。

### 🔒 安全加固（交叉 review + codex 6 轮独立验证）

一轮深度审批安全审查，每条线索对着真实代码核验后才修：

- **hard-allow 绕过（23 个实测确证）**：命令替换 / 重定向逃逸 / 引号绕过 / 编码绕过（ANSI-C `\x`·`\u`·`\U`）/ git 破坏命令（`branch -D`·`push --force`·`remote remove`）/ 路径逃逸 / 命令执行器（`command`·`env` builtin）等，逐项堵死。
- **codex 6 轮独立验证**：抓出约 25 个自审漏掉的边角变体，逐条核验后修 + 回归测试锁死（`b4_security_audit` 90+ 断言）。hard-allow 定位为「best-effort 减打扰白名单」，纵深靠 deny-list（拦最危险）+ LLM 灰区 + 用户审批。
- **IPC 数据丢失**：审批请求/响应非原子写 + 写失败仍删请求 → 改原子写（tmp+rename）+ 仅写成功才删；飞书状态机多处缺陷（OK 自触发、卡映射不清理、时间窗过滤、`_exec` 无 timeout）。
- **Rust 数据完整性**：`clean_user_text` 死循环（OOM 风险）、FTS 跨 session 重复插入（v0.5.7 膨胀复活）、git 同步事务化、hook 等待语义（单机超时、`beacon_alive` 加进程名校验防 PID 复用假阳性、请求文件原子写）。

### 🐛 其他

- **更新检查**：改拉 release 列表挑最高 semver（不依赖 GitHub `latest` 指针）；DMG 按硬件 arch 选包，修 Intel 用户下错包。
- **性能**：git 同步增量化 + 批量 scan 查询 + 每步计时埋点。
- **deny-list 误拦**：`<< 'EOF'`（`<<` 与 delim 间带空格）的 quoted heredoc body 未被剥离，正文含危险字面词时误拦合法命令（dogfood 修复）。

---

## [0.9.4] — 2026-06-09

> **auto-save 数据保护重大修复**（静默失败 2 个月的 bug）+ Beacon commit 数 / Risk / Dashboard / 项目分组多项修复。25 个 commit since v0.9.3（含大量 research/wiki 文档）。

---

### 🐛 auto-save 数据保护：修复静默失败 + 仓库膨胀（重点）

`~/.memora` 的自动 git 快照（数据保护）此前会因三层问题永久失效：

1. **孤儿锁** — git 操作被强杀（强退/崩溃）留下 `index.lock`，之后每次 auto-save 都因 "index.lock: File exists" 失败。实测一个锁来自 4 月 3 日 —— auto-save 已**静默坏了 2 个月**无人察觉。
2. **并发 race** — sync 周期重叠导致两个 auto-commit 并发跑 git，抢锁甚至**把 index 写坏**（`index file smaller than expected`）。
3. **仓库膨胀** — DB 备份文件（`memora.db.backup-*`）因 `.gitignore` 只挡 `*.db` 漏过滤，被吞进 git 历史，`.git` 撑到 6GB。

修复：① 孤儿锁自愈（存在 ≥30s 自动清理）；② 进程级互斥锁串行化 git 段，根除并发 race / index 损坏；③ `.gitignore` 放宽到 `memora.db*` 挡住所有备份变体。本机数据已清理：`.git` 6.2G→743M，`~/.memora` 25G→2.7G。

### 🐛 Beacon 卡片 commit 数封顶 999

commit 数之前用 `getGitCommits(limit:999).length` 计算，超过 999 的项目被 fetch 上限截断，永远显示 999。改用 `getGitCommitCount`（SQL `COUNT(*)`，与 fetch limit 解耦）取真实总数。

### 🐛 Risk 引擎：.env 模板误报

deny `.env` 但放过 `.env.example` / `.template` / `.sample` / `.dist`（这些是模板，不含密钥）。

### 🐛 Dashboard 多项修复

- 跨天 session 按消息时间戳分天，token 按比例 prorate。
- 柱状图按本地时区分天，与顶部卡片对齐。
- 「项目活跃排行」标题加时间窗，避免与侧边栏数字对不上。

### 🆕 项目分组增强

- project group 加 `display_path`，名字 + 路径可编辑，默认折叠公共目录前缀。
- 合并建议支持 checkbox 部分合并，accept 默认复用最活跃 group。

### 🆕 侧边栏 / 建议

- 临时路径 group 默认折叠到「显示临时项目 (N)」toggle。
- 合并建议：临时路径不进 + 同 remote dedup + 一键全部忽略。

### 🐛 其他

- 对话历史：没选中文字时右键不再显示「复制」按钮。

---

## [0.9.3] — 2026-05-24

> **Dashboard 4 个增强**（借鉴 Claude/Codex 监控工具）+ **Codex 模型解析修复**。2 个 commit since v0.9.2。

---

### 🆕 Dashboard 时间窗增加「今天」/「24 小时」

之前最短 7d，缺短周期洞察（"今天烧了多少 token"）。chip 顺序：今天 / 24小时 / 7天 / 30天 / 90天 / 全部。Rust `time_window_cutoff` 用 sentinel 编码：0=今天（本地时区 0:00 起）/ -2=24h（now - 24h）/ -1=All / N>0=N 天。

### 🆕 Dashboard 缓存命中率 progress bar

Tokens 卡内嵌横向 progress bar + 百分比。`cache_hit = (billed - total) / billed`，反映 prompt cache 效果（越高说明 prompt 设计稳定 + 缓存命中好）。`_StatNumberCard` 加 `progress + progressLabel` 字段。

### 🆕 Dashboard 模型分布 section（新）

按工具分组（Claude Code / Codex / OpenCode）列每个 model 的 message 数 + 占比 progress bar。能看到 CC opus-4-6 vs 4-7 切换历史 / Codex gpt-5.5 vs 5.4 使用比例。Rust 加 `ModelShare` struct + `dashboard_model_distribution` SQL。

### 🆕 Dashboard 柱图 hover tooltip

`_DailyBarChart` 改 StatefulWidget + MouseRegion。鼠标移到任意一天柱子，柱顶浮出 tooltip 显示日期 + 当天精确数值 + metric label。同时柱图把 hover 柱高亮（不透明 vs 默认 0.85 alpha）。

### 🐛 Codex ingester 修：5 万 assistant 消息全无 model 数据

Codex jsonl 在 `turn_context` line type 的 `payload.model` 携带模型字符串（`gpt-5.5` / `gpt-5.4` / `gpt-5.3-codex` 等），但 `codex.rs` ingester 之前 `model` 写死 `None` → dashboard 模型分布里 codex 全是 `(unknown)`。

修法：main loop payload 处理顶部统一扫 `model` 字段（cover 所有 line type），track session-level `current_model`，写 assistant message 时填入。`dashboard_model_distribution` SQL 加 `AND content NOT LIKE '[tools:%'` 过滤 `flush_tools` 合并的占位 message（model 恒为 None，污染统计）。

修后 codex 30d 窗 `(unknown)` 从 885 → 89（89 条是 jsonl 早期 `turn_context` 之前的真实 assistant，那时 Codex 还没记 model — 数据源限制不可恢复）。实操：cargo clean 强制 rebuild + DELETE codex sessions/messages 触发全量 re-ingest 219 个 jsonl → 1026 条 with-model 正确写入。

---

## [0.9.2] — 2026-05-22

> **小版本**：v0.9.1 dogfood 后 3 个 UI bug 修复。3 个 commit since v0.9.1。

---

### 🐛 Beacon 默认弹屏跟主显示器（不跟鼠标）

`focusScreen()` 之前 fallback 顺序：`preferredDisplayId → 鼠标 → main`。多屏环境下 Beacon 跟随鼠标飘忽。

改成 `preferredDisplayId → NSScreen.main → 鼠标 → 第一个屏`：默认跟主屏稳定位置；主屏被全屏 app/Game Mode 遮挡时才 fallback 鼠标所在屏；preferredDisplayId 显式设置过仍优先。

### 🐛 5 处 SelectableText → Text（修 copy 失效残留）

v0.9.1 修了 `_buildMessageRich` 的 `SelectableText.rich`，但漏了 5 处其它 `SelectableText`：dialog 里 3 处（hook install detail / 文件目标路径 / 错误 detail）+ pattern card commandSample + diff 视图。这些都跟外层 `SelectionArea` 冲突，copy 不生效。全部改 `Text` 让 SelectionArea 统一接管。

### 🐛 Session 消息右键加 Copy 菜单项

`onSecondaryTapUp` 在每条 message body 拦截右键弹"从此处继续"菜单，覆盖了 SelectionArea 默认 Copy。用户选中文字后右键只看到"从此处继续（保存为 Markdown）"，没法复制。

修法：自定义菜单顶部加 `[📋 复制]`（复制整条 `msg.content` + snackbar 反馈）；保留 `[💾 从此处继续]`。想 copy 选中部分用 **Cmd+C**（keyboard shortcut 不被 `onSecondaryTapUp` 拦截）。

---

## [0.9.1] — 2026-05-22

> **维护版本**：双 arch 包发行基建 + 3 类数据问题根因修复 + session 详情 copy 修复 + 版本号管理走业界标准。
> 9 个 commit since v0.9.0。无功能性新增，但发版基础设施大改造。

---

### 🆕 双 arch 包发行（aarch64 + x86_64 独立分发）

之前 v0.9.0 的 `_aarch64.dmg` 实际是 universal binary（命名误导）。现在两个真单 arch 包，体积对称 ~30MB。

- `./build.sh --arch x86_64` 单独 build Intel 包
- `./build.sh --both --dmg` 一次出双 arch 全套（自动 .app + DMG 双 notarize + staple）
- `lipo -thin` 把 Flutter 默认 universal 输出抽成单 arch（codesign 之前），避免 Apple Silicon 用户为永远用不到的 x86 slice 多下 35MB；Intel 用户在 macOS 26 上看到的"Support Ending for Intel-based Apps"弹窗只影响 Intel 包用户，不污染 Apple Silicon 用户

### 🔧 build.sh 全面硬化（16+ 类问题修复）

review-driven 重构。我自己 + codex 三轮 review 找到 16 类问题（commit 链 8d68e13 → 79f1908）：

- **R1-R6** 真问题：`--both --arch` 冲突拦截 / pubspec 每次 build 都 dirty / `cd $SCRIPT_DIR` 失败保护 / `dmg-bg.png` 缺失自动 fallback / pkill 后循环 wait 替代 sleep 1 / notarize wall-clock 日志
- **Y1-Y6** 设计/可靠性：arch 切换 clean 检测 / lipo thin fail-fast / self-invoke 用 `$SCRIPT_DIR/build.sh` 显式路径 / `--codesign` `--arch` 参数值校验 / `--both` 失败语义注释
- **G1-G6** 风格：APP_PATH 顶部一次定义 / git hash 加 `-dirty` 标记 / trap cleanup 临时文件 / hdiutil fallback volname 统一 / lipo loop 路径限定加速
- **codex P2-1**：`--both` 第一次 child build 改 pubspec 让 tree dirty → 第二次 child 算 -dirty 后缀让两个 DMG 版本不一致。修法加 `--frozen-version` 在 dispatcher 一次性算 + 锁定
- **codex P2-2**：`--both` 必须放 `$1` 才识别，`./build.sh --dmg --both` 报 Unknown option。修法 pre-scan 整个 args
- **Step 5.5 加 DMG notarize + staple**：之前 build.sh 只 notarize .app（Step 4.5），DMG 自身只 codesign 不 notarize。v0.9.0 是 release skill 手动补的；现在内置

### 🔧 版本号管理走业界标准（SemVer + Build Number 分离）

之前 `./build.sh` 默认 patch +1 + 写 pubspec.yaml → git status 永久 dirty + `git checkout pubspec` 会让版本"回退"。重构为：

- `pubspec.yaml` 只存 SemVer `version: a.b.c`，build 不修改
- Build number = `git rev-list --count HEAD`（commit 次数，跨机器一致）
- `flutter build macos --build-number=N` 传给 Flutter，写 Info.plist，**不写 pubspec**
- Info.plist：`CFBundleShortVersionString=a.b.c`（About 显示）/ `CFBundleVersion=N`
- `git_version.txt` 写完整 `a.b.c+buildN-githash[-dirty]` 给 Dashboard/About 用
- DMG 文件名只用 SemVer：`Memora_a.b.c_arch.dmg`
- Release 工作流：你手动改 pubspec a/b/c + commit + tag，build number 自动跟上

### 🐛 Dashboard 项目数 3 类数据问题（根因修复）

用户截图发现"21 个项目里有 0 member 孤儿 group / 空标题 group / 时间戳 +58213 年"。挖根因：

- **孤儿 group**：`ingest_session` 用 `INSERT OR IGNORE` 防 group_id 重复，但**只防 PK 不防 membership 已指向别处**。场景：合并 server → MerchantAI canonical 后，后续 server 目录有新 session ingest → group SQL 重建空 group。修法：SQL 加 `WHERE NOT EXISTS (SELECT 1 FROM project_membership WHERE project_id = ?1)`
- **空名 group**：OpenCode ingester 用 `directory.split('/').last()` 取 project_name，`directory="/"` 返回空串。修法：`rsplit('/').find(non-empty).unwrap_or("unknown")`
- **+58213 年时间戳**：OpenCode 6 个 session 的 `time_created`/`time_updated` 是 JS `Date.now()` 毫秒，但 ingester 用 `from_timestamp(value, 0)` 把 ms 当 sec → 时间放大 1000x → 2026 × 1000 = +58213 年。message timestamp 当时正确用了 `from_timestamp_millis`，session 级别忘了改。修法：session started/ended 也改 `from_timestamp_millis`
- 数据修复：DELETE 2 个孤儿 group / UPDATE 空名 group display_name='(root /)' / DELETE 6 个污染 session + 118 条 messages 让 Memora 重启后 re-ingest

### 🐛 Session 详情页 Copy 失效

选中消息文字 + 右键 Copy 后剪贴板空。根因：`SelectableText.rich` 跟外层 `SelectionArea` 冲突（Flutter 官方限制），SelectionArea 的 Copy 拿不到 SelectableText 的 selection 内容。修法：`Text.rich` 让 SelectionArea 统一接管。

### 📋 白板功能 MVP 设计文档（未实施）

接 MerchantAI 团队 brief（213 行 user story），落地 `docs/wiki/whiteboard-design.md`（362 行）。MVP 范围：覆盖个人 owner 启动 session 时 AI 主动汇报 todo + AI 跨 session 续 todo。不做：multi-user / git 同步 / PR webhook / 通知。估 8.5 天工时，待后续 sprint 启动。

---

### Changes

- **build**: 双 arch 包发行（`--arch` / `--both`）+ Step 5.5 DMG notarize 内置
- **build**: 版本号管理 SemVer + Build Number 分离，pubspec 不再被 build 修改
- **fix(db,ingest)**: dashboard 项目数 3 类数据问题根因修复
- **fix(session)**: SelectableText → Text.rich 修 copy 失效
- **docs**: 白板功能 MVP 设计文档落地

### 已知 limitations

- 5 处其它 SelectableText（dialog + diff 视图）尚未改 Text.rich，跟同样 SelectionArea 冲突有可能。低优先级，等用户报告
- v0.9.0 aarch64.dmg 实际是 universal 的历史遗留：v0.9.0 release 已重发为真单 arch（35MB），但仍标 v0.9.0 tag

---

## [0.9.0] — 2026-05-21

> **两大新功能 + 9 天 dogfood 大量收敛**。覆盖 v0.8.0 → v0.9.0 期间 444 个 commit。
> 新功能：**Project Grouping**（解决 monorepo 子目录 + 平行副本散落）+ **Dashboard 与分享卡片**（看见自己 AI Coding 状态 + 一键生成游戏存档式分享图传播）。
> 收敛：用户反馈机制（Phase A-E）+ Risk Engine shell-aware lexer 重写 + 11 类 false positive 修复 + Codex round-3~8 review 修法。

---

### 🆕 Project Grouping — canonical project + membership 两层模型

解决两类典型场景：MerchantAI monorepo 8+ 子目录被散到 N 个项目；LingoBuddy 4 平行实验副本（clone 自同 origin）也散开。session 数据零搬迁，合并完全可逆。详见 `docs/wiki/project-grouping-design.md`。

- **数据模型**：新加 `project_groups`（用户面向的"项目"）+ `project_membership`（cwd → group 一对一 + sub_label）+ `project_suggestions`（合并建议）三张表。原 `projects` 表保持不动作为 cwd 级 membership，sessions/commits 的 FK 引用零改动
- **migration v4**：每个现有 project 自动建同 id 的 auto group + membership，每次启动跑"愈合"逻辑补建漏掉的
- **Sidebar 右键菜单**：改名 / 合并到... / 拆分 / 设置子标签。合并 dialog 列出现有 group + "+ 创建新项目"选项
- **多 member group 详情聚合**：sessions / commits / 黑板 LLM context / 项目文件树状都按所有 member 聚合（之前单 cwd）。每行 session 显示来源 chip（sub_label 或项目名）
- **SuggestionEngine 自动推荐**：每次 sync 扫所有 cwd 的 `.git/config` 读 `remote.origin.url`（ssh/https/.git 后缀归一化），按 remote 分组生成 pending suggestion。规则 2: 同 root 同 remote (monorepo) / 规则 3: 不同 root 同 remote (clone 副本)。Banner UI 在 sidebar 顶部，三按钮：合并 / 详情 / 忽略。详情 dialog 底部 [关闭] [忽略] [合并] 闭环决策。不联网，纯本地 `.git/config`

### 🆕 Dashboard 总览面板 + 分享卡片

主 nav 左侧加 📊 Dashboard 入口（在 Agent 上方），点击进入 AI Coding 全景。设计文档：`docs/wiki/dashboard-design.md`。

- **5 个核心数字**：项目 / 对话 / 消息 / 提交 / Tokens（API 计费总量含 cache_read）。Tokens 卡 tooltip 显示工作量 token 对照
- **🔥 Streak**：连续 AI Coding 天数 + 历史最长。算法：每天有 session 或 commit +1，中断重置
- **时间序列柱图**：按当前 metric 单柱显示（4 个 metric 各自配色）
- **AI 工具分布**：CC / Codex / OpenCode / Antigravity stacked bar + 占比。按 message 数公平（非 session 数）
- **24h × 7day 活跃热力图**：反映工作节奏
- **Top 5 项目排行**：本地按当前 metric 重排序，详情行 4 维「N 对话 · M 消息 · X 提交 · Y tokens」全列
- **动作统计**：11 种 event_type（archive/star/continue/merge/split/rename/...）累计 + 近 7 天。`usage_events` 表为基础设施
- **全局 metric 切换 chip**：对话 / 消息 / 提交 / Token 一键切，热力图 / Top / 工具分布 / 柱图全部同步重新渲染。零重新加载（后端一次拉齐 4 维数据）
- **时间窗切换**：7d / 30d / 90d / All

**分享卡片**：Dashboard 右上 [Share] 按钮 → 弹左右分栏 dialog
- 左侧 8 个 section checkbox（核心数字 / Streak / 时间序列 / 工具分布 / 热力图 / 项目排行 / 动作统计 / 自定义文字）+ 5 个 Core sub-toggle（项目/对话/消息/提交/Tokens 独立勾选）+ 自定义文字 100 字限
- 右侧 800px 实时预览
- 强制项目匿名（dashboard 显示真名，分享时 A/B/C/D/E）
- 始终包含品牌 footer：Memora 忆码 logo + GitHub QR (`github.com/4over7/Memora`) + tagline「给 AI 编程加上存档点 / Checkpoint your AI coding」
- `RepaintBoundary.toImage(pixelRatio: 3.0)` 高清 PNG，`FilePicker.saveFile` 用户自选保存位置
- 全套 i18n 跟随 `S.locale`（中英）

### 🆕 用户反馈机制（治本 over 治标）

替代开发者每条 false positive 手 patch 的不可持续模式。Phase A-E 全实现。详见 `docs/wiki/user-feedback-loop-design.md`。

- **Beacon Allow forever / Deny forever 按钮**：弹窗多一行决策按钮 + 写 FeedbackStore
- **LLM-assisted PatternExtractor**：把命令样本 + Memora 判定 + 用户选择给 LLM，生成 trust pattern / deny rule 候选
- **RulesStore 统一数据层**：`~/.memora/rules.json` schema 含来源标记（builtin / user / feedback），启动 migration 老 `trusted-scripts.json`
- **Agent tab UI**（主 nav 入口）：pending feedback 列表 + 可编辑 pattern + [LLM 优化] 按钮 + [Confirm 写入 rules]；已生效规则列表 + 删除。**LLM 永远是 advisor，用户是 decider**（关键产品决策）
- 安全红线：内置 deny-list（`rm -rf` / `push --force` 等）永不能被用户反馈覆盖

### 🆕 Risk Engine 大量收敛（自 v0.8.0 起）

- **Shell-aware lexer**：从字符串扫描换成 AST，正确处理 quote / heredoc / 命令替换 / subshell / 重定向。修一批字面词 false positive
- **hard_allow.dart 单独抽出**：59 个单测覆盖 readonly 命令、git read/write、build/test、lark-cli read、SSH 只读探活、cmd substitution、Edit/Write 路径等
- **ApprovalOrchestrator 抽出**：从 UI state 解耦，独立单测
- **F11 v2 信任脚本命令模式**：pattern 含空格时走完整命令含 args glob 匹配。`~` 路径展开
- **C 修法多段命令**：长链命令每段独立评估，所有段都安全才 hard-allow
- **A 修法 deny-list grace timer**：deny-list 命中加 25s grace deny 显式写 response，防 hook timeout 静默放过（之前 timeout = 空 stdout = 放过，对 `rm -rf` 是漏洞）。也移除自动拒兜底改用 audit event
- **#23 SSH 远程 sudo+readonly+非 secret 路径 hard-allow**（B 方案治本）
- **#27 _stripShellRedirect 剥 subshell 括号** / **#28 lark-cli pipeline trust 多行 split + 安全 $(...) 剥**
- **#26 `/tmp` Write hard-allow** + **macOS TMPDIR `/var/folders` hard-allow**
- Codex round-3~8 review 多轮修：user deny glob anchor 到 cmd boundary / shell wrapper path 剥（含绝对路径 `/bin/bash`）/ bash `-c` 边界 / heredoc unquoted body 含 `$()` 标 unsafe / multi heredoc 保守 bail / npm/cargo publish 加命令头限定防字面词 FP
- **SQL deny rule** 加 SQL 客户端 context 限定 + **#17 grace deny 补 audit event**
- **trust pattern UI 校验放宽**：`~/` 开头、空格命令模式都支持
- **#19 isTrusted 识别 bash/sh/zsh wrapper 调用**

### Fixed

- **FTS5 search 连字符 keyword 报错**：搜 `sing-box-for-apple` 0 结果，根因 FTS5 把 `-` 当 NOT/column 操作符。phrase 包装每个 token 修
- **multi-member sessions 排序按 endedAt**：之前按 startedAt 让长 session（CC 早起开当晚仍在续）排到中间。跟 Rust SQL `ORDER BY COALESCE(ended_at, started_at) DESC` + UI 显示对齐
- **工具分布百分比 1491% bug**：`clamp(1, 1 << 30)` 把 total 上限截到 1.07B，16B tokens 除错。批量 `1 << 30` → `1 << 62`
- **分享卡片导出 ENTITLEMENT_REQUIRED_WRITE**：build.sh codesign 没传 `--entitlements`，加 `--entitlements macos/Runner/Release.entitlements` + plist 加 `user-selected.read-write`
- **Streak badge 居中** + **柱图渲染 width=0 修复**（Column crossAxis=start 不 stretch，显式 SizedBox(width) + CustomPaint(size)）
- **Beacon Allow forever security 反转 bug (Codex round-1 critical)**：`denyForever` 被 hook 当成 allow 放过，但 audit log 写 "deny"。normalize action 修
- **rules.json 写盘原子化**：之前崩溃后 trust list 静默丢失
- **`commandSample` redact**：feedback.jsonl + 发给 LLM 前过滤 secret（API key/token）
- **events.jsonl 补 feedbackLabel** + **setCachedForTesting 走 RulesStore** + **replaceTrustPatterns dedupe by id**
- **GitHub URL 修正**：之前编造的 `eastworld-ltd/memora` 改成真实 `4over7/Memora`

### Changed

- 项目维度 token 显示从 `input + cache_creation + output`（工作量）切到 `+ cache_read`（API 计费总量），跟 Dashboard 一致
- Sidebar 项目列表从 cwd basename 维度切到 canonical group 维度
- AI 工具分布从按 session 数算改成按 message 数（公平：5 message 与 500 message 的 session 不再等量）

### Internal

- Rust：`crates/memora-core/src/{db.rs,suggestion.rs}` 加 4 张表（project_groups / project_membership / project_suggestions / usage_events）+ 7 个 dashboard 聚合 SQL + streak 算法。`crates/memora-ingest` shell parser AST
- Dart：app.dart inline 加 dashboard / share card / project grouping 全套 widget（~2000 行）；l10n.dart i18n keys 翻倍
- 测试：cargo test 全过；flutter test **617 通过**（自 v0.8.0 加了几十个 dashboard / grouping / shell lexer 单测）
- 依赖新增：`qr_flutter ^4.1.0` + `path_provider ^2.1.4`（分享卡片 QR + 保存路径）
- frb codegen 多轮重跑同步

### Known limitations

- **dashboard 历史空白**：`usage_events` 表是 v0.9.0 才加的埋点，前 30 天图大部分是空（从今往后才有数据）
- **没 git remote 的本地 repo**（`git init` 未 push）SuggestionEngine 命中不了，走右键手动合并
- **跨机器同步未支持**
- Continue 向导多 member group 仍按 primaryProject 单 cwd 拉 branches（语义不清，留 follow-up）

## [0.8.0] — 2026-05-12

> **Codex 接入 + 一天 dogfood 后的大规模 risk engine 改进**。v0.7.0 飞书托管上线后一天集中 dogfood，发现一批 v0.4 留下的实现 bug + 安全设计漏洞 + UX 问题。**23 commits** 跨 Codex PreToolUse 完整链路、规则收紧（audit ~1700 LLM 调用浪费 + 安全 timeout 漏洞）、信任脚本机制 v2、LLM 性能调优。

### Added

#### 🆕 Codex PreToolUse hook 接入（Phase 1+2）

- **完整审批链路**：Memora 现在能拦截 Codex 的 Bash / apply_patch 工具调用，跟 CC 共享同一套 deny-list / hard-allow / Risk Engine / 飞书托管路径
- **schema 探测 Phase 1**：先用 dump 模式抓真实 Codex hook payload，确认 `tool_name` 直接是 "Bash" / "apply_patch"（跟 CC 一致）+ snake_case 字段名，几乎 1:1 复用 CC 处理代码
- **Phase 2 normalize**：apply_patch V4A 格式 parser，单文件 Add → Write、Update → Edit、Delete/多文件 → Bash fallback 走 LLM 灰色（防 Delete 静默放过）
- **响应协议适配**：Codex 严格限制（只接 `permissionDecision:deny`，不接 allow/ask），timeout/allow 一律空 stdout 等同放过
- **hook_installer 加 Codex PreToolUse entry**：timeout 86400s 跟 CC 对齐
- 详见 `docs/wiki/hooks-expansion-plan.md` Phase 3 详细设计章节

#### 🆕 信任脚本机制 (F11 v1+v2)

- **F11 v1**：用户自管 hard-allow 白名单 `~/.memora/trusted-scripts.json`，命令首段命中 → 跳过 LLM + Beacon 直接放过。支持精确路径或文件路径 glob (`*` 不跨 `/`、`**` 跨 `/`)
- **F11 v2 命令模式**：pattern 含空格时走"完整命令含 args"glob 匹配，例：
  - `lark-cli im +messages-send --chat-id oc_xxx*` — 限定特定 chat-id
  - `curl https://api.internal.example.com/*` — 限定特定 URL 前缀
  - `psql -h prod-db.internal` — 不含 `*` 退化为前缀匹配
  - basename 容错：pattern 写 `lark-cli ...` 也能匹配实际 `/opt/homebrew/bin/lark-cli ...`
- **C 修法多段判定**：长链命令（`cd && git status && lark-cli msgs-send`）每段独立评估，命中 `_readOnlyCmd` / `_gitReadCmd` / `_gitWriteCmd` / `_buildTestCmd` / `_larkReadCmd` / trust pattern 任一即"安全段"；**所有段**都安全才整条 hard-allow
- **Settings UI 管理**：Settings → 风险引擎 → 已信任的脚本，列表 + 添加（含空格触发命令模式）+ 删除
- **cache 即时同步**：每次 hook request 重新 load `~/.memora/trusted-scripts.json`，UI 改完立即生效不用重启

#### 🆕 飞书托管 hook 超时可配 (F10)

- 默认 86400s (24 小时)，从 v0.7.0 的 Phase 1 1800s 升级
- Settings → 通知渠道 → 飞书托管 → 加 "飞书等待超时" chip 行：5min / 30min / 2h / 24h 可选
- toggle 开启时默认写入 86400，跟代码 `kDefaultHookTimeoutSeconds` 同步

#### 🆕 LLM provider 升级

- **DeepSeek V4 Flash 上线**：default model `deepseek-chat` → `deepseek-v4-flash`，配 chip UI 让用户在 v4-flash / v4-pro 之间一键切换
- **DeepSeek V4 默认关 reasoning**（`thinking: {type: disabled}` 注入）：RiskEngine LLM 调用从 5-30s 降回 2-3s，本地审批 JSON 决策不需要 reasoning trace
- **Settings UI chip 候选 model**：Add/Edit dialog 在 model TextField 下方显示 chip，点 chip 直接填入；保留自由输入能力
- **静默 migration**：旧 `deepseek-chat` / `deepseek-reasoner` 自动升级到 `deepseek-v4-flash`

### Changed

#### 🛡️ Audit 后规则收紧（共 8+1 类 false positive 修）

基于 `docs/review/historical-audit-2026-05-11.md`（30 天 18418 条 risk-verdict 审计）发现 ~1700 条 LLM 调用浪费 + 边界 case 漏判：

- **D8 `--force-with-lease` 不再误伤 deny-list**：协作场景推荐的强制 push 形式跟 plain `--force` 区分，加 negative lookahead 排除（解锁 18+ 条合法协作 push）
- **D8.1 force-with-lease 走 LLM 灰色而非 hard-allow**：rewrite history 仍需用户明确批一次
- **D5 pnpm/yarn 白名单扩**：补 `build/test/dev/start/lint/typecheck/tsc/format/check/preview/jest/vitest/playwright/tsx/biome`，cd 前缀剥离支持多层链
- **D6+D7 `$(...)` bail-out 放宽**：先剥单引号 + quoted HEREDOC 后再判 bail（约 500 条 LLM 调用省掉）
- **D2 readonly 白名单扩**：xargs / awk / sed -n / python3 -m json.tool / PlistBuddy print / od / xxd / hexdump / strings / who / sar / free / ss / ifconfig 等
- **D3 SSH 只读探活 hard-allow (F14)**：`ssh [safe-flags] <host> "<readonly inner>"` 拆开判定，禁止隧道/转发 flag (-L/-R/-D/-W/-N/-T)，inner 全 readonly → hard-allow
- **D4 lark-cli 只读 sub 加白名单**：`im +chat-messages-list` / `messages-get` / `messages-mget` / `messages-reactions-list` / `chat-search` / `docs +fetch/search`
- **D9 deny-list `-m "..."` 不再误命中 commit message body**：扩 `stripShellLiterals` 剥 `-m`/`--message` 后的双引号字符串（不含 `$`/反引号），区分于 `-c "SQL"` flag（仍保留参与 match）
- **多段 git push hard-allow 漏判修**：`_isDefinitelySafe` 多段判定补 `_gitWriteCmd` / `_buildTestCmd`，跟单段判定对齐
- **F8 shell quoting strip**：deny-list regex 前剥单引号 / quoted HEREDOC body，修 `git commit -m "(truncate 200 字符)"` 误命中 `\bTRUNCATE\b` deny rule 的 false positive

#### 🛡️ Deny-list 安全门槛硬化 (A 修法)

- **deny-list 命中加 25s grace window 防 timeout 静默放过**：原非托管模式下 deny-list 命中只是 `rec=ask` 弹 Beacon，用户超时未点 → Codex 路径 hook timeout 空 stdout = Codex 放过执行（rm -rf 等危险命令）
- 修法：deny-list 命中时弹 Beacon **+** schedule 25s timer，过后 response 文件不存在 → Memora 写 deny response，hook 100ms 内 poll 到拒绝 → Codex 拦
- 用户**在 25s 内**点 Beacon Allow 仍可主动覆盖（保留人工 final say）

### Fixed

#### F5/F6/F7 follow-up

- **F5 hook reconcile**：install 改"清除老 memora 条目重写"，timeout 等字段升级会自动同步（旧"已安装就跳过"路径在 v0.7 把 PreToolUse timeout 60→86400 时不更新，老用户 settings.json 卡 60s）
- **F5 marker 严格判定**：从 substring `"memora-hook"` 改成命令首 token basename 严格等于，防 `echo memora-hook-failed` 类命令被误判清除
- **F6 飞书多卡片歧义保护**：用户在飞书回 `y/n` 时若有 >1 张 active 卡片，refuse 擅自匹配，发独立 P2P 提示 "请用 👍/👎 reaction 到具体卡片"
- **F7 Beacon SHOW 同步最新 risk state**：BeaconChannelAdapter 缓存 `_pendingRiskArgs`，BeaconService SHOW 时通过 `pendingRiskLookup` callback merge，避免 "AI 分析中" 卡 UI
- **F7 cache 泄漏修**：`_pendingRiskArgs` 4 个 terminal 路径全部 evict
- **F7 allow-list merge**：merge 用显式 13 字段白名单防 adapter 未来加字段灌进 Swift channel

#### F1-F4 follow-up

- **F1 hook reason 按场景区分**：CC 看到的 `permissionDecisionReason` 不再硬编码 `"Beacon: user denied"`，按 Memora/Beacon/飞书/deny-list 等填具体原因
- **F2 缓存命中 Beacon 不闪过**：托管模式 callback 走 Future race + 30ms timeout 防同步早返回前 SHOW 1ms 闪
- **F3 风险评估 radio 三态**：替代旧"风险引擎 + 省心模式"两 toggle，mental model 单一线性
- **F4 飞书自己响应卡片下加 reply 反馈**：用户回 y/n 后除了 🆗 表情还显示"✓ 已批准 / ✗ 已拒绝"thread reply

#### UX + Codex review 修

- **F9 `pnpm --filter <pkg> build` 类工作流不再走 LLM 等审批**：v0.7.0 dogfood 暴露的常见模式
- **F12 DeepSeek V4 慢导致 Beacon 卡 "分析中"**：关 reasoning 后审批延迟从 5-30s 回到 2-3s
- **Open in Ghostty 两个问题**：cd 不再被当 program；不再 split 所有已有窗口
- **P1.1 (codex review) CLI installer 同步**：`memora-hook install` 子命令 PreToolUse timeout 60→86400 + reconcile（之前 CLI 独立路径，团队用 CLI 装会卡 60s）
- **P1.2 (codex review) export JSON 默认剔除 credentials**：`exportJson()` 默认不含 `api_key`，加 `includeCredentials: true` opt-in
- **P2.5 (codex review) `should_skip_project` 改白名单**：原一刀切 `.xxx` 会误伤 `~/.config/nvim` 类 dotfiles 项目
- **README "Background sync every 15 seconds" 改事实**：实际事件驱动 + 60s 兜底

### Technical

- 新增 `lib/src/trusted_scripts_service.dart`：`TrustedScriptsConfig` + cache + `isTrusted`/`isSegmentTrusted` 两种 API
- 新增 `crates/memora-hook/src/main.rs::handle_codex_pre_tool_use` + `normalize_codex_tool` + `parse_apply_patch` (V4A format)
- `lib/src/risk/deny_list.dart::stripShellLiterals` 扩支持 `-m`/`--message` 后双引号 commit message body 剥离
- `lib/src/llm_service.dart` provider chip / DeepSeek thinking disabled / 静默 migration
- `crates/memora-core/src/hook_installer.rs::is_hook_up_to_date` 暴露（task #11 待 UI 接入）
- workspace.package: 0.2.0 → 0.2.3（CLI installer 同步触发 + hook hash 变化时 Codex 端需重 Trust）

### Tests

- flutter test: 320 passed / 15 skipped (Lark v2 stale tests 待 mock 重写 task #13)
- cargo test workspace: 全过（hook_installer 9 + Codex normalize 8 + deny_list 80 + trusted_scripts 22 等）
- flutter analyze: 0 issue

### Docs

- `docs/wiki/hooks-expansion-plan.md` 补 Phase 3 详细设计（Codex PreToolUse 接入完整 checklist）
- `docs/wiki/rule-editor-design.md`（新）—— v0.8.x 量级可视化规则编辑器设计
- `docs/review/historical-audit-2026-05-11.md`（新）—— 18418 条 risk-verdict 30 天审计报告

### Follow-up (留 task)

- #10 Memora Hooks UI 加 Codex Trust 激活提示文案（每次 hook hash 变需重 Trust）
- #11 Hooks 版本检测改用 `is_hook_up_to_date()` 对比 hooks.json schema
- #13 Lark 测试重写 mock 架构（解 5 个 v2 stale skip）
- #14 app.dart 6873 行拆审批 orchestrator（v0.8.x 重构）
- #15 ~/.memora 自动 git 加 Settings toggle + 文案
- #16 review_tests/ stale 处理

---

## [0.7.0] — 2026-05-11

> **飞书托管模式上线**。开关一开，AI 工具的工具调用审批走飞书私信：90% AI 自动批静默放过，10% 真需要决策的双通道弹（Beacon + 飞书私信任一响应即放行）。出差/离开电脑也能完整跑 AI 工作流，hook 等响应可长达 30 分钟（Phase 1 保守值，Phase 2 拉到 24 小时）。**39 commits** 含一堆 v0.4 留下的实现 bug 修复。详细设计：`docs/wiki/approval-modes-design.md` v2。

### Added

- **🔒 飞书托管模式**（核心新功能）：
  - Settings → 通知渠道 → 飞书托管开关，开启后所有 PreToolUse 审批走飞书 P2P 私信
  - **AI 优先决策**：deny-list 命中直接拒（沉默）；Risk Engine 判 low/medium 自动批（飞书完全静默）；只有 LLM 判 high/ask 才发卡等用户
  - **Beacon 立即 / Lark deferred 双通道**：用户在电脑前 Beacon 0 延迟立即弹（v2 设计原则）；离开电脑 5-10s 后 LLM 跑完才决定要不要也发飞书。任一通道响应即放行（first-response-wins）
  - **飞书审批 UX**：用户在飞书回 `y/n`（或 👍/👎 reaction）→ Memora 给 reply 加 🆗 表情回执 + 在卡片下发 `✓ 已批准/✗ 已拒绝` thread reply
  - 卡片初始就带 AI 建议（不再先"分析中..."再 reply）—— 减少飞书消息数 3:1
  - Hook 端 timeout 1800s（Phase 1 保守）/ 86400s（Phase 2 准备）
- **风险评估模式 radio 三态**（替代原"风险引擎+省心模式"两 toggle）：不启用 / AI 辅助（默认）/ AI 自动批，mental model 单一线性
- **Lark channel toggle = 飞书托管 toggle**（合并）：UI 单一开关同时控制 channel 启用 + 托管 config
- **Settings UI 整理**：Beacon tab 合并到通知渠道；"项目角色" → "AI 编程设置"
- **Blackboard 三栏支持折叠**（默认显示前 8 条，超出"展开剩余 N 条"），避免 LLM 输出 90+ 条撑爆页面
- **commit 时间线分页加载**：项目页 chip 显示真实 total（而非 fetch limit 50），折叠区底部"显示更多 50 条"按钮按需 fetch
- **LarkChannel 失败诊断**：`lastError` 字段 + `~/.memora/logs/lark-*.log` 日志，UI 显示真实错误（不再笼统"未安装"）

### Changed

- **审批模式三选一**（淘汰 B 模式"开飞书但不开托管"，用户群微到忽略不计）：模式 A（Beacon-only）/ 模式 C（飞书托管），中间态合并到 C
- **dispatch 非托管模式不参与 Lark fan-out**：避免飞书刷屏。Lark 只在托管模式 LLM 判 high/ask 时单独发卡
- **`_isDefinitelySafe` shell 解析剥离 redirect**：`ls 2>&1 | head` 等 readonly 命令不再因 `&` 被切分误判为灰色地带
- **省心模式 rec=ask 不再自动批**：LLM 明确要求人工判时不该被悄悄放（之前漏判）
- **commit chip 用真实总数**：从 db `SELECT COUNT(*)` 取，不再用 fetch list 长度
- **黑板增量判断同步**：使用 `_gitCommitTotal` 而不是被 limit 的 `_gitCommits.length`
- **Codex token 解析**（之前一直 0）：Codex jsonl 里有 `event_msg.token_count` 事件，按 OpenAI 协议语义映射到 Memora 内部 4 类字段
- **Archived 对话折叠**：默认主列表只显示未存档，末尾加"📦 N 个已存档"折叠节
- **导出对话默认文件名**用易读时间格式
- **Lark 卡片 reply 留言**：dismiss / updateWithRisk 用 `messages-reply` 文本（v2 schema 废弃了 `messages-patch`）
- **hook reason 按 source 区分**：CC 看到的 `permissionDecisionReason` 不再硬编码 `"Beacon: user denied"`，按 Memora / Beacon / 飞书 / deny-list / 各自模式填具体来源

### Fixed

- **Beacon 67% 通知 1ms 闪过**：`onRequestParsed` async 但 `_scanRequestFiles` 没 await，BeaconService 抢先 SHOW 然后被 hard-allow 路径 dismiss。修 await 让 callback 决定完再加 queue
- **双 Memora 实例 race**：build.sh `osascript quit "Memora"` 只杀 LSDatabase 注册的 /Applications 实例，build 目录直接启动的不管。`pkill -x Memora` 也大小写错了（实际进程名 `memora` 小写）。修 build.sh 加 `pkill -x memora`
- **缓存命中 Beacon 仍然 67ms 闪过**：托管模式 callback 走 race + 30ms timeout 防同步早返回前 SHOW
- **飞书 reply "y/n" 完全无效**：LarkChannel.pollOnce v0.4 只拉 reactions 不拉 text reply，托管模式实际不可用。加 `+chat-messages-list --user-id` 拉 P2P 私信
- **Blackboard / Drift / Spark JSON 截断**：Anthropic `max_tokens` 写死 2048 不够长 JSON 输出。加 `maxTokens` 可选参数，长任务显式传 8192
- **Lark cli macOS .app 找不到**：PATH 不全 + lark-cli shebang `env node` 找不到 node。改用绝对路径 + 给子进程显式增强 PATH
- **Lark cli messages-send `--receive-id` 已废**：改 `--user-id`（私信）/ `--chat-id`（群聊）
- **飞书卡片 V2 schema `tag:action` 已废**：改用 `div + lark_md` 纯文本提示行
- **`kDefaultLarkReceiver` 硬编码 Leon open_id**：改为空 + 空 receiver 守卫（之前任何人开 Lark 默认就发到 Leon 飞书）
- **Lark channel toggle 点了不响应**：v0.4 `onChanged` 漏调 `LarkChannel.setEnabled(v)`
- **托管模式自己响应卡片下没 reply 反馈**：用户回 y/n 后只有 🆗 表情，缺"已批准/已拒绝"。加 LarkChannel.handleInboundEvent 主动发反馈 reply

### Technical

- `lib/src/managed_mode_service.dart` — 托管 config IPC（`~/.memora/managed-mode.json`）
- `lib/src/app.dart::onRequestParsed` — 按 managedMode 分模式 A/C 时序
- `lib/src/channels/lark_channel.dart` — 全面重构：路径解析 + 环境注入 + lastError + reply 拉取 + 表情回执 + 自己响应 reply 反馈
- `lib/src/channels/channel_registry.dart::dispatch(channels:)` + `attachLateChannel()` — 支持选择性 fan-out + 后期 attach
- `crates/memora-hook/src/main.rs` — 读托管 config 动态 timeout（30s/1800s/24h）+ response reason 透传
- `crates/memora-core/src/hook_installer.rs` — CC settings.json `PreToolUse.timeout` 60s → 86400s
- 测试：`flutter test` 282 passed / 15 skipped（v2 stale 标 skip，待重写 mock 架构）+ `cargo test --workspace` 全通过 + `flutter analyze` 0 issue

### Docs

- `docs/wiki/approval-modes-design.md`（v2 设计 + 4 follow-up）
- `docs/wiki/lark-managed-mode.md`（Phase 1 实施细节）
- `docs/wiki/hooks-expansion-plan.md`（CC 13 hook 调研 + 6 个产品方向）
- `docs/wiki/lingobuddy-needs-evaluation.md`（外部需求评估，吸收 F1/F5）

---

## [0.6.0] — 2026-05-06

> 项目页 token 统计从单数字升级成"总数 + 4 类细分悬停"，配套数据层把 4 类 token 分别落库；导出对话时自动复制路径到剪贴板，文件名也从毫秒时间戳换成易读格式。

### Added

- **项目页顶部加 token 统计 chip + Tooltip 4 类细分**：之前 chip 行只有"对话/消息/提交"三项，token 数据其实在 session 卡片显示了（"2.5M tokens"），但项目级没汇总。**改动**：chip 行新增紫色 token chip 显示全项目 total（用 K/M/B 格式化），鼠标悬停展开 4 类——`新输入` / `写缓存` / `输出`（这三项加起来等于 total）+ `缓存命中`（标注"不计入总数：同一份上下文反复读取"，避免再次困惑 v0.5.8 修过的 cache_read 累加问题）。
- **导出对话保存后自动复制文件路径到剪贴板**：`_exportConversationUpTo` 写完文件直接 `Clipboard.setData(file.path)`，SnackBar 文案改成"已保存到 X · 路径已复制到剪贴板"。常见后续动作是把路径贴回 AI 对话里继续讨论历史，省一次手动复制。
- **导出对话默认文件名换成易读时间格式**：`MerchantAI_recall_1778059262327.md` → `MerchantAI_recall_2026-05-06_18-21-02.md`。无 `:`（macOS Finder 显示 `:` 实际是 `/`）、文件系统/工具友好、字母排序 = 时间排序。

### Changed

- **数据层：session token 拆 4 列存储**：`mof.rs::SessionStats` 新增 `input_tokens / cache_creation_tokens / cache_read_tokens / output_tokens` 4 个字段，`total_tokens` 派生为 `input + cache_creation + output`（仍保持 cache_read 不进 total 的口径）。`db.rs::sessions` 表 ALTER 加 4 列；`pragma user_version=2` 触发一次性全量 reingest，让旧 session 行回填 4 类细分（跟 v0.5.8 同样套路）。`list_projects` SQL 用 `SUM(...)` 把 4 列项目级聚合一并返回。
- **Rust API 暴露 4 类细分给 Flutter**：`ProjectInfo` / `SessionInfo` 各加 4 个字段，`flutter_rust_bridge` 重新生成。

### Technical

- `crates/memora-core/src/mof.rs:56-75`：`SessionStats` 4 字段扩展 + 字段语义注释
- `crates/memora-ingest/src/claude_code.rs:53-60`：`RawUsage::cache_read_input_tokens` 转为正式字段（带语义注释）；ingest 主循环 4 类分别累计
- `crates/memora-core/src/db.rs:117-149`：4 列 ALTER + `user_version=2` 迁移
- `rust/src/api/simple.rs:328-394`：`ProjectInfo` / `SessionInfo` API 字段扩展
- `lib/src/app.dart:5689-5749`：`_TokenChip` widget（chip + Tooltip 渲染）
- `lib/src/app.dart:3180-3194`：导出对话路径复制 + 时间格式
- 测试：`flutter test` 297/297 + `cargo test --workspace` 全通过 + `flutter analyze` 0 issue

---

## [0.5.8] — 2026-05-02

> 修两个潜伏 bug：① 长 session token 数 O(N²) 虚高（一次 215.7M 的 session 真实只有几 M）；② 大项目（10K+ 消息）点 Blackboard 60s 撞 timeout。再附两个小修：session→session 切换时返回按钮失效，Beacon 焦点切到非编程 app 不收回。

### Fixed

- **长 session token 数 O(N²) 严重虚高**：`crates/memora-ingest/src/claude_code.rs::RawUsage::total_input` 把 `cache_read_input_tokens` 也累加到 session.total_tokens。但 cache_read 是"这次 LLM 推理读了多少缓存"——一个对话里前面所有消息都会进 prompt cache，第 N 条 assistant 推理时把整段历史从缓存读出来一次。跨整 session 累加 cache_read 等于把同一份内容数 N 次（O(N²)）。实测 410 条消息的 session 跑出 215.7M tokens 就是这样来的。**修法**：`total_input = input_tokens + cache_creation_input_tokens`，不再加 cache_read。配套 `crates/memora-core/src/db.rs` 加一次性数据迁移（`pragma user_version=1` 控制只跑一次）：所有 sessions.raw_size 设 -1，下次 sync 触发 `scan_raw_directory` 全量 reingest，session.token_count 用新公式覆盖。
- **大项目 Blackboard 撞 60s 超时**：BiggerLadder（50 commits / 14 sessions / 10269 msgs）点 Blackboard 报 `TimeoutException after 0:01:00: Future not completed`。`lib/src/llm_service.dart::_callOpenAI` 写死 60s。**修法**：`complete()` 加可选 `timeout` 参数，Anthropic 默认 30s→60s、OpenAI 60s→120s；Blackboard / Drift / Spark 长任务显式传 180s。新增 `_postWithRetry`：`TimeoutException` / `SocketException` / `ClientException` 自动重试 1 次（隔 2s），每次打 `[LLM] tag attempt=N in=B out=B took=ms status=N` 日志。失败时 UI 不再清空黑板——保留上次结果，顶部红条提示"刷新失败，显示历史结果"+ 重试按钮，没有历史结果才弹错误框。
- **session → session 切换后返回按钮失效**：`_pickSession` 的 setState 无条件把 `_view` 写到 `_previousView`。当用户在 session view 里再点另一个 session（搜索/相关跳转），`_previousView` 被覆盖成 `_View.session`，点返回时 `_view = _previousView = _View.session` 等于无变化，看起来"按钮失效"。**修法**：只有从非 session view 进入时才更新 `_previousView`。
- **Beacon 焦点切到非编程 app 不收回**：`macos/Runner/BeaconPanel.swift::checkAndNotifyWindow` 的守卫同时拦 `.notification` 和 `!isMouseOver`，但鼠标 hover 是悬停态、不是激活态，焦点切走是更强信号。**修法**：守卫只保留 `.notification`（审批弹出不能被打扰），focus 切到非编程 app 立即 collapse。同时给 `activeAppChanged` / `checkWindow` 加 diag log，方便后续排查。

### Added

- **LLM 调用本地日志文件 `~/.memora/logs/llm-YYYY-MM-DD.log`**：release build 下 `debugPrint` 被 macOS unified log 吞掉，看不到 `[LLM]` 行。`_postWithRetry` 现在同步写文件，按日切分自然滚动，失败 silently catch。一行一条记录，方便后续 grep / awk 统计真实耗时与 input/output 大小关系。
- 设计文档：
  - `docs/wiki/blackboard-timeout-fix.md`：本次 Blackboard 修复方案 + 验证路径
  - `docs/wiki/session-summary-multilayer.md`：多层 session 摘要设计草案，⏸ 待立项（前提待验证）

### Technical

- `crates/memora-ingest/src/claude_code.rs:60-72`：`RawUsage::total_input()` 公式收紧
- `crates/memora-core/src/db.rs:117-138`：`pragma user_version` 数据迁移机制
- `lib/src/llm_service.dart:223-380`：`complete(timeout)` + `_postWithRetry` + `_appendLlmLog`
- `lib/src/spark_service.dart`：Blackboard / Drift / Spark / SessionSummary 路径显式声明 timeout
- `lib/src/app.dart`：`_blackboardRefreshError` 状态 + UI 红条 + `_pickSession` previousView 守卫
- 测试：`flutter test` 297/297 + `cargo test --workspace` 1/1 + `flutter analyze` 0 issue

---

## [0.5.7] — 2026-04-26

> 修一个潜伏所有版本的 db corruption 根因：FTS5 全文索引 segment 失控膨胀导致 db 涨到 6 GB，最终磁盘满 → SQLite "database disk image is malformed"。**所有用户都建议升级**（你的 db 可能也在膨胀）。

### Critical Fixed

- **DB FTS 索引失控膨胀（db 涨到 6 GB → 30 MB）**：`crates/memora-core/src/db.rs::ingest_session` 之前是"全量替换"模式——每次 raw 文件 size 变化（FSEvents 每次 jsonl append 都触发），就 DELETE 该 session 全部 messages + INSERT 全部。FTS5 contentless 模式下 DELETE 写 tombstone segment、INSERT 写新 segment，auto-merge 跟不上 churn → segments 累积到 94 万（messages 才 4.5 万行，21 倍异常）→ fts_data 涨到 3.6 GB → db 涨到 6 GB → 磁盘满 → wal 写 ENOSPC → 半写 frame → "database disk image is malformed" → 所有 ingest 失败 → UI 数据冻结。
- **修法**：`ingest_session` 改增量——先 SELECT existing message id 集合，只 INSERT 不在集合里的（CC append-only + parse_session 已 dedup → 安全）。effect: sync 一个长了 3 条的 jsonl 只产生 3 个 FTS segment（之前是 500 个）。
- **升级路径**：自更新拿到新版本后，老用户的 db 还是膨胀状态——可以手动跑一次 hot fix 缩 db：
  ```bash
  pkill -f "Memora.app/Contents/MacOS/memora"
  sqlite3 ~/.memora/memora.db "INSERT INTO messages_fts(messages_fts) VALUES('rebuild'); VACUUM;"
  open /Applications/Memora.app
  ```
  本机实测 6.35 GB → 34 MB（缩 99.5%），耗时 ~2 秒，数据零丢失。

### Added

- 完整调研 + 修复设计持久化在 `docs/wiki/db-fts-bloat-fix.md`，含 3 层根因证据链 + 边界论证 + rollback 方案。

### Cumulative changes since v0.5.0（中间 v0.5.1～v0.5.6 dogfood，未单独 release）

- **v0.5.6**：windowID ↔ session 映射（私有 AX API `_AXUIElementGetWindow`），同 ghostty 多窗口现在能精准识别"焦点哪个窗口标哪个 session"
- **v0.5.5**：摘要 cross-source 综合（注入 git branch + 关联 commits + prompt 硬约束保留版本号/hash/数字）；同 app 多窗口切换时 Beacon 重新计算（Swift fromAX 不 dedupe + Dart 去掉 project.id 去重）
- **v0.5.3**：iTerm2 进程链 BFS（4 层穿透 iTerm2 → iTermServer → login → zsh）；自更新 helper 加 `xattr -cr` 自动清 quarantine
- **v0.5.2**：build.sh 在打 DMG 前先 notarize + staple `.app`（之前只 staple DMG，从 DMG 拖出来的 .app 没自带 ticket，离线/VPN 异常时无法首次启动）
- **v0.5.1**：终端类窗口（Ghostty/iTerm/Terminal/Warp/...）用 PID → 子进程 cwd 反查项目，覆盖 Codex CLI / 普通 cd 场景

### Technical

- `crates/memora-core/src/db.rs:254-310`：`ingest_session` 重写为增量
- 测试：`cargo test -p memora-core -p memora-ingest` 10/10 + `flutter test` 297/297 + `flutter analyze` 0 issue

---

## [0.5.6] — 2026-04-25

> 同 ghostty 多窗口"焦点哪个就标哪个 session"——用私有 AX API 拿 windowID 解决了 v0.5.5 的"BFS 区分不了同 app 多窗口"问题。

### Fixed

- **同 ghostty 同项目多窗口 [当前] 标记不跟焦点切换**：v0.5.5 已经让窗口切换触发 `_sendExpandedData` 重入,但 `_detectCurrentSessionIdxByPid` 自身从前台 ghostty 主 PID BFS 找子孙——同 ghostty 多窗口共享主 PID,BFS 集合完全相同,永远返回第一个命中的 session。根因调研发现:macOS 公开 API 完全没有"GUI 窗口 → PTY"的映射,WindowServer 和 PTY 子系统是分开的(IOKit / kernctl / sysctl 都没有,ghostty 自己也没暴露 IPC 查询接口)。
- **修法**：用私有 API `_AXUIElementGetWindow` 拿 focused window 的 CGWindowID(hammerspoon / yabai / AeroSpace 等长期使用,稳定),Swift `windowChanged` 事件附带 `windowId` 字段。Dart 维护 `_windowSessionMap`，hook 触发那刻把"焦点窗口 windowID ↔ 这次 hook 的 sessionId"绑定。`_detectCurrentSessionIdxByPid` 优先级 1 = 查 windowID 映射,fallback 才是 BFS。
- **用户体验**：在新窗口里第一次跑任何会触发 hook 的工具(哪怕 Read 一个文件),windowID ↔ session 关联立刻建立,之后切回该窗口 [当前] 标记瞬间正确。映射 30 分钟未刷新自动失效。

### Technical

- `macos/Runner/BeaconPanel.swift`：`@_silgen_name("_AXUIElementGetWindow")` 私有 API 声明,新增 `getFocusedWindowID(for: pid)`。
- `lib/src/beacon_service.dart`：新增 `_lastForegroundWindowId` + `_windowSessionMap` 字段。`_scanRequestFiles` 在记 `_sessionByPid` 旁边同步写 `_windowSessionMap[当前 windowId] = sessionId`。`_detectCurrentSessionIdxByPid` 加一层 windowID 直接查表的优先级。

---

## [0.5.5] — 2026-04-24

> Session 摘要 cross-source 综合（注入 commits + git branch）+ 同 app 多窗口切换 Beacon 不刷新。

### Added

- **Session 摘要 cross-source 综合**：之前 LLM 只看对话内容，生成的摘要常常是抽一条结论而非"会话回顾"。本版给摘要 LLM 同时注入：
  - 当前 git branch（来自 SessionInfo）
  - **该 session 关联的 git commits**（v0.6 起 sessionId↔commit 已关联，按时间逆序最多 40 条，含 hash + message + 行数 stats）
  - 对话本身
- **prompt 硬约束**：必须保留版本号 / commit hash 前 7 位 / 文件名 / PR 编号 / 测试数等量化锚点；commits landed 段非空时摘要必须出现至少一个具体 commit 描述（不是泛泛"修了几个 bug"）；末尾如对话有暗示下一步则用 `下一步: xxx` / `Next: xxx` 单独一行
- **拼接策略改进**：user 消息全保留（800 字以上才头 350 + 尾 350），assistant 消息 600 字以上才压缩成 head 250 + tail 250，总长 12KB 上限
- 字数预算 80-150 → 100-180 字（避免被字数压扁）

### Fixed

- **同 ghostty 同项目两个窗口切换 Beacon 不刷新**：两层 bug：(1) Swift `checkAndNotifyWindow` 只看 `windowTitle == lastWindowTitle` 去重，同标题不发事件给 Dart；(2) Dart `_onWindowChanged` 看 `project.id == _lastMatchedProjectId` 直接 return 不重算 [当前] session。修法：Swift 区分事件源——AXObserver 来的事件**不去重**（focus 切换是真信号，必发），poll 来的事件保留 title 去重（避免 1.5s spam）；Dart 去掉 project.id 去重，让 `_sendExpandedData` 重入（v0.4.19 race guard 已保证可重入安全；sessions/branch/git/summary 走缓存，唯一新成本 `_detectCurrentSessionIdxByPid` ~50ms）。

### Technical

- `lib/src/spark_service.dart`：`generateSessionSummary` 新签名加 `sessionCommits` + `gitBranch`；`_buildTranscript` 重写拼接策略；中英 prompt 重写。
- `lib/src/app.dart`：`_generateSessionSummaries` 用 `_gitCommits.sessionId` 一次构建 `commitsBySession` map，每个 session 调用时传入对应 commits + branch。
- `macos/Runner/BeaconPanel.swift`：`checkAndNotifyWindow` 加 `fromAX` 参数；AX 触发不 dedupe，poll 触发 dedupe。
- `lib/src/beacon_service.dart`：去掉 `project.id == _lastMatchedProjectId` 去重 return。

---

## [0.5.3] — 2026-04-22

> 修 v0.5.1 的 PID→cwd 反查在 iTerm2 上完全不工作 + 自更新 helper 主动清 quarantine。

### Fixed

- **iTerm2 用户切到 Codex 终端时 Beacon 仍然不展开**：v0.5.1 的 `_resolveTerminalProjectPath` 只用 `pgrep -P <terminal_pid>` 找直接子进程，但 iTerm2 的进程链是 `iTerm2 → iTermServer-X.Y.Z → /usr/bin/login → zsh`（4 层深），shell 不是 iTerm2 的直接子。还有用户多 tab 多 codex 场景，每个 codex 都属于不同 zsh tab。修法：改为**一次性 ps -A 拉全系统 process tree，BFS 递归找到所有 shell 后代 PID（深度上限 6）**，每个 shell 拿 cwd，匹配 Memora 项目集合。覆盖 iTerm2 / Ghostty / Apple Terminal / Warp 等所有终端的进程模型。
- **自更新 helper 主动清 quarantine**：飞书 / AirDrop / 浏览器 等渠道下载的 DMG 会给文件打 `com.apple.quarantine` xattr，自更新走 helper script 启动（不走 GUI 双击授权流程），macOS TCC 会拦"未经用户主动授权 launch"，表现为"无法打开"。helper 在 `cp -R` 后加 `xattr -cr` 清掉 quarantine，避免用户被迫手动跑命令。

### Technical

- `lib/src/beacon_service.dart`：`_resolveTerminalProjectPath` 重写为 BFS 递归 + 全系统 ps tree。
- `lib/src/update_service.dart`：helper script 模板 cp 后加 `xattr -cr "$installDir/$appName"`。

---

## [0.5.2] — 2026-04-22

> 修一个潜伏所有版本的 release 流程坑：拖到 /Applications 后离线/网络异常时无法首次启动。

### Fixed

- **离开 DMG 的 .app 没自带公证票据 → 第一次启动卡"无法打开"**：之前 `build.sh` 只对 DMG 跑 `xcrun stapler staple`，没单独 staple `.app`。从 DMG 拖到 `/Applications/` 的 `.app` 因此**没有 ticket**，macOS 必须联网去 Apple 验证公证状态。用户在 VPN 异常 / DNS 故障 / 离线场景下会卡"无法打开"。修法：`build.sh --dmg` 流程在打 DMG 之前先 `notarize + staple .app`，DMG 内的 `.app` 自带票据，离开 DMG 也能离线启动。**v0.5.0 / v0.5.1 都有这个坑**——通过 GitHub Release 走 web 下载的用户因为联网没问题，所以没人反馈；但飞书私发 DMG / 局域网传输等场景就触发。

---

## [0.5.1] — 2026-04-22

> Codex CLI 用户终于能看到 Beacon 展开态项目详情了。

### Fixed

- **Codex CLI / 普通 cd 终端窗口不展开 Beacon**: v0.5.0 焦点优先 fix 之后，Beacon 项目识别只剩"窗口标题解析"一条路。CC 主动写 `Claude Code — /path` 标题所以能识别；Codex CLI 不写，普通 cd 也不写。这两类用户切到终端窗口时 Beacon 一直停在 collapsed。新增**终端类 app 优先用 PID → 子进程 cwd 反查项目**：
  - `pgrep -P <terminal_pid>` 拿前台终端的子 shell PIDs（多 tab 各一个）
  - 每个 PID `lsof -d cwd` 拿当前工作目录
  - longest-prefix 匹配已知 Memora 项目
  - 唯一命中 → 切；多 tab 多项目命中 → 不切（无法判断焦点 tab，避免乱切）；都不命中 → fallback 到窗口标题
  - 按 terminalPid 缓存解析结果 10 秒，避免每次窗口聚焦都跑 shell
- **覆盖范围**：Ghostty / iTerm / Apple Terminal / Warp / Alacritty / Kitty 全部受益。IDE 类（Cursor / VSCode / Antigravity）继续走窗口标题（IDE 主动写项目名更准）。

### Technical

- `lib/src/beacon_service.dart`：新增 `_resolveTerminalProjectPath(terminalPid)` + `_cwdOfPid(pid)`。`_onWindowChanged` 终端 app 分支优先走新路径。

---

## [0.5.0] — 2026-04-22

> 自动更新上线 + Beacon 焦点窗口优先 + 黑板带 session 回顾摘要 + 代码 0 warning。从此版本起,用户不需要再手动下载 DMG。

### Added

- **自动更新（Auto-Update）**: 启动时静默查 GitHub Releases API,有新版本立刻**后台下载** DMG 到 `systemTemp`,下载完主界面顶部出现紫色 banner "Memora vX.Y.Z 已下载 · 重启完成更新"。用户点 "立即重启" 后,主进程写一段 helper bash(hdiutil attach → cp .app → detach → relaunch)然后 `exit(0)`,helper 静默完成替换并拉起新版本。Settings → About 区也有 "检查更新" 入口,可看进度/重试。**不做自动重启**——用户可能正在调试,强行重启会丢上下文。`Distribution` 部分砍掉(Memora 只有 GitHub 公开发布,不需要 speakout 那套 Gateway 降级路径)。
- **Session 回顾摘要**: 项目详情页"对话·过程"卡片新增一段 LLM 生成的会话摘要,跟着 UI 语言。摘要 prompt 不写 commit message 风格的"做了什么",而是覆盖三个维度:**起因**(用户为什么发起这次对话)、**过程**(讨论了什么、走了哪些方向、遇到什么阻碍)、**阶段性进展**(完成 / 决定 / 待办 / 还在讨论 / 卡住)。目的是让用户日后翻 session 列表时能瞄一眼判断"这个会话跟我想找的东西相关吗"。
  - **触发**: 用户点"黑板"按钮时顺带触发,黑板按钮上显示进度 "总结对话 12/33"
  - **缓存**: `~/.memora/session-summaries/<session_id>.json`,用 `messageCount` 作 fingerprint,无对话更新就跳过 LLM,再次点黑板不重生
  - **并发**: 3 路并行,避免 30+ session 串行太慢
  - **无 LLM**: 静默不显示该行,不留空白

### Fixed

- **Beacon 后台 hook 抢走焦点窗口的项目展示**: 用户在 Memora 窗口编辑代码时,speakout 后台的 claude session 触发 Hard Allow hook,Beacon 突然切到 speakout。根因 `app.dart:262` Hard Allow 路径主动调 `notifyActiveProject(req.cwd)`,任意项目的偶发 hook 都会盖掉用户当前焦点窗口的项目。修法:`notifyActiveProject` 简化成只记录 `_lastHookProjectPath`(供 window title 为空时回退),不再写 `_lastMatchedProjectId`、不再触发 expanded 推送。窗口聚焦成为 Beacon 项目上下文的**唯一**来源。详见 `docs/wiki/beacon-focus-vs-hook.md`。
- **审批弹出后 dismiss 不恢复用户原始意图**: 之前 `dismissNotification` 写死"前台 app 是编程类 → expanded,否则 → collapsed",违背"用户在审批弹出前手动收缩了 → 审批结束应该回到 collapsed"的常识。新加 `stateBeforeNotification` 快照:进入 `.notification` 状态时记下基础形态(`collapsed`/`expanded`),`dismissNotification` 优先恢复快照值,只有冷启动直接弹审批的情况才走旧的"前台 app 类型"回退。

### Technical

- **代码质量 0 warning**: 之前 `flutter analyze` 20 个 issue + Swift 编译 1 个 unused var,本版全部清完。删 5 个 unused import、2 个 unused field(含 3 处死写入)、rename `_log`/`_logFile`、`(_, __)` 改 Dart 3 pattern `(_, _)`、helpers_test 5 处 string concat 改 `[...].join()`、Swift `let statsH` 整行删。`flutter test` 297 用例全部通过。
- **新增**: `lib/src/update_service.dart`(精简自 speakout,~230 行)、`docs/wiki/{auto-update,beacon-focus-vs-hook,session-summary}.md`、l10n 11 个新字符串。
- **修改**: `lib/src/app.dart`(顶部 banner + 启动 checkForUpdate + Settings 更新行 + session summary lazy generate / cache / render)、`lib/src/beacon_service.dart`(`notifyActiveProject` 简化)、`lib/src/spark_service.dart`(`generateSessionSummary` + 中英 prompt)、`macos/Runner/BeaconPanel.swift`(`stateBeforeNotification` 快照与恢复)。

---

## [0.4.19] — 2026-04-21

> Beacon 展开态多 session 聚合 + 项目级 agent 配置对比。双核迭代打磨。

### Added

- **Beacon 展开态多 session 聚合**:之前只读 `sessions.first` 的 transcript 尾部,输出单 session 描述式摘要,忽略了 Memora 相对 CC recap 的独占价值——多 session 并行视角。现在最多聚合 top 3 活跃 session,prompt 按 session 数量自适应:
  - 单 session:1-2 行状态 + 可选"待你决定:..."
  - 多 session:每行一个 session(`Session N: <在干啥>`),标 `(当前)` 区分用户焦点,最后一行可选待决事项
  - 输入侧 transcript 预过滤 12KB → 1800 字/session,剔除 `file-history-snapshot` / `tool_use` / `tool_result` / `queue-operation` / `system` 噪声元数据(之前 LLM 会看着 file-history-snapshot JSON 乱凑出"AI 完成一轮对话含 428 token"这种废话)
- **精准识别"用户当前在哪个 session"(三层信号)**:
  - hook 写 `ppid` (= `claude` 进程 PID) 到 beacon-request JSON
  - Dart 端记 `_sessionByPid`,从前台终端 PID `pgrep -P` 递归找 CC 后代,命中 map 即判定
  - `pgrep -x claude` + `lsof -p <pid> -a -d cwd` 拿到项目里活着的 claude 集合,作为"本项目最多显示几个 session"的 ground truth(不依赖 hook 是否跑过)
  - 失败 fallback 到 jsonl mtime
- **Beacon 展开态动态高度**:summary 之前硬 48pt × 3 行 = 90 字上限,超过就吞尾巴。新静态 `BeaconContentView.measureSummaryHeight` 用 `NSString.boundingRect` 实测文本高度,clamp 到 3-6 行(51-102pt)。140/180 字 prompt 结果能完整显示。
- **项目级 agent 配置对比**:项目详情页 L2 区出现 2+ agent 配置(CLAUDE.md / AGENTS.md / AGENTS.override.md / .cursorrules / .claude/CLAUDE.md / .codex/AGENTS.md)时,section 标题右侧出现"对比 & 同步 (N)"按钮。对话框两列 LCS 行级 diff,一侧独有的行红色高亮,列头文件名可点击用 `open` 拉起默认编辑器自行修改。

### Fixed

- **Beacon summary LLM 看不到真对话乱凑总结**:`file-history-snapshot` 之类的元数据记录动辄 5KB,把 2000 字 tail 窗口塞满,真正对话被挤出,LLM 只好瞎编。修法详见上述 tail 预过滤。
- **beacon 展开态异步 race**:`_sendExpandedData` / `_refreshSummaryAsync` 多个 await(DB / LLM / 文件),用户快速切窗口时旧请求可能在新请求已匹配之后完成,把 Swift 面板 push 成旧 project 数据。`_pushExpandedData` 前加 `_lastMatchedProjectId != project.id` 的 guard,过期数据丢弃。
- **hot session 筛选把僵尸拉进来**:之前 `endedAt == null` 判活跃,但 CC 很少写 session-end → 一大堆老 session 符合。改为三层滤:jsonl mtime < 2h + `_sessionByPid` 已知死的剔除 + 按"项目里活着的 claude 数"封顶。
- **LLM 输出字面 `\n` 被当文本**:prompt 里原本用 `'\\n'`(Dart 源 -> 运行时两字符 `\n`),LLM 偶尔照搬。改为真换行符 + 兜底 `replaceAll(r'\\[nN]', '\n')`。
- **Beacon summary 字数上限**:80 字太紧写不下第 2 行"待你决定",调到 140/180(对齐 tweet 心智模型)。

### Technical

- `beacon_service.dart`:新 `_sessionByPid` map / `_lastForegroundPid` / `_detectCurrentSessionIdxByPid` / `_aliveClaudePidsInProject` / `_transcriptMtime` / `_pidForSession`。
- `helpers.dart`:新 `extractConversationText(rawTail, maxChars)` pure-Dart 过滤器 + 5 个单元测试。
- `memora-hook/main.rs`:beacon-request JSON 加 `ppid = libc::getppid()` 字段。
- `BeaconPanel.swift`:`BeaconContentView.measureSummaryHeight` + `transitionTo(.expanded)` 动态算高。
- `app.dart`:`_AgentConfigDiffButton` + `_showAgentConfigDiffDialog` + `_lineDiff` LCS 实现。

---

## [0.4.4] — 2026-04-21

> 修 CC v2.1.114+ queued prompt 在 Memora 里不显示的问题(用户发送的消息被当 transient 丢弃)。

### Fixed

- **CC v2.1.114+ 的 queued prompt 在 Memora UI 里缺失** —— CC 更新后,用户在工具跑完前输入的 prompt 会被 queue。transcript 里新出两种记录:`type:"queue-operation"`(enqueue 瞬间 transient)和 `type:"attachment"` + `attachment.type:"queued_command"`(CC 真正处理时写入,含 `prompt`)。Memora ingest 只 match `user`/`assistant`,两种都被丢 → UI 里看不到用户这条 prompt,只见 AI "凭空回答"。修法:`claude_code.rs` 加 `RawAttachment` struct + 单独分支接 `queued_command`;`queue-operation` 保持丢弃(attachment 已覆盖,避免重复入库)。2 个新单元测试。
- **历史 session 回填** —— 扫 `~/.memora/raw/claude-code/` 找到 25 个含 queued_command 的 session,把它们在 DB 里的 `raw_size` 置 0 强制下次 sync 重新 parse。升级后自动生效,无需用户操作。

### Technical

- `claude_code.rs`:`RawAttachment { att_type, prompt }` struct + line_type "attachment" 分支。
- 单元测试:`test_parse_queued_command_attachment` + `test_non_queued_command_attachment_ignored`。

---

## [0.4.3] — 2026-04-20

> 修 `git -C <path>` 绕过 deny-list 的安全漏洞 + 对话历史图片可点击查看。

### Fixed

- **`git -C /path ...` 绕过 deny-list(安全问题)**:CC 常用 `git -C /abs/path <subcmd>` 在指定目录跑 git,但 deny-list 的 7 条 git 规则全都是 `^git\s+push\s+...` 形式,遇到 `git -C /victim push --force` 正则不匹配,直接放行进入 LLM 层 —— LLM 有概率判 allow,是真实 bypass。hard-allow 的 `_gitReadCmd` / `_gitWriteCmd` 同样漏判,导致 `git -C /path log ...` 白白等 7-8 秒 LLM。修法:`helpers.dart` 加 `kGitCmdPrefix` 正则片段(覆盖 `-C <dir>`、`-c <k=v>`、`--git-dir[=|空格]<path>`、`--work-tree[=|空格]<path>`,可重复可缺省),app.dart 两条 hard-allow 正则 + deny_list.dart 全部 7 条 git 规则一起加前缀。5 个 prefix-bypass 回归测试。

### Added

- **对话历史图片引用可点击查看**:消息正文里的 `[Image: source: /abs/path.png]`(CC 附件元数据)现在渲染成紫色可点击"📎 图片"链接,点击用系统 `open` 拉起默认看图 app。`[Image #N]` 无路径占位符渲染成灰色标签。文件不在时 SnackBar 提示"图片已不在本地"。最小实现:不复制、不持久化;`parseMessageSegments` pure-Dart 拆段 + `SelectableText.rich` + `TapGestureRecognizer`;9 个边界测试。

### Technical

- `helpers.dart`:`kGitCmdPrefix` public const + `parseMessageSegments` / `MessageSegment`。
- `app.dart::_isDefinitelySafe`:用 `kGitCmdPrefix` 拼接 `_gitReadCmd` / `_gitWriteCmd`。
- `app.dart::_buildMessageBody` + `_openImagePath`:图片段 render + 点击 open。
- `deny_list.dart`:全部 7 条 git 规则前缀扩展。
- 回归测试:`deny_list_test.dart` +5,`helpers_test.dart` +9。

---

## [0.4.2] — 2026-04-20 (skipped — dogfood only, not released)

Superseded by 0.4.3. 0.4.2 dogfood build contained only the image display feature without the git-prefix security fix.

---

## [0.4.1] — 2026-04-20

> 省心模式体验打磨:常见 grep 不再走 LLM,auto-allow 不再留面板残影,展开态面板可随手收起。

### Fixed

- **Hard Allow 漏判**:`_isDefinitelySafe` 的命令分段正则按 `|;&` 硬切,不懂 shell 引号,导致 `grep "A\|B\|C" file` 被切成多段失配、降级走 LLM。新加 shell-aware 预处理:遇到 `$(...)` / 反引号直接 bail-out;否则先剥掉 `"..."` / `'...'` 内容再 split。常见带 regex alternation 的 grep 重新走第 2 层瞬时放过,不弹通知、不等 LLM。
- **Auto-allow 后面板残影 1-2 秒**:`BeaconChannelAdapter.dismiss` 在 self-display 模式下是 noop,`_autoAllowRequest` 只写响应文件 + 删请求文件,面板得等 BeaconService 下一次 scan 周期(~2s)才被 `_checkNotificationResolved` 清掉。`BeaconService.dismissById(reqId)` 新公开方法,auto-allow 直接同步清 active id + 推进队列 + 发 native dismiss。Lark 先响应的路径同步调用,本地面板也瞬时收。
- **展开态面板挡住编程窗口内容**:点面板空白区域之前不响应(展开态 click-through 被 noop 掉);改为点非按钮区域即收缩到小圆点,方便临时看被遮住的代码;窗口切换时仍自动重新展开。"Open in Memora" 保留在按钮上。

### Technical

- `app.dart::_isDefinitelySafe`:shell-aware split。
- `beacon_service.dart`:`dismissById` public 方法,兼顾 active / queued 两种状态。
- `BeaconPanel.swift::mouseDown` + `handleClick`:expanded/full 空白区域点击 = 收缩,不再激活 Memora 主窗口。

---

## [0.4.0] — 2026-04-18

> v0.4 把 Memora 从"被动通知层"演进为"**有判断力的代理层**"。三件事一锅做:Channel 抽象 + Risk Engine + 新 Approval UI——它们共享同一个数据流(`ApprovalRequest`),分开做是浪费。完整设计见 `docs/v0.4-design.md`(~850 行)。

---

### ✅ 本版已完成

#### Added — 主线

- **Channel 抽象**:通知不再绑定 Beacon。`ChannelRegistry` 统一调度,每个请求 fan-out 到所有启用的 channel,first-response-wins,其他 channel 自动 dismiss。
- **Risk Engine 三层决策架构**(基于 29035 条真实样本评估,完整设计见 `docs/auto-allow-design.md`):
  - **Hard Deny**(规则,永远问):`rm -rf`、系统路径、凭据覆盖、`git push --force` / `reset --hard` / `rebase` / `filter-branch`、`sudo + 写/删除`、`curl|bash`、SQL `DROP/TRUNCATE/DELETE no WHERE`、`npm/cargo publish`、`.ssh/.aws/.env`、`.github/workflows`、`~/.memora/raw/` 和 `~/.claude/` 对话记录等。
  - **Hard Allow**(规则+上下文,一定放过):项目内 Edit/Write(git 可恢复)、只读命令(grep/cat/ls/git log/SELECT)、构建测试(`flutter test`、`cargo build`、`npm run`)、git 日常(commit/push 非 force)、项目内 `rm`。
  - **灰色地带**:LLM advisor(DeepSeek / Claude / OpenAI)出 verdict + 动词/意图/关键参数/理由,LLM 离线/超时走静态 fallback,再不行走 raw fallback。
- **新 Approval UI**:替代 raw shell command 显示。`🔍 搜索源代码 + AI 建议接受 + 关键参数高亮`。Spinner during analysis → done with semantic verb → fallback states when LLM times out (15s) or unparseable.
- **EventLogger**:append-only `~/.memora/events.jsonl` 记录每个 PreToolUse、verdict、user decision,供 transcript-aware prompt + 审计。
- **两种操作模式**(2026-04-18 设计修订):
  - **正常模式(默认)**:deny-list + Hard Allow 之外的所有操作都弹通知,LLM verdict 作为参考。
  - **省心模式(opt-in + 风险提示)**:LLM 判 `level != high && rec != deny` 全部 auto-approve。
  - 决策纯函数 `lib/src/risk/decision.dart`,11 个边界 case 单元测试。
- **自动刷新**:`Directory.watch` 监听 `~/.memora/raw/` + 60s 兜底轮询。UI sessions 自动更新,不用点刷新按钮。

#### Fixed

- **Beacon 项目误匹配**(v0.3.0 起潜伏 11 天,2026-04-18 surfaced):Ghostty OSC 7 上报用户键入大小写(`/Users/leon/Apps/memora` 小写 `m`),`_matchProject` 之前是 case-sensitive + first-match,误加的父目录项目(`/Users/leon/Apps`)吞掉所有大小写不一致的子项目。修复:case-insensitive + longest-prefix。
- **AX 权限被 macOS 静默撤销**(影响所有 v0.3.0+ 用户):`flutter build macos && cp -R` 留下 adhoc 签名 → macOS 撤销 accessibility → `AXDocument` 返回 nil → Beacon 检测静默失效。`CLAUDE.md` 写明禁止跳过 `./build.sh`。
- **Hook 协议脱节(F1, P0)**:CC hook 从不传 `toolInput`,所有路径都从 `requestDetail` 回退解析。Cache key 扩展。
- **路径含空格(R1)**:正则 `(/\S+)` 被截断,三处同步改为 `(.+?)(?:\s+\(|$)`。
- **递归删除规则漏拦(R2)**:`rm -fr build/` 反向 flag + `/tmp/` 场景补全。
- **Beacon 渲染**:expanded 项目切换不刷新(transitionTo early-return);Hard Allow 时不显示 expanded;统计行错位(绝对→相对定位);文字叠加(dynamicContainer 整体替换);hook projectPath 回退。
- **Risk Engine post-validation**:只读命令(cat/grep/git log)被 LLM 误判 high 时 demote 回 `read/low/allow`。
- **Edit/Write auto-allow 路径提取**:rawArgs.file_path 缺失时从 requestDetail 提取,项目内编辑按设计 auto-allow。
- **Hard Allow notification dismiss 时序**:dispatch 移到三层判断之后,auto-allow 请求不再闪 toast。
- **Compact 消息丢失**:压缩前 transcript 片段现在保留。
- **搜索 / commit 跳转精度**:`ListView` → `scrollable_positioned_list`,精确定位目标行。
- **LLM timeout 15s**:防止 provider 慢卡死整个 pipeline。

#### Changed

- **Deny-list policy 大幅收紧**(2026-04-11,基于 13005 条真实样本,`docs/risk-evaluation-report.md`):deny list 是"硬编码最后一道安全门槛",不是综合风险分类器。以下移到 LLM defer:
  - `npm/pip/cargo/brew install`(只保留 `publish`)
  - `sudo apt update` 等 read-ish sudo(只保留 `sudo + rm/chmod/dd/tee/systemctl restart`)
  - `curl -X POST` / `curl --data` / `gh api POST`(只保留 `curl|bash`)
  - `psql production-db` keyword 启发式
  - `cp memora.db`(只保留 SQL 写操作)
  - **结果**:deny FP -86%,static fallback 准确率 +24pp。

#### Removed

- **`auto_allow_low_risk` SharedPreferences key 不再读取**,替代为 `unattended_mode`(默认 `false`)。升级影响:旧默认 `true` 的用户升级后进入正常模式,体感更"啰嗦"。要恢复"自动放过"请开省心模式。无 schema migration 是有意为之。
- **LLM `confidence >= 0.8` auto-allow 阈值**:prompt rule 6 已用 confidence 兜底成 `rec=ask`,双保险过保守。
- **`.stationary` collectionBehavior**(4ace347 引入 → 097a7f8 撤掉):基于错误判断加的。详见"已知问题"段。

#### Quality assurance

本版经历 **4 轮独立第三方评审**(`docs/review/` 16 篇报告)+ 完整修复响应链:

| 状态 | 项目 |
|---|---|
| 已关闭(4) | F1 Hook 协议脱节(P0) / F5 Beacon token 混淆 / R1 空格路径 / R2 递归删除漏拦 |
| 保持开放(4) | F4 `app.dart` 过大 / F6 README 已过时 / F7 隐藏目录项目被静默跳过 / F8 API Key 明文 SharedPreferences |
| 转产品决策(3) | F2 安全策略变化需同步测试/文档 / F3 MVP 仅 CC 审批需明示 / F9 `~/.memora` 自动 Git 缺可见控制 |

关闭基线 `08462d9` + `5ebe9c5`,总表见 `docs/review/final-status-matrix-2026-04-16.md`。

---

### 🚧 Beta / 未实机验证

这些代码已就位,**但本版没有完整端到端 dogfood**,当作 beta 使用,遇问题欢迎反馈:

- **LarkChannel(飞书 / Lark)**:Phase D 代码完整(交互卡片 + reactions 轮询 + Settings UI),计划在 v0.5 window 内完成端到端验证。Settings → Channels 可以配置试用。

---

### ⚠️ 已知问题(本版未修)

- **F8 / 安全**:LLM provider API Key 保存在 SharedPreferences 明文,导出 JSON 带出。计划迁 macOS Keychain(见"Coming next")。
- **F4 / 架构**:`lib/src/app.dart` ~5000 行,审批编排跨文件维护成本高。
- **F6 / 文档**:README "15 秒后台同步" 与当前真实行为(FSEvents + 60s 兜底)不一致。
- **F7 / 可用性**:`should_skip_project()` 会静默跳过一些真实 dotfile/dot-config 项目(如 `.dotfiles/`)。
- **F9 / 产品**:`~/.memora` 的自动 Git 是有意设计,但用户不可见、不可控(无状态展示、空间占用、开关)。
- **Mission Control 中 Beacon 仍可见(有意不做)**:完整调研 `docs/wiki/beacon-mission-control-hide.md`。`com.apple.expose.*` distributed notification 不可靠(假阳性多 + 抢焦点);`.transient` collectionBehavior 在全屏 app 中跟丢;`CGWindowList` 轮询 99.99% 时间空转工程上不合算。等可靠方案再做。

---

### 🔮 Coming next(v0.5+ 预告,未启动)

按 `docs/v0.4-design.md` 的"非目标"段,这些是**本版明确不做,下一版开始考虑**的:

- **LarkChannel 端到端 dogfood + 稳定化**:本版为 beta,v0.5 做验证。
- **多 channel 同时启用 + fallback 链**:Beacon 30s 无响应 → 推 Lark → 再无响应 → Telegram。
- **Per-project / per-tool risk policy 配置 UI**:允许用户为具体项目或工具定制 deny-list / Hard Allow。
- **Beacon 重构为 BeaconChannel**:v0.4 保留 Beacon 原样以降低 refactor 风险,v0.5 统一抽象。
- **迁 API Key 到 macOS Keychain**(F8):安全债偿还。
- **`lib/src/app.dart` 拆分**(F4):独立重构周期。
- **README / 设置页 / 官网文案同步**(F6 / F3):明确当前 MVP 仅 CC 审批。
- **IM Inbound(手机给 Claude Code 下命令)**:v0.6 之后的远期目标。

---

#### Technical

- 新增设计文档:`docs/v0.4-design.md`(主设计 ~850 行)、`docs/auto-allow-design.md`(三层决策)、`docs/risk-classification-review.md`(方法论)、`docs/risk-evaluation-report.md`(13005 样本)、`docs/wiki/beacon-mission-control-hide.md`(MC 调研)。
- 新增模块:`lib/src/channels/`(5 文件)、`lib/src/risk/{deny_list, static_fallback, prompt, risk_engine, event_logger, decision}.dart`。
- 测试:`test/decision_test.dart`(11 边界 case)、`test/deny_list_test.dart`(70 case 对齐当前 policy)、`test/v04_integration_test.dart`(E1-E4 端到端)。全套 **278 测试通过**。

## [0.3.1] — 2026-04-08

### Critical fix
- **Hook adhoc signing → Developer ID** (`build.sh`): v0.3.0 shipped with cargo-default adhoc-signed `memora-hook`, which macOS 14+ Gatekeeper SIGKILLs at exec time (exit 137). All PreToolUse / PostToolUse hooks silently failed on user machines after install — Memora captured zero hook events. `build.sh` now re-signs the binary with Developer ID Application after `cargo build` and before `flutter build` (rust_lib `include_bytes!` freezes hook bytes at compile time, so signing must happen before the framework is sealed). The hook's CDHash propagates through `include_bytes!` into the .app bundle, then through Apple notarization — Gatekeeper recognizes the same CDHash when the hook later runs from `~/.memora/bin/`. **Anyone running v0.3.0 should upgrade to v0.3.1 to restore hook event capture.**

### Added
- **Hook version check in Settings**: Settings → Hooks displays the version of the installed `memora-hook` binary alongside the expected version. Mismatch shows an amber warning banner with reinstall instructions. New `--version` flag on `memora-hook` enables this.
- **Configurable Beacon notification sound**: 14 system sounds (or None) selectable in Settings → General. Each option has a preview button.
- **Settings UI redesign**: Two-pane layout (760×560) with macOS-style left sidebar — General / LLM / Personas / Hooks / About. Replaces the previous single-column scrolling layout.

### Fixed
- **Beacon expanded panel text overlap**: Stale `NSTextField` subviews accumulated across `updateState` calls, leaving previous project info visible underneath new content. Now uses a `dynamicContainer` pattern that wipes all dynamic subviews on each transition.
- **Beacon click intent**: Clicking anywhere on the expanded panel previously navigated to the main app. Now only the explicit "Open ↗" hit rect triggers navigation; the rest of the panel is non-interactive.
- **Display dropdown stale on monitor hot-plug**: Settings → Beacon → Preferred Display still showed the old list after connecting/disconnecting an external monitor. Swift now sends `beacon:displaysChanged` to Dart on `screenParametersChanged`, and `PopupMenuButton.onOpened` re-fetches before render.
- **Beacon sentinel liveness**: PreToolUse used a stale sentinel PID without `kill(0)` check, occasionally allowing requests after the matching Beacon process had crashed.
- **Hook protocol — empty stdout ≠ allow**: Hook now always writes explicit `permissionDecision` JSON, never relying on Claude Code's silent default.
- **Stale LLM test**: Groq `effectiveModel` test asserted the old `llama-3.3-70b-versatile` default after `3a64fd2` updated it to `meta-llama/llama-4-scout-17b-16e-instruct`.

### Technical
- New Rust API `hook_version_status()` in `crates/memora-core/src/hook_installer.rs`, exposed to Dart via flutter_rust_bridge as `check_hook_version()`.
- `build.sh --no-bump` flag added so release builds can be rebuilt against an exact pubspec version without auto-incrementing patch.
- `NSScreen.didChangeScreenParametersNotification` listener invokes Swift→Dart method channel callback (`beacon:displaysChanged`) so Flutter can refresh display state without polling.

## [0.3.0] — 2026-04-07

### Added — Beacon V3
- **File-based authorization IPC**: Hook polls `~/.memora/beacon-responses/<toolUseId>.json` for up to 30s, then falls back to terminal prompt. Replaces fragile keystroke simulation entirely — works across macOS Spaces, multi-window terminals, and any focus state.
- **AXDocument-based project detection**: Reads `kAXDocumentAttribute` (OSC 7 cwd) directly from focused window. No reliance on window titles. Works for Ghostty/Terminal/iTerm2/Warp out of the box.
- **Preferred display setting**: User can pin Beacon (project info + notifications) to a specific monitor. Critical for multi-monitor setups (e.g., vertical secondary screen) where notifications would otherwise be missed.
- **Per-request resolved tracking**: PostToolUse writes `toolUseId` (not timestamp) → precise notification dismissal even with rapid queue.
- **Notification queue**: Multiple PreToolUse requests now display sequentially instead of being lost.

### Fixed
- **UI not updating on rapid project switches**: `transitionTo` early-returned when state was unchanged, leaving stale data on screen. Now `beacon:show` always re-renders contentView.
- **Notification panel clipping on non-notch displays**: Panel height was `210 + notchH` but content used `max(notchH, 32)` as top padding — buttons clipped on external monitors. Fixed to use consistent topArea formula.
- **Hook timeout leaves stale UI**: Hook now deletes request file on timeout so Dart auto-dismisses the orphaned notification.
- **Stale response file accumulation**: Dart cleans `~/.memora/beacon-responses/` orphans (older than 30s) on startup.
- **Wrong code signing on `flutter build macos`**: Documented that `build.sh` must be used for proper Developer ID signing — ad-hoc signed builds lose accessibility permissions.

### Removed
- All keystroke simulation code (`sendKeystrokeToTerminal`, `findAndRaiseTerminalWindow`, `raiseWindowContaining`, `sendKeyCodeToPid`, `keystrokeForAction`) — replaced by file-based IPC. ~100 lines of fragile AX/CGEvent code gone.

### Technical
- v3 entity model: `terminalPid → window[]` is one-to-many (unstable), `sessionId → projectId` is many-to-one (stable, cacheable). Documented in `docs/beacon-entity-model.md`.
- `kAXDocumentAttribute` returns `file:///` URL → parsed to absolute path → Dart path-prefix matching identifies project.
- `NSScreen.displayID` extension via `deviceDescription[NSScreenNumber]` for stable display identification across reconnects.
- `NSApplication.didChangeScreenParametersNotification` listener auto-handles display hot-plug.

## [0.2.22] — 2026-04-06

### Added — Beacon (Dynamic Island for AI Coding)
- **Beacon floating panel**: macOS Dynamic Island-style status indicator that lives above the menu bar
- **Authorization dual-channel**: Hook returns "ask" (non-blocking), Beacon and terminal both show prompts simultaneously
- **Keystroke forwarding**: Beacon Allow/Deny sends keystrokes to target terminal via `CGEvent.postToPid()` — no focus switch
- **Configurable permission rules**: `_allowAllRules` in Dart determines 2 vs 3 button layout per source+tool (Edit/Write → 3, Bash → 2)
- **Pixel crab mascot**: Claude Code-style block crab (12×10 grid, #D97757), static in collapsed, walking animation in expanded
- **Concave arc shape**: Panel "grows from screen edge" with bezier quarter-ellipse arcs (15×30, kappa=0.5523)
- **Context-aware state machine**: Non-coding app → collapsed, coding app → expanded, notification → auto-dismiss on resolved
- **Click expanded → Open in Memora**: Opens project detail in main app with `NSApp.activate`

### Fixed
- Notification auto-dismiss race condition: uses hook timestamp instead of Dart poll time
- Stale request files cleaned on app startup (prevents phantom notifications from old sessions)
- AXObserver for instant window switch detection
- build.sh auto-installs to /Applications/ to prevent version mismatch

### Technical
- NSPanel at statusBar+1 level, pure black NSView (NSVisualEffectView cannot be pure black)
- Path drawn in visual coords (y=0 top) then flipped via `CGAffineTransform(scaleX:1, y:-1)` to CG coords
- `isGeometryFlipped` only affects sublayer positioning, NOT path rendering
- PostToolUse hook writes resolved timestamp to `~/.memora/beacon-resolved/latest`

## [0.2.18] — 2026-04-06

### Added
- **Pixel crab mascot**: Claude Code-style block crab with walking animation in expanded state
- **Click expanded → Open in Memora**: Opens project in main app via `NSApp.activate`, "Open ↗" label

### Fixed
- Notification auto-dismiss race: uses hook's own timestamp instead of Dart poll time

## [0.2.14] — 2026-04-05

### Added
- **Pet + indicator** persist across all states (collapsed/expanded/notification) for visual continuity
- **Keystroke forwarding**: `CGEvent.postToPid()` sends to target terminal, no focus switch
- **allowAllSupported configurable**: Dart-side `_allowAllRules` per source+tool (Edit/Write → 3 buttons, Bash → 2)

### Fixed
- 5 Beacon logic bugs: button count mismatch, auto-dismiss, keystroke sent to wrong terminal
- Removed full state: expanded click now opens Memora directly

## [0.2.8] — 2026-04-05

### Added
- **Collapsed state redesign**: Same arc shape as expanded, notch+80 width, crab + pulse indicator in notch area

### Fixed
- Arc 15×30 with correct tangent direction (horizontal at screen top, vertical at body edge)
- Content position starts from panel top (not menu bar bottom)
- Stale request file cleanup on startup (not 30s timeout)

## [0.2.3] — 2026-04-05

### Added
- **Concave arc shape**: Panel "grows from screen edge" with bezier quarter-ellipse arcs (kappa=0.5523)
- **build.sh auto-install**: Copies app to /Applications/ to prevent version mismatch debugging

### Fixed
- Reverted isGeometryFlipped (it flips entire shape upside down, not individual paths)
- Adaptive notification buttons: 2 or 3 based on allowAllSupported

## [0.2.0] — 2026-04-05

Beacon V2: systematic overhaul of visual design and authorization flow.

### Changed
- Hook PreToolUse returns "ask" (non-blocking), terminal and Beacon show prompts simultaneously
- Inverse corner arcs use CAShapeLayer (replaced mask-based approach)
- All visual coordinates drawn in visual space then flipped via CGAffineTransform

## [0.1.11] — 2026-04-03

Beacon V1: first floating panel + Hook PreToolUse + full interaction overhaul.

### Added
- **Memora Beacon**: Floating panel detects active coding app, shows context (collapsed/expanded/notification)
- **Hook PreToolUse**: Rust hook notifies Beacon before AI tool executes
- **Beacon V2 interaction**: Focus-screen tracking + notification UI + jump-to-window
- **LLM-powered Beacon summary**: Reads transcript, AI generates context summary
- **PostToolUse hook**: Instant notification dismiss + summary refresh on tool completion
- **Beacon Allow/Deny dual path**: Authorization works even after hook timeout
- **Beacon Notch fusion**: Dynamic Island positioning, pure black background fuses with notch
- **Incremental transcript copy**: With validation and full-copy fallback
- **Hotkey recorder**: 1/2/3 key combos via native NSEvent monitor

### Fixed
- Beacon no longer steals focus from active app
- Beacon follows active window's screen on multi-monitor setups
- AX window title traverses all windows as fallback
- Bundle ID corrected to com.eastworld.memora
- Session re-ingestion handles duplicate message IDs
- Default hotkey changed to ⌃⌘M (avoid ⌘M system minimize)
- Ghostty added to Beacon whitelist

### Changed
- Drift redesigned from audit tool to status dashboard

## [0.1.10] — 2026-04-03

### Fixed
- Antigravity (Google LanguageServer) API switched from reqwest to curl (macOS linker fix)

## [0.1.9] — 2026-04-02

### Added
- **Antigravity (Google) support**: Knowledge Items via LanguageServer API (.pb format)
- **Blackboard full history**: Reads all commits + sessions (no longer limited to 50/10)

## [0.1.8] — 2026-04-02

First notarized release — no more Gatekeeper warnings on macOS.
Content same as v0.1.7 with Apple notarization + staple applied.

## [0.1.7] — 2026-04-02

### Fixed (Independent Review)
- **F1**: commit-link processing moved after git sync to prevent silent association loss; design doc synced to actual implementation
- **F3**: Drift verdict cancellation now persists (DELETE from DB instead of UI-only clear)
- **F5**: flutter analyze 0 issues, flutter test 52/52 pass (fixed Ollama URL + custom_anthropic assertions)
- **F6**: Settings "Reinstall Hooks" button for retry after skip/failure
- **F7**: Branch filter shows empty state instead of silently falling back to all sessions

### Improved
- **F2**: Git auto-backup commands log failures to stderr via run_git() helper
- **F4**: Drift reads strategy files full text (up to 5000 chars); continue context expanded to 50 commits + 10 sessions
- Blackboard view toggle: Insights | vs Plan (replaces separate Drift card)
- Prev/next commit navigation buttons (replaces inaccurate scrollbar markers)
- Search: keyword highlighting, snippet around match, result count, clear project selection
- Session timeline: right-click commits for Recall/Branch
- Branch dialog: Browse button for target directory in Clone mode
- Responsive Blackboard cards (horizontal >700px, vertical otherwise)
- Titlebar: fullSizeContentView eliminates gap between traffic lights and content

## [0.1.6] — 2026-04-01

### Added
- **Drift Awareness**: Plan vs actual work comparison in Blackboard
- **Context-Linked Commits**: AI conversations auto-linked to git commits
- **i18n**: English + Chinese language switching
- **Onboarding tour**: 6-step interactive guide
- **Scrollbar minimap**: Git commit position markers
- UI fixes: scrollbar, version display, Branch dialog, menu descriptions

## [0.1.5] — 2026-04-01

### Added
- **Continue V2**: Cross-tool workspace creation — git clone + MEMORA_CONTEXT.md with full conversation history, git log, and project files
- **Drift Awareness**: Plan vs actual work comparison panel in Blackboard (3+1 layout), with user verdict buttons (OK / Circle back / Adopt)
- **Context-Linked Commits**: Hook detects commits during AI sessions and links them to conversation sessions; linked commits show 💬 icon
- **Onboarding tour**: 6-step interactive guide with spotlight overlay, replayable via ? button
- **i18n**: English + Chinese language support, switchable in Settings
- **Scrollbar commit markers**: Orange minimap on scrollbar showing git commit positions
- **Auto git protection**: ~/.memora/ data auto-committed on every sync (DB excluded)
- **Pretty DMG installer**: Dark background, drag-to-Applications layout via create-dmg
- **GitHub Release**: Public repo at github.com/4over7/Memora with wiki documentation

### Fixed
- Scrollbar markers now use estimated pixel heights instead of item index ratio
- Scrollbar thumb thickens on hover (6px → 10px) for easier mouse grabbing
- Version display shows full git hash (e.g. 0.1.5+998237f)
- Target directory in Clone mode now has Browse button for folder picker
- Recall/Branch context menu items have subtitle descriptions
- Excluded *.db from ~/.memora/ git protection (was causing 73GB disk bloat)

### Design Documents
- `docs/drift-awareness-design.md` — Three-layer drift detection design
- `docs/context-linked-commits-design.md` — Commit ↔ session binding via trailer
- `docs/vision-version-control-for-ai-era.md` — Vision: Git's consciousness layer

## [0.1.4] — 2026-04-01

### Added
- **Pretty DMG installer**: Dark background + drag-to-Applications layout (via create-dmg)

### Fixed
- Excluded *.db from ~/.memora/ git protection (was causing 73GB disk bloat)

## [0.1.3] — 2026-04-01

First DMG distribution with i18n and onboarding.

### Added
- **Continue V2**: Cross-tool workspace creation — git clone + MEMORA_CONTEXT.md
- **i18n**: English / Chinese switching in Settings
- **Onboarding tour**: 6-step interactive guide with spotlight overlay
- **Scrollbar commit markers**: Orange minimap for git commit positions
- **Auto git protection**: ~/.memora/ data auto-committed on every sync
- Ollama + custom provider support
- Draggable vertical divider in session timeline

### Fixed
- Version display shows git hash (e.g. 0.1.2+623fcb9)
- Guide replay button more visible in header

## [0.1.2] — 2026-03-31

### Added
- **Dual-track timeline**: Conversations and git commits side by side with draggable split ratio
- **Blackboard**: AI-powered project analysis (outcomes, discussions, unlanded ideas)
- **History Branching**: Worktree and Clone modes from any historical commit
- **Message-level Continue**: Right-click to export conversation as Markdown
- **Six-layer knowledge model**: L1 Persona through L6 Outcome
- **Multi-provider LLM**: 12 providers + Ollama + 2 custom modes
- **Full-text search**: Across conversations and git commits (FTS5)
- **SpeakOut import/export**: Bulk API key management
- **Sidebar collapse**: Collapsible project list
- **Manual project add**: Add projects not auto-detected
- **Brand colors**: Tool-specific colors (Claude orange, Codex purple, etc.)
- **Settings**: LLM providers, Persona configs, About page
- **52 Flutter tests** + Rust integration tests
- **Code signing**: Developer ID Application certificate
- **DMG packaging**: Automated build.sh with version auto-increment

### Supported Tools
- Claude Code (hook, real-time)
- Codex CLI (hook, real-time)
- OpenCode (hook + SQLite)
- Cursor (SQLite polling)

## [0.1.1] — 2026-03-31

### Fixed
- Version display: show version only, no duplicate build number

## [0.1.0] — 2026-03-29

### Added
- Initial release
- Flutter desktop (macOS) + Rust backend via flutter_rust_bridge
- Three Rust crates: memora-core, memora-ingest, memora-hook
- Hook-driven conversation capture
- SQLite storage with events.jsonl pipeline
- Basic project view with session list
