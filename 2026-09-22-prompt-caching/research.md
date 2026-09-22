# Prompt Caching / Prefix 调研（截至 2026-09-22）

> 取材原则：只用一手来源（厂商官方文档 / 官方 cookbook / 官方 OpenAPI / 官方仓库源码）。
> 每条结论后面标 `[编号]`，编号对应文末参考列表。

## 0. 取材通道说明（先记下来，避免以后重复踩）

- `platform.openai.com` 与 `developers.openai.com` 在当前网络返回 **403**（拿不到官方文档页）。
  可用的一手替代通道：**`openai/openai-openapi` 官方 OpenAPI 规范**（字段语义）与
  **`openai/openai-cookbook` 官方教程 notebook**（Prompt Caching 101 / 201）。[1][2][3]
- `docs.claude.com` / `platform.claude.com` 的**文档页**在当前网络被重定向到
  "app-unavailable-in-region"，但 **`https://platform.claude.com/llms-full.txt` 可以整包下载**
  （35.7 MB，包含全部官方文档正文）。本次的 Anthropic 结论都从这份整包文档里按 `url:` 前言切出来。[4][5]
- Hermes 的 `prompt_builder.py` 等源码：`raw.githubusercontent.com` 不可达，
  走 **`api.github.com` 的 git blobs 接口**取到（base64 解码）。[6]

## 1. 机制：缓存的是"前缀"，不是"内容"

- **OpenAI**：命中要求 **完全相同的重复前缀**（exact repeated prefix match），
  且命中以 **128 token 为粒度**递增；请求前缀里的 messages、images、audio、
  tool definitions、structured output schema 都可缓存。[2]
- **Anthropic**：**写入只发生在断点上**——给某块打 `cache_control` 才会写入一条记录；
  **读取时先算断点处的前缀哈希**，没有则**逐块向前回看**，找"之前请求已经写入过的条目"。
  关键区别：回看找的是**既往写入**，不是"稳定内容"。[4]
- 省下来的东西：Transformer 前向传播里 **K/V 张量的重复预填充**。命中时这段计算被跳过，
  所以延迟随"输出长度"增长，而不是随"对话总长度"增长。[2]

## 2. 门槛与粒度（决定"够不够格被缓存"）

| 厂商 | 最小可缓存长度 | 粒度 / 上限 |
|---|---|---|
| OpenAI | **1024 token**（低于 1024 一定不命中，`cached_tokens` 恒为 0） | 命中按 **128 token** 递增 [1][2] |
| Anthropic | 按模型 **512 / 1024 / 2048 / 4096** token；不够长会被**静默跳过**，不报错 | 显式断点**最多 4 个**；回看窗口 **20 个块** [4] |

Anthropic 各模型最小长度（官方原文）[4]：
512 = Fable 5.1 / Mythos 5.1 / Opus 5 等；1024 = Opus 4.8 / Sonnet 5 / Sonnet 4.6 等；
2048 = Mythos Preview / Opus 4.7 / Haiku 3.5；4096 = Opus 4.6 / Opus 4.5 / Haiku 4.5。

## 3. 价格：省多少、什么时候回本

**Anthropic**（官方定价页原文）[5]：

| 计费项 | 相对基础输入价 | 说明 |
|---|---|---|
| 5 分钟缓存写入 | **1.25×** | 缓存有效 5 分钟 |
| 1 小时缓存写入 | **2×** | 缓存有效 1 小时 |
| 缓存读取（命中） | **0.1×** | Fable 5.1 / Mythos 5.1 为 0.025× |

- 回本点：**5 分钟档读 1 次就回本；1 小时档要读 2 次**。[5]
- TTL 从**请求开始**计算，被命中时免费刷新；如果一个响应流式输出了 4 分钟，
  下一次请求必须在它结束后约 1 分钟内发出，才还在 5 分钟窗口里。[4]

**OpenAI**（官方 cookbook 表格）[2]：

| 模型 | 输入（每 1M） | 缓存输入（每 1M） | 折扣 |
|---|---|---|---|
| gpt-4o | $2.50 | $1.25 | 50% |
| gpt-4.1 | $2.00 | $0.50 | 75% |
| gpt-5-nano | $0.05 | $0.005 | 90% |
| gpt-5.2 | $1.75 | $0.175 | 90% |
| gpt-realtime (audio) | $32.00 | $0.40 | 98.75% |

- 延迟：官方实测 **1024 token 时约快 7%**，**150k+ token 时快 67%**，最高可省约 80% 的 TTFT。[2]
- 官方给的算例：900 token 的 prompt **永远不命中**；拉到 1100 token 且命中率 50% 时省约 33% 的 token 成本。[2]

## 4. 什么会让缓存失效（"改一个字就 miss"的机制）

1. **前缀哈希是累积的**：断点前的任何一块变了，哈希就变，下一次请求拿不到那条记录。[4]
2. **时间戳/随机 ID 放进前缀**是最经典的坑：哈希每次都不同，于是**每次都写、从来不读**，
   钱花了、缓存没省。[4]
3. **工具定义与它们的顺序也算前缀**：OpenAI 明确"tools/schemas 注入在 developer 指令之前，
   改动它们会失效"；Anthropic 在对话中途往 tools 数组**头部**插入工具同样会 miss，
   官方建议用 `defer_loading` 追加而不是改头部。[2][4]
4. **同一前缀还要落到同一台机器**：OpenAI 按 prompt **前约 256 个 token 的哈希**做路由，
   所以提供了 `prompt_cache_key` 来提高"同一前缀落同一台机器"的概率。[2][3]
