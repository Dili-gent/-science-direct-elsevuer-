---
name: lit-fetch
description: 用户要「下载论文全文 / 找 PDF / 批量下文献 / 检索 Web of Science 或 ScienceDirect / 用学校权限拿全文」时使用；从 DOI 出发，先用 Unpaywall/OpenAlex/Crossref/Europe PMC/arXiv 找开放获取副本，找不到时用受控浏览器（复用你自己的机构登录）抓出版社 PDF，支持批量、断点续跑、逐篇台账与报告全部落盘。
---

# lit-fetch：从 DOI 到 PDF

一句话：**先找合法的开放获取副本，找不到再动你自己的机构权限，绝不做任何绕过。**

## 何时使用

- 用户给了一批 DOI / 题录 / Excel 清单，要下载全文。
- 用户说「这篇下不到」「帮我找 PDF」「批量下文献」。
- 用户要在 **Web of Science / ScienceDirect** 里检索并导出题录。
- 用户有学校订阅，想用已登录的浏览器抓全文。

**不要用**：只查一条文献的元数据（直接 `web_fetch` DOI 页更快）；要从零检索并筛选（用 `paper-hunter` skill）；要绕过付费墙或验证码（本工具明确不做）。

## 三层能力与边界

| 层 | 靠什么 | 需要账号 | 典型命中 |
| --- | --- | --- | --- |
| L1 开放获取 | Unpaywall / OpenAlex / Crossref / Europe PMC / arXiv | 否（建议填邮箱） | OSTI、arXiv、Nature OA、Springer、机构库 |
| L2 出版社直链 | DOI 前缀 → URL 模板 | 否 | 同上 + 少数 bronze OA |
| L3 机构订阅 | 复用**你自己**已登录的浏览器会话 | 是，你自己登 | ScienceDirect、Wiley、ACS、RSC、IOP |

**明令禁止的行为**（代码里没有对应开关，不要试图加）：
破解验证码、伪造/共享 cookie、使用 Sci-Hub 等盗版镜像、把机构凭据写进配置或日志。

## 动手前先自检

```powershell
# 1) 环境：Node、浏览器、目录、邮箱
litget doctor

# 2) 想用机构权限的话，确认会话还活着
litget session check
```

`doctor` 里两项没打勾就先修：
- **联系邮箱**：Unpaywall 强制要求，`litget.config.json` 的 `http.email` 或 `$env:LITGET_MAILTO`。
- **浏览器**：找不到就设 `browser.exe` 或 `$env:LITGET_BROWSER`。

## 主流程

### 第 1 步：看一个 DOI 有什么可下

```powershell
litget resolve 10.1038/nature12373
```

输出会列出**每个元数据源是否可用**，以及按主机可信度排序的候选地址（信任 2 = 裸 HTTP 通常能拿到，0 = 基本一定被拦）。
先跑这一步，能避免在注定失败的主机上浪费重试。

### 第 2 步：单篇下载

```powershell
litget fetch 10.1038/nature12373
```

内部是「HTTP 与浏览器并行竞速」：哪个先拿到合格 PDF 用哪个。
实测同一台机器上 arXiv 走裸 HTTPS 要 126 秒，走浏览器 0.12 秒——所以不要假设 HTTP 更快。

### 第 3 步：批量

先有 worklist（JSON 数组，字段 `doi` 必填，`title/year/authors/venue/no` 选填）：

```powershell
litget batch examples/worklist.json --concurrency 3
```

产物：

```
work/fetch.jsonl      逐篇台账（append-only，一次尝试一行）
work/report.md        可读报告（状态分布、来源主机、明细）
work/report.json      机器可读
work/manifest.json    DOI → 已落盘文件（续跑靠它，不用再联网）
work/cache.json       元数据与候选计划缓存
```

**续跑就是再跑一次同一条命令**：已下载的会标 `existing` 并秒过（实测 4 篇续跑 15 秒，其中 3 篇 0 ms）。

### 第 4 步：L1/L2 拿不到，走机构权限

```powershell
litget session check                 # 没会话就下一步
litget login sciencedirect           # 打开登录页，由你本人输账号密码
litget session check                 # 回来验证
litget fulltext 10.1016/j.jpowsour.2009.04.024
```

`litget login` **只负责打开页面**，不读取、不记录、不转发密码。遇到人机验证时脚本会停下并把窗口留给你，**不会尝试自动破解**。

### 第 5 步：检索（需要订阅）

```powershell
litget search wos "TS=(perovskite) AND PY=(2022-2026)" --limit 25 --out wos.json
litget search sd  "oxygen electrode degradation" --from 2020 --type FLA --limit 25
```

### 第 6 步：交付前校验

```powershell
litget verify library/       # 头、体积、页数；不合格会点名
```

报告里的每个数字都来自台账，**没做的校验不会出现在报告里**。

## 状态词表

`ok` 成功 · `existing` 已存在 · `no_candidates` 没有可用链接 · `no_pdf_link` 文章页无 PDF 链接 · `blocked` 被反爬拦 · `paywalled` 需订阅 · `throttled` 被限流 · `not_pdf` 返回的不是 PDF · `not_found` 目标不存在 · `stopped` 触发预算提前停止 · `error` 网络/脚本错误

判读优先级：`blocked`/`paywalled` → 走 L3；`no_candidates` → 确认邮箱与元数据源；`not_pdf` → 多半是验证页，删掉重下。

## 硬规矩

1. **不要为了「都得下来」而关掉 TLS 校验**（`--insecure`）。那条路会让 MITM 不可见，且实测不关也完全能跑通。
2. **不要提高并发到 6 以上**打免费 API。OpenAlex/Crossref 有 polite pool，Semantic Scholar 无 key 时 ~73% 429，这不值得赌。
3. **失败要如实说**。`no_pdf_link` 不等于「这篇不存在」，`blocked` 不等于「期刊不开放」。
4. **机构全文只在你自己有权限时用**，并遵守数据库的使用条款。
5. 先 `--dry-run` 看计划，再真正下载。

## 资源

| 需要什么 | 读哪个文件 |
| --- | --- |
| 安装、配置、命令全表、设计取舍 | `README.md` |
| 各源 API 细节、主机可信度分级、出版社规则 | `docs/sources.md` |
| 每个出版社的已知坑与状态判定 | `docs/publishers.md` |
| 症状 → 原因 → 处理 | `docs/troubleshooting.md` |
| 离线自检（改代码后先跑） | `scripts/selftest.js` |
| **要从零检索、做文献综述、建筛选表** | skill `paper-hunter`（本 skill 只管拿到全文） |

**和 `paper-hunter` 的分工**：`paper-hunter` 把研究问题变成「值得读的 A/B/C 分级清单」，
本 skill 把清单里的 DOI 变成 PDF。衔接方式：把候选的 `doi` 收集成 worklist
（JSON 数组或 JSONL，每项至少含 `doi`），然后 `litget batch <file>`。

如果用户还处在「这个方向有谁做过」的阶段，先用 `paper-hunter`；拿到清单后再回来下载。
