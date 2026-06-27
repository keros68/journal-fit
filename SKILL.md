---
name: journal-aware-revision
description: SCI期刊感知的论文诊断工具：需求驱动，先问清楚要查什么，再按需对照期刊要求+同刊惯例，给出修改方向。不代写。
version: 3.4.0
---

# Journal-Aware Revision

交互式论文诊断工具。对照目标期刊的投稿要求和同刊发表论文的风格特征，检查用户草稿中的问题，给出修改方向。

**只做两件事：检查 + 给修改方向。** 不帮写、不生成替换文字、不做写作模板。

## 快速配置（新用户必读）

### 零配置能用的部分

以下 API 完全免费、无需 key、直接 curl：

| 功能 | API | 备注 |
|------|-----|------|
| 范文检索 | OpenAlex | 无限制 |
| 范文摘要 | Semantic Scholar | rate limit ~1 req/s |
| 范文元数据 | CrossRef | 无限制 |
| OA 全文获取 | Europe PMC | OA 论文免费下载 |
| Guidelines (Elsevier/MDPI) | Jina Reader | `r.jina.ai/{url}`，免费 |

**零配置时 Guidelines 覆盖率：~50%**（Elsevier/MDPI/Springer 部分可用，Wiley/ACS/T&F 不行）。

### 最小配置（1 个 key → 100% 覆盖）

只需配置 **Tavily API key**（免费 dev 额度 1000 次/月）：

1. 注册：https://tavily.com（免费）
2. 获取 API key
3. 写入 Hermes 配置：

```json
// ~/.claude/settings.json
{
  "env": {
    "TAVILY_API_KEY": "tvly-your-key-here"
  }
}
```

或直接设为环境变量：`export TAVILY_API_KEY=tvly-your-key-here`

**配置后 Guidelines 覆盖率：100%**（6/6 出版商全覆盖，详见 `references/guidelines-fallback-matrix.md`）。

### 可选：草稿解析依赖

```bash
pip install python-docx pymupdf   # .docx 和 .pdf 解析
# .md/.txt 无需额外依赖
```

### 配置检查

首次使用时，agent 应检查 `TAVILY_API_KEY` 是否存在：

```python
import os
has_tavily = bool(os.environ.get("TAVILY_API_KEY"))
if not has_tavily:
    # 告知用户：Wiley/ACS/T&F 期刊的 Guidelines 将无法自动获取
    # 指引：https://tavily.com 注册免费 key
```

## 触发条件

- 用户提到"按XXX期刊要求整改/修改论文/检查"
- 用户提供期刊名 + 论文草稿，要求对齐期刊风格
- 用户提到"投稿前检查""pre-submission check"

## Phase 0 — 需求澄清（必须先做）

用户触发后，**不要直接开跑**，先问清楚三件事：

**Q1. 目标期刊是哪个？**
（需要刊名或 ISSN，用于后续 Guidelines 获取 + 范文检索）

**Q2. 论文草稿在哪？**
（本地文件路径，支持 .docx/.pdf/.md/.txt）

**Q3. 你主要想查什么？**

给用户以下选项（用 clarify 工具）：

- **A. 格式合规检查** — 字数/章节结构/参考文献格式/图表数量/摘要格式等硬性要求
- **B. 风格对齐检查** — 摘要长度/引用密度/关键词数量/篇幅等软性指标，对标同刊惯例
- **C. 语言风格检查** — 被动语态/句长/hedging 等语言层面（需要 OA 范文全文支持）
- **D. 特定部分对标** — 只看某个章节（如摘要/引言/结论），和同刊范文做结构对比
- **E. 全面检查** — A + B 组合，先过格式再看风格

> 用户选什么，后面就只跑什么。不跑无关的步骤。

用户回答后，确认理解再继续。

## Phase 1 — 期刊信息获取

### 1.1 基础信息

用 `letpub` skill 获取期刊 IF、分区、审稿周期、OA/APC 等基础信息。

### 1.2 Author Guidelines

> ⚠️ **禁止用"出版商通用要求"替代真实 Guidelines。**
> 实测发现，用 Elsevier 通用默认值替代 STOTEN 真实 Guidelines，**13 项中 6 项出错**（包括"Graphical Abstract 可选→实际必需"、"参考文献格式要求 Vancouver→实际投稿时任意格式"、"摘要上限 250→实际 300 词"）。没有真实 Guidelines 的格式检查不如不做。

