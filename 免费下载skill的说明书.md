# lit-fetch

从 DOI 到本地 PDF：**先找开放获取副本，找不到再用你自己的机构权限，绝不做任何绕过。**

一个零依赖的命令行工具，同时也是一个 [DeepSeek Harness](https://github.com/deepseek-ai) skill。
适合「手里有一批题录，要把能合法拿到的全文都拿下来，并清楚知道哪些拿不到、为什么」的场景。

---

## 它做什么

```
DOI ──► 元数据与 OA 发现 ──► 候选地址排序 ──► 双传输竞速下载 ──► 校验 ──► 台账/报告
        Unpaywall            按主机可信度      HTTP ∥ 浏览器
        OpenAlex             分级             谁先拿到合格PDF用谁
        Crossref
        Europe PMC
        arXiv
```

| 能力 | 说明 |
| --- | --- |
| 开放获取下载 | 多源聚合去重、按「主机是否真的给 PDF」排序，不是按 API 声称的 OA 状态 |
| 出版社直链 | DOI 前缀 → URL 模板，机械可推导的那部分（OSTI/arXiv/Nature/Springer/IOP/RSC…） |
| 机构订阅全文 | 复用**你自己**已登录的浏览器会话抓 ScienceDirect/Wiley/ACS 等 |
| 批量与续跑 | append-only 台账、DOI→文件清单、元数据缓存；重跑已下载的秒过 |
| 校验 | `%PDF-` 头、`%%EOF` 尾、体积下限、页数；不合格会点名 |
| 报告 | 状态分布、失败归因、按来源主机统计；每个数字都来自台账 |

## 它不做什么

这个边界是设计出来的，不是没实现：

- **不绕验证码、不绕付费墙。** 遇到人机验证就停下，把浏览器窗口留给你。
- **不碰盗版镜像。** 没有 Sci-Hub / LibGen / Z-Library / Anna's Archive 的任何代码或链接。
- **不接触你的密码。** `login` 只负责打开登录页，不读取、不记录、不转发凭据。
- **不假装覆盖 100%。** 闭源且无机构权限的论文就是拿不到，报告会如实写 `no_candidates` 或 `blocked`。

---

## 快速开始

```bash
git clone <your-repo-url> lit-fetch
cd lit-fetch

# 1. 生成配置，填上你的邮箱（Unpaywall 强制要求）和浏览器路径（可留空自动探测）
node bin/litget.js config init

# 2. 自检：Node、浏览器、目录、邮箱
node bin/litget.js doctor

# 3. 先看一个 DOI 有哪些可下地址
node bin/litget.js resolve 10.1038/nature12373

# 4. 下载
node bin/litget.js fetch 10.1038/nature12373

# 5. 批量
node bin/litget.js batch examples/worklist.json
```

**要求**：Node ≥ 21（用到内置 `WebSocket` 驱动浏览器）。除此之外**零依赖**，不用 `npm install`。
机构订阅那部分需要一个 Chromium 系浏览器（Edge / Brave / Vivaldi / Quark 都可以）。

> Windows 上没装 Node 也没关系，DeepSeek Harness 自带一个：
> `%USERPROFILE%\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe`

---

## 三层能力与需要的条件

| 层 | 依赖 | 需要账号 | 说明 |
| --- | --- | --- | --- |
| **L1 开放获取** | 只有网络 | 否 | Unpaywall / OpenAlex / Crossref / Europe PMC / arXiv。建议填邮箱进 polite pool |
| **L2 出版社直链** | 只有网络 | 否 | 对 OSTI、arXiv、Nature OA、Springer、部分 bronze OA 有效 |
| **L3 机构订阅** | 一个浏览器 + 你自己的会话 | 是 | 复用你已登录的 profile，不新建登录、不碰密码 |

**只想要 L1/L2？** 不需要配置浏览器，也不需要学校账号：

```bash
node bin/litget.js fetch 10.1038/nature12373 --no-browser
```

---

## 配置

配置来源按优先级合并（后面的覆盖前面的）：

1. 内置默认值（`lib/config.js` 的 `DEFAULTS`）
2. `<仓库>/litget.config.json`
3. `~/.config/litget/config.json`（Windows：`%APPDATA%\litget\config.json`）
4. `$LITGET_CONFIG` 指向的文件
5. 环境变量

`node bin/litget.js config init` 会生成一份完整模板。**只有两项是真正要你填的**：

```jsonc
{
  "http": {
    // Unpaywall 强制要求；OpenAlex/Crossref 也用它进 polite pool。
    // 别用 example.com，Unpaywall 会直接 422 拒绝。
    "email": "you@your-university.edu"
  },
  "browser": {
    // 留空则自动探测。要固定就用完整路径。
    "exe": "",
    // 永不启动的浏览器 id。默认禁 Chrome；不想禁就删掉这条。
    "blocked": ["chrome"]
  }
}
```

常用环境变量（等价于改配置，适合临时覆盖）：

| 变量 | 对应配置 | 用途 |
| --- | --- | --- |
| `LITGET_MAILTO` | `http.email` | 联系邮箱 |
| `LITGET_BROWSER` | `browser.exe` | 浏览器完整路径 |
| `LITGET_PROFILE` | `browser.profileDir` | 机构会话所在的 profile 目录 |
| `LITGET_LIBRARY` | `paths.library` | PDF 落盘目录 |
| `LITGET_WORK` | `paths.work` | 台账/缓存目录 |
| `LITGET_INSECURE_TLS=1` | — | **不推荐**，仅在坏掉的代理后面临时用 |
| `LITGET_DEBUG=1` | — | 打印完整异常栈 |

完整字段含义见 `lib/config.js` 顶部的注释——每个字段都有说明，不在 README 重复。

---

## 命令全表

```bash
litget doctor [--net]         # 自检；--net 额外测 Crossref 连通性
litget resolve <doi>          # 看候选地址、每个源是否可用、按可信度排序
litget fetch <doi|文件> ...   # 下载；可给多个 DOI 或一个 worklist 文件
litget batch <worklist.json>  # 批量，可续跑，写报告
litget verify [目录]          # 离线校验已下载 PDF
litget report                 # 汇总台账
litget session check          # WoS / ScienceDirect 会话是否有效
litget login wos|sciencedirect [--school "学校名"]
litget search wos|sd "<检索式>" [--limit N] [--from YYYY] [--to YYYY] [--type FLA|REV]
litget fulltext <doi> [--url 文章页] [--select "CSS选择器"]
litget browser open|close|status
litget config init|show
```

所有命令都支持 `--json`（机器可读）。常用开关：
`--dry-run` 只看计划、`--force` 忽略已存在、`--only 1,2,5` 只跑这几条、
`--concurrency N`、`--include-hard-hosts`（连出版社站点也试一遍，通常白费）、`--verbose`。

### worklist 格式

数组或 JSONL，只有 `doi` 必填：

```json
[
  { "no": 1, "doi": "10.1038/nature12373", "title": "...", "year": 2013, "authors": ["Kucsko, G."] },
  { "doi": "10.2172/1411042" }
]
```

字段名有容错：`rank/no/id`、`title/TI/name`、`journal/JF/venue`、`authors/author/AU` 都能认；
没有 DOI 时会尝试从 `url` 里抠。

---

## 产物长什么样

```
library/                                    # 你的 PDF
  2013_Kucsko_Nanometre-scale thermometry in a living cell.pdf
work/
  fetch.jsonl      # append-only 台账，一次尝试一行
  report.md        # 人读报告（UTF-8 BOM + CRLF，Windows 直接打开不乱码）
  report.json      # 机器可读
  manifest.json    # DOI → 已落盘文件（续跑靠它）
  cache.json       # 元数据与候选计划缓存（删掉可强制重新解析）
```

**续跑**：命令不变，再跑一次即可。已下载的标 `existing` 直接跳过。
实测 4 篇的批次，首次 ~150 秒，续跑 **15 秒**（其中 3 篇走 manifest 短路，0 ms）。

文件名默认 `{year}_{first_author}_{title}`。缺字段会自动省略，而不是拼出 `untitled_n.d._NA`；
同名冲突追加 `_2`、`_3`，绝不覆盖。要改就设 `download.nameTemplate`
（可用 `{no} {year} {first_author} {title} {venue} {doi} {source}`）。

---

## 状态词表

| 状态 | 含义 | 该怎么办 |
| --- | --- | --- |
| `ok` | 成功 | — |
| `existing` | 已存在，跳过 | — |
| `no_candidates` | 没有找到任何候选地址 | 确认邮箱填了、DOI 拼对了；这篇可能真没 OA 副本 |
| `no_pdf_link` | 文章页里挖不到 PDF 链接 | 有机构权限就走 L3 |
| `blocked` | 被反爬/人机验证拦 | 有权限走 L3；脚本不会绕 |
| `paywalled` | 需要订阅 | 走 L3 |
| `throttled` | 被限流 | 等一会儿，降并发 |
| `not_pdf` | 拿到的不是 PDF（多半是验证页） | 删掉重下；或用 L3 |
| `not_found` | 目标不存在 | 检查 DOI |
| `stopped` | 触发时间预算，提前停止 | 单独重试这一篇 |
| `error` | 网络/脚本错误 | 看 `--verbose` 的尝试明细 |

---

## 作为 DSH skill 使用

本仓库根目录的 `SKILL.md` 就是 skill 定义。装进 DeepSeek Harness：

```powershell
# 方式一：用户级（所有会话可用）
Copy-Item -Recurse lit-fetch "$env:USERPROFILE\.dsh\skills\lit-fetch"

# 方式二：项目级（只在某个工作区可用）
Copy-Item -Recurse lit-fetch "<工作区>\.dsh\skills\lit-fetch"
```

装好后会话里就能直接说「把这个 DOI 的全文下下来」，模型会按 `SKILL.md` 的流程走。

配合 `paper-hunter` skill 使用：`paper-hunter` 负责检索与筛选，它产出的题录转成 worklist 后交给 `lit-fetch` 下载。

---

## 迁移到另一台机器 / 分享给别人

本仓库是**自包含**的：代码、文档、skill 定义全在里面，没有 `npm install`，没有编译步骤。
只要目标机器有 Node ≥ 21，把目录搬过去就能跑。

### 一、怎么搬（三种方式，任选）

**方式 A：走 git（推荐，便于更新）**

```bash
# 本机：推到 GitHub（或任何 git 远程）
git remote add origin git@github.com:<你>/lit-fetch.git
git push -u origin main

# 目标机器
git clone git@github.com:<你>/lit-fetch.git
cd lit-fetch
```

**方式 B：直接拷目录**

把整个 `lit-fetch/` 文件夹复制过去（U 盘、网盘、`scp`、共享盘都行）。
`.git` 目录可带可不带——不带也能正常运行，只是失去版本历史。

```powershell
# Windows 示例
Copy-Item -Recurse lit-fetch D:\somewhere\lit-fetch
```

```bash
# macOS / Linux 示例
rsync -a --exclude .git lit-fetch/ user@host:~/lit-fetch/
```

**方式 C：打包成压缩包**

```bash
git archive -o lit-fetch.zip HEAD      # 不带 .git，最干净
```

### 二、目标机器上第一次跑（三步）

```bash
cd lit-fetch

# 1. 生成配置模板
node bin/litget.js config init

# 2. 填两处：http.email（必填）和 browser.exe（留空则自动探测）
#    编辑 litget.config.json

# 3. 自检
node bin/litget.js doctor
node bin/litget.js fetch 10.1038/nature12373    # 冒烟测试
```

`doctor` 全部打勾就能用了。**不需要 npm install，也不需要 Python。**

**没有 Node？** 两条路：

```powershell
# 装了 DeepSeek Harness 的机器，直接用自带的 Node
$node = "$env:USERPROFILE\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\node\bin\node.exe"
& $node bin/litget.js doctor
```

```bash
# 没装 DSH 就装一个 Node：https://nodejs.org（LTS 即可，要 ≥21）
```

### 三、让另一个 agent 用

**同一个 DSH 里的另一个会话 / 工作区**：把目录放进对应 skills 目录即可，会话会自己发现。

```powershell
# 所有会话可用（用户级）
Copy-Item -Recurse lit-fetch "$env:USERPROFILE\.dsh\skills\lit-fetch"

# 只在这个工作区可用（项目级）
Copy-Item -Recurse lit-fetch "<工作区>\.dsh\skills\lit-fetch"
```

装完新开会话，说「把这个 DOI 的全文下下来」就会命中。

**非 DSH 的 agent**（Claude Code / Cursor / Codex 等）：它们不认 DSH 的 skill 目录，
但 `SKILL.md` 就是一份普通 Markdown 操作手册，直接喂给它们即可：

- Claude Code：放进 `.claude/skills/lit-fetch/SKILL.md`，或写进 `CLAUDE.md` 引用
- Cursor：写进 `.cursor/rules/lit-fetch.mdc`
- 其它：把 `SKILL.md` 内容粘进系统提示，或让它先 `read` 这个文件

这些 agent 只要能执行 shell 命令，就能用 `node bin/litget.js ...`，能力完全一样。

### 四、什么东西**不能**带走

| 项目 | 为什么 | 正确做法 |
| --- | --- | --- |
| `litget.config.json` | 里面是你的邮箱（和个人偏好） | 目标机器重新 `config init` 生成 |
| `library/`、`work/` | 你已下载的 PDF 和台账 | 需要就单独拷，别混进仓库 |
| **浏览器 profile 里的机构登录会话** | **这是你的账号凭据，绝不能复制或分享** | 目标机器自己 `litget login wos` 重新登录 |
| `.git/config` 里的远程地址/token | 可能含凭据 | 别拷 `.git`，或重新设远程 |

前两项 `.gitignore` 已经排除。第三项是硬边界：机构订阅权限属于**你本人**，
换机器、换人都不该复用——这也是 L3 层只调用「你自己的浏览器会话」而不内置任何凭据的原因。

### 五、跨平台的已知情况

| 平台 | 状态 | 说明 |
| --- | --- | --- |
| Windows | **已实测** | Edge / Chrome(可选) / Brave / Vivaldi / Quark 探测与 CDP 抓取均验证过 |
| macOS | 代码已支持，**未实测** | 探测 `/Applications/*.app/Contents/MacOS/*`；路径与 profile 目录已按平台分支 |
| Linux | 代码已支持，**未实测** | 探测 PATH 上的 `google-chrome`/`chromium`/`microsoft-edge`；`--no-sandbox` 可按需开 |

跨平台差异只集中在三处，都写在 `lib/config.js` 里，出问题改那里即可：
浏览器候选路径（`browserCandidates`）、配置目录（`platformConfigDir`）、profile 目录（`defaultProfileDir`）。

macOS/Linux 上如果 `doctor` 找不到浏览器，最快的解法是显式指定：

```bash
LITGET_BROWSER=/usr/bin/chromium node bin/litget.js doctor
```

### 六、升级已安装的副本

因为 skill 是**复制**进去的（不是软链），升级要重新覆盖：

```powershell
# 拉最新代码后，覆盖技能目录（保留你的 litget.config.json）
git pull
Copy-Item -Recurse -Force lit-fetch\bin, lit-fetch\lib, lit-fetch\docs, `
  lit-fetch\scripts, lit-fetch\SKILL.md, lit-fetch\README.md `
  "$env:USERPROFILE\.dsh\skills\lit-fetch\"
```

改完跑一次 `node scripts/selftest.js` 确认没坏。

---

## 几个设计取舍（为什么这么写）

**为什么用浏览器而不是纯 HTTP？**
因为同一台机器上，同一个 arXiv PDF：裸 HTTPS 要 **126 秒**，浏览器 **0.12 秒**。
所以主路径是「两个传输并行竞速，谁先拿到合格 PDF 用谁」，而不是先 HTTP 后降级。

**为什么按主机排序，而不是按 API 说的 OA 状态？**
实测命中率由**目标主机**决定：OSTI/arXiv/Nature/Springer/机构库 ≈ 100%，IOP/Wiley/ACS/MDPI ≈ 0%（403 或 200-HTML 软墙）。
元数据源说的 `is_oa=true` 对「能不能下」几乎没有预测力。

**为什么 Fetch 拦截要在放行之前读 body？**
内建 PDF 查看器走特权通道，导航结束后 `Network.getResponseBody` 拿到的已经不是 PDF 了。
必须在 `Fetch.requestPaused` 里先 `getResponseBody` 再 `continueRequest`。
（早期版本按直觉先放行后读取，结果抓到 360 字节的 HTML 壳。）

**为什么 4xx 不重试？**
服务器已经答复了，重试只会得到同样的答复，白白消耗时间和 IP 信誉。

**为什么有单篇时间预算？**
一篇付费墙论文挂着 6 个慢落地页时，会把整批拖住。默认 120 秒/篇，超了记 `stopped` 继续下一篇。

---

## 目录结构

```
lit-fetch/
├── bin/litget.js           CLI 入口与子命令
├── lib/
│   ├── config.js           配置合并、浏览器探测
│   ├── browser.js          CDP 客户端（零依赖，Node 内置 WebSocket）
│   ├── pdf.js              PDF 抓取四策略 + 校验
│   ├── http.js             限速、重试、PDF 下载
│   ├── sources.js          OA 发现层 + 主机可信度表
│   ├── adapters.js         出版社 URL 规则
│   ├── miner.js            落地页挖 PDF 链接
│   ├── oa.js               解析与下载编排
│   ├── institutional.js    WoS/ScienceDirect 检索、登录、全文
│   ├── batch.js            批量与报告
│   ├── cache.js            元数据缓存与文件清单
│   ├── worklist.js         worklist / ledger I/O
│   ├── s2.js               Semantic Scholar（可选）
│   └── util.js             命名、DOI 归一、标题比对
├── scripts/selftest.js     离线自检（不联网）
├── docs/                   sources / publishers / troubleshooting
├── examples/worklist.json
├── SKILL.md                DSH skill 定义
└── README.md
```

---

## 验收与自检

```bash
node scripts/selftest.js     # 离线，2 秒，改代码后先跑这个
node bin/litget.js doctor    # 环境检查
```

`selftest.js` 覆盖：PDF 魔数/截断判定、文件名清洗（Windows 保留名、全角标点、长度）、
DOI 归一化、同名冲突分配、状态聚合、报告生成、配置合并与浏览器排除逻辑。**不联网**。

真实网络路径的验收方式：

```bash
node bin/litget.js resolve 10.1038/nature12373   # 应该列出多个源与候选
node bin/litget.js fetch   10.1038/nature12373   # 应该拿到 ~2.3MB PDF
node bin/litget.js verify  library/              # 应该全部合格
```

---

## 已知限制

- **元数据源会变。** 站点改版后选择器要修，改 `lib/institutional.js` 的 `SIGNALS`/解析脚本。
- **出版社 URL 规则会变。** 改 `lib/adapters.js`，那里是纯数据 + 小函数。
- **Semantic Scholar 无 key 时限流严重**（实测 ~73% 429），所以默认不启用；有 key 就设 `oa.semanticScholarKey`。
- **`fulltext` 的具体 PDF 选择器因出版社而异**，没有通用解，用 `--select` 指定。
- **不适用于大模型训练数据级的抓取**——限速是刻意保守的。

---

## 许可与使用边界

MIT（见 `LICENSE`）。同时注意那份文件末尾的 **Acceptable use** 一节：
本工具只做「用你自己的权限、合法地取回你能取回的东西」，不提供任何绕过手段。
通过订阅数据库下载的内容仍受该数据库使用条款约束。

---

## 致谢

提炼自一个真实在用的文献下载工作流：把分散的临时脚本收敛成「配置化 + 可续跑 + 如实报告」的工具，
并修掉了原脚本里几个会骗人的地方（状态被后者覆盖、报告写了代码没做的校验、并发过高引发 429、文件名绑定列表序号等）。