5. **并发**：Anthropic 说明缓存条目要在**首个响应开始之后**才可用，
   想靠并发请求命中缓存，得先等第一个响应。[4]

## 5. 两家 API 形式的不同（实验目标第 4 条）

| 维度 | OpenAI | Anthropic |
|---|---|---|
| 开启方式 | **自动**：超过 1024 token 自动生效，无需改请求 | **显式**：在请求体顶层加 `cache_control`（自动缓存）或在具体块上打断点 |
| 断点标记 | 无块级标记；用 `prompt_cache_key` 影响路由 | 块级 `cache_control: {"type":"ephemeral"}`，可加 `"ttl":"1h"` |
| 有效期 | 内存缓存默认；`prompt_cache_retention: 24h` 或 `prompt_cache_options.ttl` 开启延长 | 默认 5 分钟，命中免费刷新；可选 1 小时（写入 2×）[4] |
| 观测字段 | `usage.prompt_tokens_details.cached_tokens` | `usage.cache_creation_input_tokens` / `cache_read_input_tokens` [4] |
| 语义差异 | 命中 = 前缀匹配 + 128 token 粒度 + 同机路由 | 命中 = 断点哈希 + 20 块回看 + 最多 4 断点 |

OpenAI 侧字段的一手依据来自官方 OpenAPI 规范：`prompt_cache_key`
（"Used by OpenAI to cache responses for similar requests to optimize your cache hit rates"）、
`prompt_cache_retention`（`in_memory` | `24h`，已标记 deprecated，代之以
`prompt_cache_options.ttl`），另有 `prompt_cache_breakpoint` / `prompt_cache_diagnostics`。[3]

## 6. 反刍：Hermes Agent 自己怎么处理这件事（真实源码）

> 任务书的"反刍"阶段要求读 Hermes 的 `prompt_builder.py`。实际取到的相关文件：
> `prompt_builder.py`、`prompt_caching.py`、`prompt_cache_boundary.py`、`prompt_cache_scope.py`
> 与官方文档 `developer-guide/prompt-assembly.md`。[6]

1. **缓存边界由"构造方"显式登记**，而不是在请求时用分隔符去猜：
   `prompt_cache_boundary.register_stable_prefix()` 记录本次拼装出的稳定前缀，
   `find_stable_prefix()` 只认"已登记前缀 + 非空白尾巴"（尾巴全空白会让 Anthropic
   返回 HTTP 400）。源码注释明确写了**为什么不用标记串解析**：标记可能合法地出现在
   技能正文里，任何分隔符启发式都会"缩小缓存前缀或把可变内容吸进来"。[6]
2. **默认放 4 个断点**：稳定系统前缀、稳定边界处、以及**最近 3 条消息**；
   所有断点共用一个 TTL。[6]
3. **TTL 按"谁在驱动会话"自动选**：`AUTO_CACHE_TTL = "auto"`，
   机器驱动的来源（`subagent` / `cron` / `oneshot` / `webhook` / `api` / `batch` …）
   与人工对话的节奏不同，5 分钟档对机器节奏会频繁过期。[6]
4. **缓存作用域要"跨轮次轮换仍然稳定"**：`prompt_cache_scope.py` 把物理
   `session_id` 映射到**压缩谱系的根**，让压缩产生的换 id 不把缓存打到新桶里；
   托管方按响应粒度发 id 时用 `gateway_session_key` 声明会话，并哈希成 `gwk_<sha256[:24]>`。[6]
5. **官方组装顺序把易变项放到最后**：`prompt-assembly.md` 列出已缓存系统 prompt 的分层顺序，
   第 9 层才是"时间戳 / 可选会话 ID"，并说明"这种分离使稳定前缀保持稳定，从而有效缓存"。[6]
6. **临时覆盖层不进入缓存前缀**：`HERMES_EPHEMERAL_SYSTEM_PROMPT`、prefill 消息这类
   轮次级指令被刻意排除在已缓存前缀之外。[6]

> 对照结论（写进页面的"反刍"一节）：教科书式规则是"静态在前、动态在后"，
> 而一个真实 agent 要在**代码层面**保证这件事：**登记的边界 + 固定断点数 + 按驱动方选 TTL +
> 跨压缩稳定的作用域**。这四条正好对应上面 1-4 点。

## 参考

1. OpenAI Cookbook. *Prompt Caching 101*. `https://github.com/openai/openai-cookbook/blob/main/examples/Prompt_Caching101.ipynb`（2026-09-22 取）
2. OpenAI Cookbook. *Prompt Caching 201*. `https://github.com/openai/openai-cookbook/blob/main/examples/Prompt_Caching_201.ipynb`（同上）
3. OpenAI. *OpenAPI specification*. `https://github.com/openai/openai-openapi/blob/main/openapi.yaml`（字段定义与描述）
4. Anthropic. *Prompt caching*. `https://platform.claude.com/docs/en/build-with-claude/prompt-caching`（经 `llms-full.txt` 整包取得）
5. Anthropic. *Pricing*. `https://platform.claude.com/docs/en/about-claude/pricing`（同上）
6. Nous Research. *hermes-agent* 源码：`agent/prompt_cache_boundary.py`、`agent/prompt_cache_scope.py`、
   `agent/prompt_caching.py`、`agent/prompt_builder.py`、`website/docs/developer-guide/prompt-assembly.md`
   （经 GitHub git-blobs API 取得，本地副本在 `work/`）
