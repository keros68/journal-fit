# journal-fit

AI agent 投稿前期刊适配检查 skill：根据目标期刊 Author Guidelines 和同刊同主题范文，对论文草稿做格式合规、风格对齐、语言特征和特定章节诊断，并输出可复查的修改方向。

它的目标不是代写论文，也不是预测录用概率，而是在投稿前帮作者先看清楚：哪些地方不符合目标期刊硬性要求，哪些地方和同刊惯例偏离，哪些判断因为证据不足不能下结论。

> 中文为主，English version below.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Agent%20Skill-SKILL.md-green.svg)](SKILL.md)

## 适用场景

- 已经有目标期刊，想按 Author Guidelines 做投稿前格式合规检查。
- 论文基本写完，想知道摘要、关键词、参考文献、图表、声明和章节结构是否明显偏离目标期刊要求。
- 想把自己的摘要、引言、结论或其他章节，和同刊同主题文章的写法做一个数据化对照。
- 需要区分“必须修改”的期刊硬性要求和“建议优化”的同刊风格惯例。
- 想让 AI agent 先做证据收集、规则抽取和诊断排序，再由作者自己决定怎么改。

## 它做什么

- 先问清楚目标期刊、草稿位置和检查范围，不默认把所有检查都跑一遍。
- 从真实 Author Guidelines 中抽取硬性要求，例如字数、摘要格式、关键词、Highlights、Graphical Abstract、参考文献、图表和声明。
- 在抓不到官方页面时，按降级策略使用 reader、搜索结果、第三方聚合页或用户粘贴内容，并标注来源限制。
- 解析 `.docx`、`.pdf`、`.md` 或 `.txt` 草稿中和当前检查范围相关的特征。
- 用同刊同主题范文做软性风格画像，例如摘要长度、参考文献数量、章节篇幅和引用密度。
- 对小样本风格指标使用中位数和 IQR，标注样本量、数据完整度和置信度。
- 输出 `P0 必须修改`、`P1 建议修改`、`P2 可选优化`、`已达标` 和 `无法评估`。

## 不做什么

- 不生成替换段落，不代写摘要、引言、结论或 cover letter。
- 不预测录用概率，不把格式适配包装成学术质量评价。
- 不用出版商通用默认值替代真实 Author Guidelines。
- 不把第三方聚合站或搜索摘要当成最终官方要求。
- 不绕过付费墙、验证码、登录墙或机构权限。
- 不承诺穷尽所有同刊范文；样本不足时会标注低置信度或无法评估。

## 工作流程

```text
目标期刊 + 论文草稿 + 检查范围
  ↓
确认只查格式合规 / 风格对齐 / 语言风格 / 特定章节 / 投稿准备度
  ↓
获取真实 Author Guidelines，并记录来源可靠性
  ↓
解析草稿中和当前范围相关的特征
  ↓
如需风格对齐，检索同刊同主题范文并提取可用指标
  ↓
按 P0 / P1 / P2 输出证据、风险和修改方向
```

## 使用方式

在支持 skills / agent instructions 的 agent 里，可以直接发送：

```text
请从 GitHub 安装这个 skill，并在之后需要按目标期刊要求检查论文、做投稿前适配诊断或同刊风格对齐时优先使用它：
https://github.com/keros68/journal-fit
```

安装后，重启或新开 agent 窗口测试：

```text
使用 $journal-fit 按 Journal of Hydrology 的要求检查这篇论文草稿，只看格式合规和摘要风格对齐。
```

如果 agent 不能自动安装 GitHub skill，可以手动 clone 到它的 skills 目录：

```bash
# Claude Code
git clone https://github.com/keros68/journal-fit.git \
  ~/.claude/skills/journal-fit

# Codex
git clone https://github.com/keros68/journal-fit.git \
  ~/.codex/skills/journal-fit

# 通用 agent 约定目录
git clone https://github.com/keros68/journal-fit.git \
  ~/.agents/skills/journal-fit

# 项目局部使用
git clone https://github.com/keros68/journal-fit.git \
  ./.agents/skills/journal-fit
```

没有正式 skill loader 的环境，也可以把 `SKILL.md` 作为 agent instruction 使用；需要了解 Author Guidelines 抓取策略或实测踩坑时，再附带 `references/`。

## 可选配置

基础流程不要求 API key。常见公开数据源包括 OpenAlex、Semantic Scholar、CrossRef 和 Europe PMC。

如果宿主环境支持 Tavily，可选配置：

```bash
export TAVILY_API_KEY="YOUR_API_KEY"
```

如果需要解析 Word 或 PDF 草稿，可在 Python 环境中安装：

```bash
pip install python-docx pymupdf
```

没有这些依赖时，agent 应该降级处理可读文本，而不是假装已经完成完整解析。

## 输出内容

常见输出包括：

- 投稿前 readiness call；
- Author Guidelines 来源和可复查证据；
- 草稿当前值和目标期刊要求的对照表；
- 同刊同主题范文的样本量、数据完整度、中位数和 IQR；
- P0/P1/P2 修改方向；
- 已达标项目；
- 无法评估项目及原因；
- 第三方来源或低置信度数据的风险提示。

## 文件结构

- `SKILL.md` - skill 主说明、触发规则和执行流程。
- `agents/openai.yaml` - 兼容运行时的 UI 元数据。
- `references/guidelines-fallback-matrix.md` - 2026-06-27 Author Guidelines 抓取方案实测矩阵。
- `references/pitfalls-and-solutions.md` - 实测踩坑记录和修正原则。

## 已知局限

检查质量取决于目标期刊页面可访问性、草稿可解析性、同刊范文召回质量、OA 全文可用性和宿主 agent 的联网能力。第三方聚合页可能滞后，最终投稿前仍建议人工复核期刊官网、Author Guidelines、投稿系统提示、文章类型、图表要求、声明要求和最新政策。

## Attribution and Redistribution

This project is the original journal-fit skill by keros68:

https://github.com/keros68/journal-fit

The project is released under the MIT License. Redistribution, forks, modified versions, and repackaged copies must preserve the copyright notice and license text. Please do not present modified copies as the original project or imply endorsement by the original author.

## English

journal-fit is a portable AI agent skill for pre-submission journal fit diagnostics. It helps an agent compare a manuscript draft against real Author Guidelines and same-journal, same-topic exemplars, then produce evidence-backed revision directions.

It is diagnostic only. It does not write replacement prose, predict acceptance probability, bypass access restrictions, or treat generic publisher defaults as journal-specific requirements.

Typical checks include format compliance, same-journal style alignment, language-style diagnostics when enough OA full text is available, and section-specific comparisons for areas such as the abstract, introduction, or conclusion.

Quick start:

```text
Install this skill from GitHub and use it for pre-submission journal fit checks:
https://github.com/keros68/journal-fit
```

After installation, restart or open a new agent window and call:

```text
Use $journal-fit to check this manuscript draft against the target journal's Author Guidelines and same-journal conventions.
```

The skill reports P0 must-fix issues, P1 recommended changes, P2 optional improvements, passed items, and items that cannot be assessed from available evidence.

## License

MIT. See [LICENSE](LICENSE).
