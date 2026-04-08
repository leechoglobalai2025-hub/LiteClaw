# 🐾 LiteClaw — 安全优先的 AI 中控系统

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-NVIDIA%20DGX%20Spark-76B900.svg)](https://www.nvidia.com/)
[![Paper](https://img.shields.io/badge/论文-AI时代的开源-orange.svg)](https://leechoglobalai.com/ai时代的开源-公开设计思路、图纸与sop流程图/)

> 🇰🇷 [한국어](README.ko.md) | 🇨🇳 中文 | 🇺🇸 [English](README.md)

**安全 AI 助手 — 零信任架构**

> 从 OpenClaw 的"踩坑"到自主研发，LiteClaw 是一个人、一台机器、一段愤怒中诞生的 AI Agent 中控系统。

---

## 📖 前世今生

### 第一章：与 OpenClaw 的相遇（2026年1月）

2026年1月31日，我安装了当时最火爆的开源 AI Agent 平台 **OpenClaw**（GitHub Stars 145,000+）。我非常喜欢它——直到五天后，一切开始崩塌。

### 第二章：API 账单的噩梦

我首先连接了 **Claude API**。起初一切看似正常——每次对话 $0.03，但费用持续攀升至 $0.10。更令人震惊的是：**input Token 是 output Token 的 50 倍**。这绝不是正常的比例。

于是我切换到了更便宜的 **Gemini API**——我有 Google 赠送的 $300 免费额度，相当于零成本。

### 第三章：被 TPM 限额封禁

连接 Gemini API 的第四天，问题爆发了：

- Telegram 持续报错
- OpenClaw 完全无法使用
- 调阅 Google 后台，发现被 **Gemini API 的 TPM（每分钟 Token 数）限额**了

![Gemini API 后台数据](images/gemini-api-rate-limit.jpg)
*Gemini API 后台：Gemini 3 Pro 的 TPM 达到 1.26M/1M，超出限制*

我询问了 GPT，得到的答案令人震惊：**每分钟 1M Token 的使用量在个人用户中几乎不可能出现，这种情况只有企业做压力测试时才会发生！**

我被封禁了一天，韩国时间下午4点才能解封。

### 第四章：根因分析

解封后，我第一件事就是安装对话压缩插件。但这只撑了一天——**又被限额了**。

于是我深入分析了 OpenClaw 的架构，发现了根本问题：

> **OpenClaw 会持续堆叠所有历史对话记录。**

而我是长上下文使用者和溯因逻辑推理者，天生就是高 Token 消费者。这意味着：**接入任何 API，我都必然被限额，或者需要支付巨额费用。**

### 第五章：LiteClaw 的诞生（2026年2月13日）

我开始寻找替代方案，发现了轻量级产品 **NanoBot**。分析之后，我设计了自己的工程方案。

**2月13日**，我用 Google 的 IDE **Antigravity**，连接 **Claude Opus 4.5**，花费 5 个小时完成了 **LiteClaw v1.0**。

![架构对比分析](images/architecture-comparison.jpg)
*Opus 4.5 对 LiteClaw vs OpenClaw vs NanoBot 的架构级客观分析——这不是人的分析，是 AI 的分析！*

| 维度 | OpenClaw | NanoBot | LiteClaw |
|------|----------|---------|----------|
| 定位 | 全功能 AI Agent 平台 | 超轻量个人助手 | 安全优先的 AI 中控 |
| 代码量 | ~430,000 行 | ~4,000 行 | ~5,000 行（42 文件）|
| 循环依赖 | 存在（大型项目难免）| 少（代码少）| **零** ✅ |
| 密钥保护 | ⚠️ 明文存储 API key 和 session token | 基础环境变量 | ✅ SecretValue 包装器——阻断 str/repr/json/pickle 全路径 |
| Agent 防火墙 | ⚠️ Docker 沙箱（默认不启用）| 无 | ✅ 8条 Shell + 4条工具正则规则，默认启用 |
| 日志脱敏 | ⚠️ 不自动脱敏 | 无 | ✅ 6种模式自动脱敏 |
| 审计引擎 | 内置 security audit 命令 | 无 | ✅ 三阶段审计（pre/exec/post），自动记录 |
| Skill 安全 | ⚠️ 36%有漏洞，76个恶意 Skill（Marketplace 安全事件）| 不适用 | Skills 本地管理，无 Marketplace 风险 |

### 第六章：迁移与进化

随后，LiteClaw 被平移到了我的 **NVIDIA DGX Spark** 上，用 **Claude Code** 持续开发迭代。

![LiteClaw v1 界面](images/liteclaw-v1-ui.jpg)
*第一版 LiteClaw 在 DGX Spark 上的 UI 界面*

### 第七章：AI 编程的突破与论文（2026年3月）

2月末，我在研究 AI 编程控制时，突然想到：**能否用 Claude Code 的 CLI 界面，按照我设计的流程图完成编程？**

**3月28日，三次测试：**

| 测试 | 方法 | 结果 | 原因 |
|------|------|------|------|
| 第一次 | 直接编程 | ❌ 失败 | 缺少施工流程图 |
| 第二次 | 多 Agent 协作 | ❌ 失败 | 自动启动"屎山平移大法" |
| 第三次 | 单 Agent 按部就班 | ✅ 成功 | 一步步按流程执行 |

测试成功后，我同步完成了论文的撰写：

> 📄 **《AI 时代的开源——公开设计思路、图纸与 SOP 流程图》**
>
> 🇨🇳 [中文版](https://leechoglobalai.com/ai时代的开源-公开设计思路、图纸与sop流程图/) | 🇺🇸 [English](https://leechoglobalai.com/open-source-in-the-ai-era-publishing-design-blueprints-sop-flowcharts/) | 🇰🇷 [한국어](https://leechoglobalai.com/ai-시대의-오픈소스-설계-사상·도면·sop-흐름도의-공개/)

**3月31日，多语言 AB 测试：**

基于论文中提出的方法论，用韩语、中文、英语三种语言的 Task 文件进行 AB 测试——**全部通过**。无论使用哪种语言，都可以完成 LiteClaw 的 AI 编程。

![LiteClaw Desktop 现状](images/liteclaw-desktop.jpg)
*当前版本的 LiteClaw Desktop —— 支持多 LLM 提供商连接*

### 第八章：现在

LiteClaw 已从最初 5,000 行的轻量工具，进化成 **8.3GB 的超级 AI Agent 中央控制器**。

> 🎯 **这证明了我的论文理论的方法论是正确的。这也是 AI 时代开源的先河——开源可以控制 95% AI 编程正确路径的 Task 文件！**

---

## ✨ 核心特性

- 🔐 **零信任安全架构** — SecretValue 包装器，API 密钥永不明文显示
- 🏗️ **L0-L8 八层架构** — 严格单向依赖，零循环依赖
- 🛡️ **三阶段审计引擎** — pre/exec/post 自动记录
- 🔒 **6种模式自动日志脱敏**
- 🤖 **多 LLM 支持** — Google Gemini、OpenAI、Anthropic、DeepSeek、xAI、Qwen、Kimi 等
- 🖥️ **本地 LLM 支持** — 通过 vLLM 连接本地模型
- 🌐 **多语言界面** — 中文、英文、韩文

---

## 🚀 如何使用（开源 Markdown 文件）

### 我们开源的是什么？

**不是代码，不需要 Python 运行。** 我们开源的是 **Markdown 格式的 Task 文件**——设计思路、架构图纸与 SOP 流程图。

### 使用方法

1. **下载**你擅长语言版本的三个 Markdown 文件（中文/英文/韩文）
2. **导入**任何 IDE（如 VS Code、Cursor）或 **Claude Code**
3. **连接** Claude Opus 4.6 模型
4. **按部就班**地按照 Markdown 中的流程图指示，让 AI 一步步完成编程

✅ **这样你就可以 100% 用 AI 编程制作出 LiteClaw 的基础架构。**

### 为什么这种方式是安全的？

> 传统开源 = 下载别人的代码 → 存在供应链投毒、恶意代码、后门等风险

> **LiteClaw 开源 = 下载设计图纸 → AI 按图纸在你的机器上从零构建 → 绝对安全**

- ✅ 没有可执行代码，不存在供应链投毒风险
- ✅ 没有第三方依赖注入的可能
- ✅ AI 在你本地环境从零生成，你可以审查每一行代码
- ✅ 允许任意二次开发，自由定制

> 详细论述请参阅论文：[《AI 时代的开源——公开设计思路、图纸与 SOP 流程图》](https://leechoglobalai.com/ai时代的开源-公开设计思路、图纸与sop流程图/)

---

## 🛡️ LiteClaw 最强之处：架构与系统安全

LiteClaw 最强大的部分，恰恰是当今"脚本小子"最不擅长的——**架构设计与系统安全**。

以下是 **Claude Opus 4.5 对 LiteClaw 的客观 AI 分析**（非人类主观评价）：

| 安全特性 | OpenClaw | NanoBot | LiteClaw |
|----------|----------|---------|----------|
| 密钥保护 | ⚠️ 明文存储 | 基础环境变量 | ✅ SecretValue 包装器 |
| Agent 防火墙 | ⚠️ Docker（默认不启用）| 无 | ✅ 8+4 条正则规则，默认启用 |
| 日志脱敏 | ⚠️ 无 | 无 | ✅ 6种模式自动脱敏 |
| 审计引擎 | 命令式（配置检查）| 无 | ✅ 三阶段自动审计 |
| Prompt 注入防护 | ⚠️ 审计发现无效 | 无 | Firewall 有工具参数扫描 |
| Skill 安全 | ⚠️ 36%有漏洞 | 不适用 | ✅ 本地管理，无 Marketplace 风险 |

---

## 🛠️ 开发时间线

| 日期 | 事件 |
|------|------|
| 2026.01.31 | 安装 OpenClaw，开始 AI Agent 之旅 |
| 2026.02.05 | 遭遇 API 账单异常与 TPM 限额问题 |
| 2026.02.13 | **LiteClaw v1.0 诞生** — 用 Antigravity + Claude Opus 4.5，5小时完成 |
| 2026.02.14 | 迁移至 NVIDIA DGX Spark，开始用 Claude Code 持续开发 |
| 2026.03.28 | CLI 单 Agent 编程测试成功 + **完成论文** |
| 2026.03.31 | 多语言（韩/中/英）AI 编程 AB 测试全部通过 |
| 2026.04.01 | LiteClaw 进化至 8.3GB 超级 AI Agent 中控系统 |

---

## 📄 论文

**《AI 时代的开源——公开设计思路、图纸与 SOP 流程图》**

先完成实验测试，再生成论文，最后用论文方法论进行多语言 AB 测试验证。

- 🇨🇳 [中文版](https://leechoglobalai.com/ai时代的开源-公开设计思路、图纸与sop流程图/)
- 🇺🇸 [English Version](https://leechoglobalai.com/open-source-in-the-ai-era-publishing-design-blueprints-sop-flowcharts/)
- 🇰🇷 [한국어 버전](https://leechoglobalai.com/ai-시대의-오픈소스-설계-사상·도면·sop-흐름도의-공개/)

---

## 📜 许可证

本项目采用 **Apache License 2.0** 许可证。详情请参阅 [LICENSE](LICENSE) 文件。

允许所有下载和使用者做任意二次开发。

---

## 📬 联系方式

- 🏢 **이조글로벌인공지능연구소** (LEECHO Global AI Research Lab)
- 🌐 官网：[https://www.leechoglobalai.com/](https://www.leechoglobalai.com/)
- 🐙 GitHub：[@leechoglobalai](https://github.com/leechoglobalai2025-hub)
- 📍 地址：韩国仁川
