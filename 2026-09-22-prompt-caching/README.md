# 前缀缓存：为什么改一个字，整段就白算了

大模型与自然语言处理 · 郑新政 · 江苏师范大学 · 2026-09-22

子任务 **C · Prompt Caching / Prefix**（LMAPI 知识网页 · 第 2 次课实验）

## 在线预览

https://impossible87.github.io/assignments/2026-09-22-prompt-caching/

> 打不开时请用仓库里的 `site/index.html` 直接打开：这是一个**自包含单文件**，样式、脚本、字体、图表数据都在文件内部，不联网也能完整显示。

## 这是什么

这一页回答一个问题：**同一段长提示词，为什么改动的位置不同，省下的钱差好几倍？**

三条结论：

1. **缓存键是"从头发起的一整段"**。命中要求前缀完全一致（OpenAI 还要求落在同一台机器上）；改动越靠前，失效越多。
2. **能不能省，由两个数字决定**：命中按 **128 token 粒度**递增；提示词低于 **1024 token** 一定不命中（Anthropic 侧按模型 512/1024/2048/4096）。
3. **前缀稳定是"运行时"的事**：靠请求结构、断点位置、TTL 与路由键，而不是把提示词写得更漂亮。

页面里有一个可编辑的四段式提示词演示（系统指令 / 工具定义 / 参考文档 / 用户问题）：
改末尾问题仍然命中、改文档中间掉一半、改系统指令或塞时间戳直接归零；
命中率、两家口径的两次请求费用、首字延迟降幅同时重算，**计算规则印在页面上**，不是黑箱。

## 仓库结构

```text
.
├── site/index.html   # 成品页（单文件，双击即可打开）
├── AGENTS.md         # 给 Agent 的规格：Task / Constraints / Data / Output 分区
├── prompt-log.md     # Prompt 迭代日志：规格 → 失败现象 → 改了哪条规格（含结论）
├── research.md       # 一手来源记录：硬数字、出处、取材通道说明
├── brief.md          # 作业要求拆解与评分点对应表
├── work/             # 中间产物：官方文档切片、Hermes 源码副本、页面源码分段
└── README.md         # 本文件
```

## 数据来源

- **OpenAI Cookbook**（官方一手）：*Prompt Caching 101 / 201* —— 1024 token 门槛、128 token 命中粒度、各模型缓存折扣对照表（50%~98.75%）、TTFT 实测（1024 token 约 7%、150k 约 67%）、`prompt_cache_key` 路由说明。抓取 2026-09-22
- **OpenAI OpenAPI 规范**（官方一手）：`prompt_cache_key` / `prompt_cache_retention` / `prompt_cache_options` 字段语义。抓取 2026-09-22
- **Anthropic 官方文档**：*Prompt caching*（断点写入、20 块回看、按模型最小长度、5 分钟／1 小时 TTL、使用字段）与 *Pricing*（写入 1.25×／2×、读取 0.1×、读 1 次回本）。抓取 2026-09-22
- **Nous Research / hermes-agent**（官方仓库源码）：`agent/prompt_cache_boundary.py`、`agent/prompt_cache_scope.py`、`agent/prompt_caching.py`、`website/docs/developer-guide/prompt-assembly.md`，用于页面的"反刍"一节。抓取 2026-09-22

取材通道说明：`platform.openai.com` 与 `developers.openai.com` 在本机网络返回 403，`platform.claude.com` 的文档页被重定向到区域不可用页面；因此 OpenAI 侧改用**官方仓库与官方规范**，Anthropic 侧改用官方 `llms-full.txt` 整包文档。原始素材副本都在 `work/` 目录里。

## 技术说明

- 单文件 HTML，无外部依赖：不请求任何 CDN、字体或图片，离线可用
- 图表为内联 SVG（首字延迟趋势、价格倍数条），交互为原生 JavaScript
- 字体为内嵌子集（思源黑体 / Inter / JetBrains Mono / Space Grotesk，均为 OFL 开源许可）
- 响应式：手机、平板、桌面均可阅读；支持深浅色与打印导出 PDF
- 演示中的 token 换算是估算口径（中文 1 字 ≈ 0.67 token、英文 4 字符 ≈ 1 token），延迟曲线在官方两个实测点之间做对数插值，页面上都已标注

## AI 使用声明

本文的选题、结构、结论、数据核对与图表设计由作者完成；资料检索、代码实现与页面迭代过程中使用了 AI 辅助。
按课程要求，过程材料一并提交：`AGENTS.md` 是本次给 Agent 的规格文件，`prompt-log.md` 记录了规格 → 失败 → 迭代的真实过程（5 轮规格改动 + 2 个实现层踩坑）。文中数据与结论均可在参考来源中查证。

## 许可

内容著作权归作者所有。