**三层 fallback 策略（按顺序尝试，成功即停，无 Tavily key 时自动跳过 Layer 1/3）：**

> **首次运行时先检查 key：** `os.environ.get("TAVILY_API_KEY")`。无 key 则直接从 Layer 2 (Jina) 开始，并在输出中告知用户"配置 Tavily key 可解锁 Wiley/ACS/T&F 期刊覆盖（免费，见快速配置章节）"。

#### Layer 1 — Tavily extract 直抓官方页（需配置 `TAVILY_API_KEY`）

```bash
curl -s -X POST "https://api.tavily.com/extract" \
  -H "Authorization: Bearer $TAVILY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"urls":["{guidelines_url}"], "extract_depth":"advanced", "timeout":60}'
```

**按出版商效果（2026-06-27 实测 6 家）：**
- ✅ **Elsevier**（STOTEN/Water Research/JHM）— 完整提取（83K–108K chars），含所有格式要求
- ✅ **MDPI**（Water）— 完整提取（108K chars）
- ⚠️ **Springer Nature**（Nature Climate Change）— Tavily 能返回 13K chars，但大部分是导航/Cookie 文本，正文关键信息提取不完整
- ❌ **Wiley / ACS / Taylor & Francis** — 被 Cloudflare/bot 检测拦截，extract 失败

#### Layer 2 — Jina Reader (r.jina.ai)（免费，无需 key）

```bash
curl -s "https://r.jina.ai/{guidelines_url}"
```

覆盖范围和 Layer 1 几乎一样（Elsevier/Springer/MDPI 能抓到，Wiley/ACS/T&F 同样被拦）。**主要价值是 Tavily key 耗尽或不可用时的免费备选。**

#### Layer 3 — Tavily search → 第三方聚合站（全覆盖）

当 Layer 1/2 失败时（Wiley/ACS/T&F），用 Tavily **search** API 搜期刊 guidelines：

```bash
curl -s -X POST "https://api.tavily.com/search" \
  -H "Authorization: Bearer $TAVILY_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"{期刊名} author guidelines word limit abstract figures", "search_depth":"advanced", "include_answer":true}'
```

Search API 的 AI answer 直接返回结构化关键数据（如"GCB 正文上限15000词，摘要250词结构化"）。然后对搜索结果中第三方聚合站 URL（manusights.com / scispace.com）用 extract 抓全文：

| 第三方站 | 覆盖 | 数据质量 |
|---------|------|---------|
| manusights.com | Wiley, ACS 等 | 高：结构化表格（字数/摘要/图表/参考文献逐项对比） |
| scispace.com | T&F 等 | 中：格式化要求摘要 |

> ⚠️ 第三方数据可能滞后于官方。输出时标注"数据来源：第三方聚合站（manusights.com），投稿前核对官方最新 Guidelines"。

#### Layer 4 — 手动粘贴（最后兜底）

所有自动方案均失败时，让用户复制粘贴 Guidelines 核心部分（通常 1-2 页）。

**常见 Guidelines URL 模式（用于构造 Layer 1 请求）：**
- Elsevier: `https://www.elsevier.com/journals/{NAME}/{ISSUE}/guide-for-authors`
- Springer: `https://www.springer.com/journal/{ID}/submission-guidelines`
- MDPI: `https://www.mdpi.com/journal/{NAME}/instructions`
- Wiley: `https://onlinelibrary.wiley.com/page/journal/{ISSN}/home/author-guidelines`
- ACS: `https://pubs.acs.org/journal/{CODE}/submission-guidelines`
- AGU: `https://agupubs.onlinelibrary.wiley.com/journal/{CODE}/about/author-guidelines`

### 1.3 Guidelines 结构化提取

从**实际抓取到的 Guidelines 文本**中提取硬性要求清单。**每一条都必须有 Guidelines 原文支撑，不可凭出版商类型猜测。**

