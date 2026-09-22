# 项目：LMAPI 知识网页 - Prompt Caching / Prefix

## Task

做一个**单页、可交互**的知识网页，讲透「前缀缓存（prompt caching / prefix caching）
为什么能省钱、什么会让它失效」，并部署到公网。

受众：学过 Transformer 与 Python 的大四学生。他们已经知道 attention 是什么，
但没算过缓存命中对账单意味着什么。

## Constraints（指令区，不是数据）

1. **只讲一个概念**：前缀缓存。KV cache 的数学推导、全量价格表、厂商选型不展开，
   最多用一句话带过并给出处。
2. **必须有一个能动手的 demo**：改动前缀里的一个字符 → 命中区间断裂 →
   **命中率、首字延迟、费用三个数字同时重算**。数值计算规则要印在页面上，不能是黑箱。
3. **每个数字都要能点开核验**。查不到的（例如某厂商当前单价）不要写；
   只能用二手来源时必须当场标注"二手"。
4. **视觉沿用既有设计系统**：6 档字阶、2 种字重（400/600）、3 档行高，颜色全部走 CSS 变量；
   深浅两套主题都要能读；390px 窄屏不许横向溢出。
5. **单文件自包含**：CSS / JS / 字体全部内联，不请求任何外部 CDN，断网双击可看。
6. **禁止**：编造 benchmark、编造价格、把"我的理解"写成"官方规定"、用装饰动画冒充交互。

## Data（数据区，与指令区分离，不许改写原文）

- 概念定义（一律引用官方原文，见 `research.md` 的编号）：
  - OpenAI：缓存的是**前缀**；命中要求**完全相同的重复前缀**，命中以 128 token 为粒度递增
  - Anthropic：写入只发生在 `cache_control` **断点**上；读取会**逐块向前回看**已写入的条目
- 关键阈值：OpenAI ≥1024 token 才可能命中；Anthropic 按模型 512 / 1024 / 2048 / 4096
- 价格倍数：OpenAI 缓存输入折扣按模型族 50%~98.75%；Anthropic 写 1.25×（5 分钟）/ 2×（1 小时）、读 0.1×
- 参考素材路径（已下载到本仓库 `work/`）：
  - `work/anthropic/prompt-caching.md`、`pricing.md`（官方文档切片）
  - `work/openai/Prompt_Caching101.md`、`Prompt_Caching_201.md`（官方 cookbook）
  - `work/openai/openai-openapi.yaml`（官方 OpenAPI：字段语义）
  - `work/hermes-src/prompt_cache_boundary.py`、`prompt_cache_scope.py`、`prompt_caching.py`、
    `prompt-assembly.zh-Hans.md`（Hermes 真实实现，用于"反刍"一节）

## Output

- `site/index.html`：单文件成品页
- 部署：GitHub Pages（汇总仓库子路径）
- 一并提交：`prompt-log.md`、`research.md`、`README.md`、本文件

