# journal-fit

AI agent 投稿前期刊适配检查 skill：按目标期刊 Author Guidelines 和同刊范文，检查论文草稿是否适合投稿，并输出可复查的修改方向。

> 中文为主，English version below.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Agent Skill](https://img.shields.io/badge/Agent%20Skill-SKILL.md-green.svg)](SKILL.md)

## API 配置（可选但推荐）

基础检查可以零配置运行。若希望提高 Author Guidelines 自动获取成功率，建议在宿主 agent 环境里配置 Tavily：

```bash
export TAVILY_API_KEY="YOUR_API_KEY"
```

不要把 API key 写进仓库。没有 key 时，skill 会降级使用公开页面、reader/search 结果或用户粘贴的指南内容。

## 适用场景

- 想按目标期刊要求做投稿前检查。
- 想区分“必须改”的格式问题和“建议优化”的风格偏离。
- 想把摘要、引言、结论等章节和同刊同主题论文做对照。
- 想让 AI agent 先收集证据、列出风险，再由作者自己修改。

## 它做什么

- 抽取真实 Author Guidelines 中的字数、摘要、关键词、图表、参考文献和声明要求。
- 解析 `.docx`、`.pdf`、`.md` 或 `.txt` 草稿中的相关特征。
- 用同刊同主题范文对比摘要长度、参考文献数量、章节篇幅等软性惯例。
- 输出 `P0 必须修改`、`P1 建议修改`、`P2 可选优化`、`已达标` 和 `无法评估`。

## 不做什么

- 不代写、不生成替换段落、不预测录用概率。
- 不用出版商通用默认值冒充期刊真实要求。
- 不绕过付费墙、验证码、登录墙或机构权限。

## 工作流程

```text
目标期刊 + 草稿 + 检查范围
  ↓
获取 Author Guidelines
  ↓
解析草稿特征
  ↓
必要时检索同刊同主题范文
  ↓
输出 P0 / P1 / P2 诊断和修改方向
```

## 使用方式

在支持 skills / agent instructions 的 agent 里安装：

```text
请从 GitHub 安装这个 skill，并在投稿前期刊适配检查时优先使用它：
https://github.com/keros68/journal-fit
```

安装后可以这样调用：

```text
使用 $journal-fit 按 Journal of Hydrology 的要求检查这篇论文草稿，只看格式合规和摘要风格。
```

手动安装示例：

```bash
git clone https://github.com/keros68/journal-fit.git ~/.codex/skills/journal-fit
```

没有正式 skill loader 的环境，也可以把 `SKILL.md` 作为 agent instruction 使用。

## 文件结构

- `SKILL.md` - skill 主说明和执行流程。
- `agents/openai.yaml` - 兼容运行时的 UI 元数据。
- `references/` - Author Guidelines 抓取矩阵和实测踩坑记录。

## 已知局限

检查质量取决于期刊页面可访问性、草稿可解析性、同刊范文召回质量和宿主 agent 的联网能力。最终投稿前仍建议人工复核期刊官网、Author Guidelines 和投稿系统提示。

## Attribution and Redistribution

This project is the original journal-fit skill by keros68:

https://github.com/keros68/journal-fit

Released under the MIT License. Forks and redistributed copies should preserve the license text and original repository link.

## English

journal-fit is an AI agent skill for pre-submission journal fit checks. It compares a manuscript draft with real Author Guidelines and same-journal examples, then reports evidence-backed revision directions.

It does not write replacement prose, predict acceptance, or bypass access restrictions.

Quick start:

```text
Install this skill from GitHub and use it for pre-submission journal fit checks:
https://github.com/keros68/journal-fit
```

Call it with:

```text
Use $journal-fit to check this manuscript against the target journal's Author Guidelines.
```

## License

MIT. See [LICENSE](LICENSE).