> **反例（STOTEN 测试中的真实教训）：**
> - 猜"参考文献格式 = Vancouver (Numbered)" → 实际投稿时不做要求，接受任何一致格式
> - 猜"参考文献风格 = Numbered" → 实际是 Author-Year (Harvard style)
> - 猜"Graphical Abstract = 可选" → 实际是必需（"You are required to provide"）
> - 猜"摘要 ≤ 250 words" → 实际 ≤ 300 words
> - 猜"关键词 ≤ 6 个" → 实际 1-7 个
> - 猜"参考文献无数量限制" → 实际建议 ≤ 50 篇（research paper）
>
> 这些差异会直接导致 P0/P1 判断错误。**没有真实 Guidelines 就不做格式检查。**

| 维度 | 提取方法 |
|------|----------|
| 稿件类型 | 搜索 "Article types" / "manuscript types" 段落 |
| 字数限制 | 搜索 "word" + 数字 |
| 摘要 | 搜索 "abstract" + "word"/"paragraph"/"structured" |
| 关键词 | 搜索 "keyword" + 数字范围 |
| Highlights | 搜索 "highlight" + "bullet"/"character" |
| Graphical Abstract | 搜索 "graphical abstract" + "required"/"optional" |
| 参考文献 | 搜索 "reference" + "limit"/"total"/"format"/"style" |
| 图表 | 搜索 "figure"/"table" + "limit"/"resolution"/"dpi" |
| 必需声明 | 搜索 "Data Availability"/"CRediT"/"declaration"/"ethics" |
| 语言 | 搜索 "English" + "American"/"British" |
| 行号/格式 | 搜索 "line number"/"continuous"/"format" |

## Phase 2 — 草稿解析

解析用户提供的论文草稿：
- `.docx` → python-docx 提取文本
- `.pdf` → pymupdf 或 z-smart-xparse 提取文本
- `.md/.txt` → 直接读取

提取与期刊要求/同刊惯例对照所需的特征维度。

## Phase 3 — 按需检查

**根据 Phase 0 用户选择的检查类型执行，不跑无关的步骤。**

### 3.1 格式合规检查（需求 A / E）

直接用 Phase 1 的 Guidelines Checklist 逐项对照草稿：

| 检查项 | 状态 | 当前值 | 期刊要求 | 偏离 |
|--------|------|--------|----------|------|
| 全文字数 | ✅/❌ | XXXX | XXXX-XXXX | - |
| 摘要字数 | ✅/❌ | XXX | ≤XXX | +XX |

✅ = 合规 / ❌ = 不合规，必须修改

### 3.2 风格对齐检查（需求 B / E）

**需要同刊范文数据。** 针对用户要查的维度，检索范文提取对应特征：

**Step 1 — 范文检索**

**必须加主题关键词过滤**，否则只按 ISSN+高被引排序会返回大量不相关论文：

```bash
curl -s "https://api.openalex.org/works?filter=primary_location.source.issn:{ISSN},type:article,from_publication_date:2020-01-01,title_and_abstract.search:{主题关键词URL编码}&sort=cited_by_count:desc&per_page=15&select=id,doi,title,abstract_inverted_index,cited_by_count,publication_year,keywords,referenced_works_count,open_access,best_oa_location"
```

主题关键词从用户论文标题/关键词中提取（如 `soil salinization remote sensing`）。

精选 10-12 篇：Research Article、高被引、同主题。

**Step 2 — 特征提取（纯 API，按需提取）**

对每篇范文，**只提取用户需要的那几个维度**：

- **Semantic Scholar**（abstract + 引用统计，**最可靠的摘要数据源**）：
```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/DOI:{DOI}?fields=title,abstract,referenceCount,citationCount,tldr,fieldsOfStudy,journal,keywords"
```
> SS 有 rate limit，每篇之间 sleep 1s。OpenAlex 和 CrossRef 的 abstract 字段大量为空。

- **CrossRef**（参考文献完整列表 + 页数）：
```bash
curl -s "https://api.crossref.org/works/{DOI}"
```

- **OpenAlex**（补充元数据）：
```bash
curl -s "https://api.openalex.org/works/doi:{DOI}"
```

- **Europe PMC**（OA 全文，提取更细粒度特征）：
```bash
curl -s "https://www.ebi.ac.uk/europepmc/webservices/rest/search?query=DOI:{DOI}&format=json&resulttype=core"
```
如返回 PMCID 且 isOpenAccess=Y，下载全文提取：章节结构、段落长度、图表数量等。

