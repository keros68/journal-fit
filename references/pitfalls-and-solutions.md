# 实测踩坑与解决方案

基于 2026-06-27 STOTEN 模拟测试（v3.0 首次完整跑通）总结。

## 1. Elsevier/ScienceDirect Guidelines 页面被拦截

**问题：** curl 和 headless browser 都拿不到 `elsevier.com/.../guide-for-authors`，Cloudflare 返回 403/拦截页。ScienceDirect 也会拦 IP。

**解决方案优先级：**
1. Tavily extract API（需配置 TAVILY_API_KEY）— 能绕过 Cloudflare，最可靠
2. Google Cache `cache:URL` 或搜索引擎快照 — 有时能拿到缓存版
3. 降级：让用户复制粘贴 Guidelines 核心部分

**注意：** 不要把"Elsevier 拦截"当硬性约束写死。这是环境相关的问题，Tavily 配好就能绕过。

## 2. 范文检索的精准度问题

**问题：** 仅用 ISSN 检索 STOTEN，返回的全是高被引但主题不相关的论文（COVID、空气质量），无法做同主题风格对标。

**解决：** 在 OpenAlex 检索中加 `title_and_abstract.search:{主题关键词}` 过滤：

```
filter=primary_location.source.issn:{ISSN},type:article,
       from_publication_date:2020-01-01,
       title_and_abstract.search:soil%20salinization%20remote%20sensing
```

**示例：** STOTEN 检索 `soil salinization + remote sensing` → 返回 11 篇精准同主题论文，覆盖参考。

## 3. Abstract 数据源选择

**问题：** OpenAlex `abstract_inverted_index` 字段大量为空（11篇中只有2篇有）。CrossRef `abstract` 字段也大多为空（Elsevier 不通过 API 返回摘要）。

**解决：** Semantic Scholar 是最可靠的 abstract 数据源。11 篇范文全部通过 Semantic Scholar 拿到了完整摘要。

**API 调用（有 rate limit，需加延迟）：**
```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/DOI:{DOI}?fields=title,abstract,referenceCount,citationCount"
```

> SS 有 rate limit，每篇之间 sleep 1s。11 篇约需 20s。

## 4. CrossRef page 字段对 Elsevier 期刊无用

**问题：** CrossRef 返回的 `page` 字段对 Elsevier 期刊实际是文章编号（如 `142030`），不是页码范围。无法用于估算全文篇幅。

**解决：** 不要依赖 CrossRef page 字段估算 Elsevier 期刊论文页数。如需篇幅特征，只能通过 OA 全文下载后用 PDF 解析。

## 5. 分阶段执行的实际情况

测试中四个 Phase 在一轮对话内跑完（约 6 个工具调用），因为：
- 模拟草稿为简短 md 文件
- 范文 API 全部成功
- 检查维度有限（只查参考文献数 + 摘要字数）

**实际使用中建议拆分的关键场景：**
- 用户选 E（全面检查）时，可能需要先跑 Phase 1+2 范文检索，第二轮再跑 Phase 3+4
- 如果范文超过 15 篇，特征提取阶段拆两轮
- 如果需要 OA 全文下载和解析（语言风格检查），单独一轮

## 6. 风格对齐的统计方法

使用中位数 + IQR（四分位距）而非均值±标准差，因为：
- 样本量小（通常 10-15 篇）
- 容易受极端值影响（如有篇论文 145 篇参考文献）
- IQR 更稳健

**偏离判断标准：**
- 草稿值 < Q1 或 > Q3 → 标记 ⚠️
- 草稿值 < Q1 - 1.5×IQR 或 > Q3 + 1.5×IQR → 标记 ⚠️ 严重偏离

## 7. Tavily 配置与跨出版商效果（2026-06-27 实测，完整版）

**Tavily API key 已配置**：写入 `~/.claude/settings.json` 的 `env.TAVILY_API_KEY`。extract skill 脚本和直接 curl 都能用。

**跨出版商 Guidelines 抓取效果（6 家出版商完整测试）：**

| 期刊 | 出版商 | Tavily extract | Jina Reader | Tavily search→第三方 |
|------|--------|---------------|-------------|---------------------|
| Science of the Total Environment | Elsevier | 88K chars ✅ | 62K ✅ | — |
| Water Research | Elsevier | 84K chars ✅ | — | — |
| J. Hazardous Materials | Elsevier | 3.4K chars ✅ | — | — |
| MDPI Water | MDPI | 108K chars ✅ | 108K ✅ | — |
| Nature Climate Change | Springer Nature | 13K chars ⚠️ | 4K ⚠️ | ✅ (manusights) |
| Global Change Biology | Wiley | ❌ | ❌ | ✅ (manusights 33K) |
| ES&T | ACS | ❌ | ❌ | ✅ (manusights 22K) |
| Arid Land Res. & Mgmt. | Taylor & Francis | ❌ | ❌ | ✅ (scispace 17K) |

**三层 fallback 策略已写入 skill v3.2：**
- Layer 1: Tavily extract → 覆盖 Elsevier/MDPI
- Layer 2: Jina Reader（免费备选）→ 覆盖范围相同
- Layer 3: Tavily search → 第三方聚合站（manusights.com/scispace.com）→ 覆盖 Wiley/ACS/T&F
- Layer 4: 手动粘贴（兜底，实测中不需要）

**详细测试数据和 curl 命令见 `references/guidelines-fallback-matrix.md`。**

**结论：**
- Tavily 对 Elsevier/MDPI 系效果优秀
- Springer/Nature 系 Tavily 能返回页面但正文提取不完整，需走 Layer 3
- Wiley/ACS/T&F 被 Cloudflare 拦截，但 Layer 3 第三方聚合站全覆盖
- **三层 fallback 可实现 6/6 出版商全自动获取，无需用户手动粘贴**

## 8. 没有真实 Guidelines 时格式检查的危害（关键教训）

在 STOTEN 模拟测试中，第一轮用"Elsevier 通用要求"替代真实 Guidelines，导致 **13 项格式检查中 6 项出错**：

| 检查项 | 通用默认值 | 真实 Guidelines | 后果 |
|--------|-----------|----------------|------|
| Graphical Abstract | 可选 | 必需 | 漏报 P0 |
| 参考文献数限制 | 无限制 | 建议 ≤ 50 篇 | 误判"严重偏离" |
| 参考文献格式(投稿) | Vancouver | 任意格式 | 误报不合规 |
| 参考文献风格(最终) | Numbered | Author-Year | 方向搞反 |
| 摘要上限 | 250 词 | 300 词 | 临界值误判 |
| 关键词数 | ≤ 6 | 1-7 | 边界值错误 |

这意味着上一轮模拟测试报告中的 P0/P1 建议有方向性错误。**没有真实 Guidelines 的格式检查不仅无用，而且有害——会产生误导性建议。**

Skill v3.1 已在 Guidelines Checklist 部分加入"禁止猜测"规则和反例。