**Step 3 — 汇总为期刊惯例范围**

将范文特征汇总为统计描述（使用中位数 + IQR，不使用均值±标准差，因为样本量小且易受极端值影响），**标注每项指标的数据完整度**：

```markdown
## 期刊风格画像: {期刊名}
# 基于 X 篇范文

## 检查维度 (X/X篇有数据)
- 范围: XXX-XXX
- 中位数: XXX
- 置信度: 高(≥8篇) / 中(5-7篇) / 低(<5篇)
```

**Step 4 — 对照检查**

| 检查项 | 状态 | 草稿值 | 期刊惯例 | 偏离 | 置信度 |
|--------|------|--------|----------|------|--------|
| 摘要字数 | ⚠️ | XXX | XXX(±XX) | 高 | 高 |

⚠️ = 显著偏离（超出 ±1σ） / ✅ = 匹配

### 3.3 语言风格检查（需求 C）

**需要 OA 全文范文。** 从 3.2 的范文中筛选有全文的，下载后提取语言特征：

- 被动语态比例
- 平均句长
- Hedging 频率（may/might/could/suggest/appear 等）
- 时态使用习惯（Methods 过去时 vs Discussion 现在时）

> 数据来源依赖 OA 全文可用性，通常覆盖率不高。提取前先确认有多少篇范文有 OA 全文，不足 3 篇则标注"数据不足，仅供参考"。

### 3.4 特定部分对标（需求 D）

聚焦用户指定的章节（如摘要/引言/方法/结论），从范文中提取对应部分做对比：
- 结构特征：段落划分、信息组织顺序
- 篇幅特征：该章节占全文比例、平均段长
- 信息密度：引用密度、数据呈现方式

## Phase 4 — 输出整改建议

**输出形式跟着需求走，不改一个字。** 只说"建议修改方向"，不生成替换文字。

按优先级排序：

```
### P0 — 必须修改（不合规项）
1. [检查项] 当前 XXX，要求 XXX → 需要调整的方向

### P1 — 建议修改（显著风格偏离，置信度高）
1. [检查项] 草稿 XXX，期刊惯例 XXX → 建议调整方向

### P2 — 可选优化（轻微偏离或低置信度）
1. [检查项] 草稿 XXX，期刊惯例 XXX → 可选调整方向

### ✅ 已达标项
- [检查项1]、[检查项2]、...
```

## 执行注意事项

### 数据完整度透明

每项风格建议必须标注**数据来源和置信度**。高置信度（≥8篇范文一致）的建议直接给出，低置信度（<5篇或仅元数据推算）的建议标注"仅供参考"。

### 不编造

- Guidelines 中的任何数字必须来自实际抓取的页面
- 风格画像中的统计值必须来自实际 API 数据计算
- 某维度数据不足（<3 篇范文有数据）→ 跳过该维度，标注"数据不足，无法评估"

### 分阶段执行

不要一次性强行跑完。合理拆分：
- 第 1 轮：Phase 0 + Phase 1（期刊信息）
- 第 2 轮：Phase 2 + Phase 3（草稿检查）
- 第 3 轮：Phase 4（建议输出）

每轮结束输出进度摘要，用户确认后继续。

### 降级策略

```
Layer 1 Tavily extract 失败 → Layer 2 Jina Reader → Layer 3 Tavily search+第三方 → Layer 4 手动粘贴
范文 API 全部失败 → 仅做格式合规检查（Guidelines 对照）
某维度数据不足 → 跳过该维度，标注原因
Springer/Nature 正文未提取到 → 直接用 Layer 3 search（跳到第三方聚合站）
```

## 局限性

此工具输出的是**格式合规和风格对齐的修改方向**，不涉及：
- 学术内容正确性
- 实验设计合理性
- 数据分析统计检验
- 研究创新性评价
- **不代写任何文字**，只诊断+给方向

最终学术质量和文字表达由作者本人负责。

## 参考文件

- `references/pitfalls-and-solutions.md` — 实测踩坑记录（Cloudflare拦截、范文检索精准度、数据源选择、统计方法等）
- `references/guidelines-fallback-matrix.md` — 6出版商×4方案的 Guidelines 自动获取测试矩阵，含各层 curl 命令和执行流程
