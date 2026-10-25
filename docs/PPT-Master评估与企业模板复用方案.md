# PPT Master 评估与企业模板复用方案

> 文档版本：v1.0 ｜ 整理日期：2026-10-26 ｜ 配套文档：`GitHub协作规则.md`
> 评估对象：[hugohe3/ppt-master](https://github.com/hugohe3/ppt-master)（v6.7.3，MIT License）
> 本文依据 2026-10-24 时点的源码（102 个脚本、全部工作流文档）与社区公开资料整理。

---

## 1. 执行摘要

PPT Master 是一套运行在 AI Agent 工具（Claude Code、Codex CLI、Cursor 等）里的 Python 技能包（skill），通过 SVG → DrawingML 管线把文档或主题转化为**原生可编辑的 .pptx**：真母版、真版式继承、原生形状、数据驱动图表与表格，而不是扁平贴图。项目约 67k stars / 5.3k forks，MIT 协议，迭代活跃，是当前 PPT 生成赛道中对"PowerPoint 原生对象模型"还原最深的开源项目。

对企业而言，它最有价值的不是"一键生成 PPT"，而是**模板资产化能力**：把企业已有的 `.pptx` 模板、VI 手册提炼为可复用的 Brand / Style / Layout / Deck 模板工作区，此后每一份新 PPT 都能自动继承企业母版与版式。本文第 3 章给出与本地 Ollama / vLLM + Qwen3:8B 配合的应用环境方案，第 5 章给出企业模板复用的三条落地路径。

**结论先行**：值得引入。建议按"先回填救急 → 再模板资产化 → 后规模化"三步走；模型接入首选 Claude Code + Ollama（Anthropic 兼容 API）的组合，团队共享 GPU 时改用 vLLM + 代理桥接；同时要对 Qwen3:8B 的能力边界建立诚实预期（见 3.5 节）。

---

## 2. 项目评估

### 2.1 基本盘

| 维度 | 情况 |
|---|---|
| 项目定位 | 在 AI Agent 工具内运行的 Python skill：把 PDF / DOCX / 网页 / 主题文字转化为原生可编辑 PPTX |
| 社区热度 | 约 67k stars、5.3k forks、21 名贡献者；第三方评测收录为 Tier S（综合分 95），PPT Skill 赛道头部 |
| 成熟度 | v6.7.3；最新提交 2026-10-24（评估时两天前），持续高频迭代；已发布 18+ 个稳定版本 |
| 工程质量 | 102 个工程化脚本；工作流文档化程度极高（路由规则、能力边界、映射指南逐条成文并带校验硬门） |
| 许可与成本 | MIT，工具本身免费；唯一成本是所用大模型的 API 用量或本地推理算力 |
| 数据安全 | 除与模型的对话外，全流程本地执行；源文档与产出不出本机 |
| 平台锁定 | 无。任何具备 agent 能力（读写文件、执行命令、多轮对话）的工具均可驱动 |

### 2.2 核心亮点

1. **"原生深度"是真差异化。** 与 Gamma、Canva AI 等输出扁平页面（图片式幻灯片）的工具不同，PPT Master 导出的 deck 带真正的 `p:sldMaster` / `p:sldLayout` 继承结构、带调节手柄的原生形状与连接符、可"编辑数据"的 PowerPoint 原生 Chart / Table 对象（`--native-charts-and-tables`）、可编辑的 OMML 公式。产出物在 PowerPoint 里打开后可继续逐对象编辑，这是"给非技术方二次编辑"场景的硬需求。
2. **三条产物路线，路由可预期。** `Generate PPTX`（全新生成）、`Create Template`（提炼可复用模板）、`Edit Native PPTX`（保留原设计回填内容）各有明确的准入条件与输出契约，由 `workflows/routing.md` 确定性分发，不会"猜"用户意图。
3. **模板体系经过认真设计。** Brand / Style / Layout / Deck 四分类 + 四套 JSON 发现索引 + 质量校验硬门 + 简报强制确认门（`[TEMPLATE_BRIEF_CONFIRMED]`），不是简单的"换皮肤"。每类模板的所有权边界（身份 / 方法 / 结构 / 应用语境）在数据模型层面显式划分。
4. **自带企业级模板资产。** 内置 McKinsey、BCG、Accenture、PwC、Deloitte、JPMorgan、华为、阿里、腾讯、小米、NVIDIA、Microsoft、Google、Anthropic、AWS、IBM、**中国电信、中国电建、中汽研**等 19 个品牌工作区与 2 个完整 Deck 模板——其中"中国电信 2025 业绩解读"示例已官方验证了"自有品牌 deck → Deck 模板 → 新内容汇报"的完整闭环。
5. **能力边界诚实透明。** 官方维护 PowerPoint ↔ SVG 映射指南，逐特性记录"已覆盖 / 有意边界 / 暂不考虑"，SmartArt 是刻意的排除项而非缺口，降低了企业引入后的预期错配风险。

### 2.3 短板与边界

| 短板 | 影响 | 应对 |
|---|---|---|
| 效果强依赖模型能力（官方明言 `harness + model = agent`，推荐 ~1M token 上下文的大模型） | 用小模型跑完整流程质量差距明显 | 见 3.5 节缓解策略；本地模型场景须管理预期 |
| 学习曲线 | 需安装 Python 3.10+、会用 Agent 工具，无 GUI | 先由技术同学搭好环境，业务同学只面对聊天框 |
| SmartArt 刻意不支持（读取后重画为普通形状） | 含大量 SmartArt 的旧 deck 不适合整页再生 | SmartArt 以原生代理（proxy）原样保留；或走 Edit Native PPTX 路线 |
| 模板结构验收以 Microsoft PowerPoint 为准 | WPS / Keynote / LibreOffice 可能归一化模板结构 | 企业若主用 WPS，验收环节须在真 PowerPoint 中执行 |
| 快速模式一次性生成、不可续接 | 上下文丢失无法恢复 | 重要任务走默认流程（带确认页与 spec lock） |
| 无自动化回滚机制 | 误操作覆盖风险 | 配合第 6 章 GitHub 协作规则：一切产出 git 化，先提交再改 |

---

## 3. 应用环境与工具选型（接入本地 Ollama / vLLM + Qwen3:8B）

### 3.1 架构原理：模型接入在 Agent 宿主层解决，而非 skill 层

首先澄清一个关键认知：**PPT Master 本身不含 LLM 调用层，也不需要为它单独适配本地模型**。它是一组 Markdown 工作流文档 + Python 后处理脚本的"skill"，谁来"读懂并执行"这些工作流，取决于你把它放进哪个 Agent 宿主。宿主负责三件事：读文件、写文件、执行命令；模型（Qwen3:8B）是宿主的"大脑"。因此本地化改造的全部工作 = **给 Agent 宿主接上本地推理服务（Ollama 或 vLLM）**，PPT Master 零改动。

```
┌─────────────────────────────────────────────────────┐
│  Agent 宿主（Claude Code / Codex CLI / Qwen Code…）  │
│  - 读取 skills/ppt-master/ 工作流文档                │
│  - 读写 projects/ 下素材与产出                       │
│  - 调用 scripts/ 下的 Python 后处理（PPTX 导出等）    │
└──────────────┬──────────────────────────────────────┘
               │ HTTP（Anthropic 格式或 OpenAI 格式）
┌──────────────▼──────────────────────────────────────┐
│  本地推理服务                                        │
│  方案A：Ollama（:11434，原生 Anthropic 兼容 API）     │
│  方案B：vLLM（:8000，OpenAI 兼容 API，可加代理桥接）   │
│  模型：Qwen3:8B                                      │
└─────────────────────────────────────────────────────┘
```

### 3.2 Agent 宿主选型

| 宿主 | 本地模型接入方式 | 与 PPT Master 的契合度 | 结论 |
|---|---|---|---|
| **Claude Code** | Ollama 官方集成：`ollama launch claude`，或手工设 `ANTHROPIC_BASE_URL=http://localhost:11434`；vLLM 场景经 claude-code-router / LiteLLM 桥接 | **最高**——官方开发与测试最充分的环境，skill 插件体系完善 | **首选** |
| **Codex CLI** | Ollama 官方集成：`ollama launch codex` / `codex --oss -m qwen3:8b`；vLLM 走 `~/.codex/config.toml` 的 `model_providers`（OpenAI 兼容） | 官方兼容列表成员；ppt-master 的 image-to-pptx 路线明确标注 Codex 支持 | 备选首选 |
| **Qwen Code** | 原生支持 OpenAI 兼容端点，对 Qwen 家族模型天然亲和 | Gemini CLI 分支，可作通用文件型 Agent 使用 | 备选 |
| **OpenCode** | 原生支持 Ollama / OpenAI 兼容 | 通用 Agent | 备选 |
| **Cline / Roo Code（VS Code）** | 内置 Ollama provider | IDE 内使用，适合已用 VS Code 的团队 | 备选 |

> 选型理由：PPT Master README 明言"作者最推荐 Claude Code——本项目开发与测试最充分的环境"。当本地模型出现 Agent 循环卡顿（小模型常见）时，Claude Code 生态的排障资料（Ollama 官方集成文档、claude-code-router、LiteLLM 社区实践）也最丰富。

### 3.3 推理服务层：Ollama vs vLLM

| 维度 | Ollama | vLLM |
|---|---|---|
| 定位 | 个人工作站 / 单机易用 | 团队共享 GPU 服务器 / 高吞吐 |
| 启动方式 | 一条命令，自动管理模型文件 | 需手动配置参数，适合长驻服务 |
| API 格式 | 同时提供 **Anthropic 兼容**与 OpenAI 兼容端点（Claude Code 可直连，无需桥接） | OpenAI 兼容；接 Claude Code 需经 claude-code-router 或 LiteLLM 转换 |
| 工具调用 | 对 qwen3 自动处理 | 必须显式开启：`--enable-auto-tool-choice --tool-call-parser hermes` |
| 上下文管理 | 默认值偏小，**必须显式调大**（官方建议跑 Claude Code / Codex 用 64k+） | 启动参数 `--max-model-len` 控制；Qwen3-8B 原生 32k，YaRN 扩展至 131k |
| 并发 | 单用户为主 | continuous batching，多用户友好 |

**结论**：个人或小规模试点用 **Ollama**（零桥接成本）；团队共享 GPU、需要并发服务时用 **vLLM**。

### 3.4 推荐组合与关键配置

**组合一（首选，单机）：Claude Code + Ollama + Qwen3:8B**

```bash
# 1. 安装与拉模型
ollama pull qwen3:8b

# 2. 调大上下文后启动（跑 PPT Master 建议 ≥32k，官方对 Agent 场景建议 64k+）
export OLLAMA_CONTEXT_LENGTH=65536
ollama serve

# 3a. 一键方式：Ollama 官方集成
ollama launch claude        # 在选择器中选 qwen3:8b

# 3b. 手工方式：环境变量直连 Anthropic 兼容端点
export ANTHROPIC_BASE_URL=http://localhost:11434
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_API_KEY=""
claude --model qwen3:8b

# 4. 在 Claude Code 中打开 ppt-master 目录，开始使用
cd ppt-master
claude
```

**组合二（团队共享 GPU）：vLLM + Claude Code（经 claude-code-router 桥接）**

```bash
# 1. 起 vLLM 服务（工具调用与上下文长度两个参数不可省）
vllm serve Qwen/Qwen3-8B \
  --max-model-len 32768 \
  --enable-auto-tool-choice --tool-call-parser hermes \
  --gpu-memory-utilization 0.90

# 2. 安装桥接层，把 Claude Code 的 Anthropic 格式请求翻译为 vLLM 的 OpenAI 格式
npm install -g @musistudio/claude-code-router
# ~/.claude-code-router/config.json 中配置 Provider：
#   name: "vllm-qwen3-8b"
#   api_base_url: "http://<gpu-server>:8000/v1/chat/completions"
#   models: ["Qwen/Qwen3-8B"]
# 启动：ccr code
```

**组合三（Codex CLI 直连）**

```bash
# Ollama 快捷方式
codex --oss -m qwen3:8b

# vLLM：~/.codex/config.toml
# model = "Qwen/Qwen3-8B"
# model_provider = "my-vllm"
# [model_providers.my-vllm]
# name = "vLLM"
# base_url = "http://<gpu-server>:8000/v1"
# wire_api = "chat"        # vLLM 版本较旧时用 chat；新版本支持 responses 可改
```

**硬件参考**：Qwen3-8B 4bit 量化权重约 5–6 GB；16 GB 显存可流畅跑 32k 上下文；要开到 64k+ 上下文建议 24 GB 显存（配合 KV cache 量化）。CPU 推理可跑通但 Agent 循环延迟显著，不推荐用于生产。

### 3.5 Qwen3:8B 的能力边界与缓解策略（必读）

必须诚实面对：**PPT Master 的产出上限由模型决定**。官方推荐 Kimi K3 / Claude 量级（~1M 上下文）的大模型，而 Qwen3:8B 是 32k 原生上下文的 8B 模型，两者存在代差。以下是现实的差距判断与工程缓解手段：

**预期差距**：

- 完整 Default 流程需读入 SKILL.md + 路由 + 工作流 + 多个 reference 文档，再加源材料，32k 上下文可能吃紧；
- 整页 SVG 创作（布局 + 文案 + 视觉）对 8B 模型的设计能力和长输出稳定性要求高，产出更依赖人工打磨；
- 多步 Agent 循环中，小模型偶发"只规划不执行"或卡在思考态（社区已有多起报告，可通过关闭 thinking 模式缓解）。

**缓解策略（按优先级）**：

1. **优先走 Edit Native PPTX 路线（第 5 章路径 A）**。"往现有模板里填内容"是三条路线中生成自由度最低、最依赖模型自由发挥也最少的路线——页面结构、版式、设计都来自源文件，模型只做选页和替换文字，8B 模型的胜任度最高。这恰好与企业"复用模板"的主诉求重合。
2. **用 Quick 模式减少上下文负担**：`"快速生成一份 5 页 PPT，不用跟我确认"` 会跳过交互确认与持久规划，读入的技能文件更少。
3. **控制任务粒度**：单次生成页数控制在 5–8 页；长材料先让模型分段提炼再生成；一次只做一个项目。
4. **上下文务必调大**：Ollama 默认上下文远小于 32k，跑本工作流前必须按 3.4 节显式设置，否则会出现"读到一半忘前面"的典型症状。
5. **分层路由（进阶）**：claude-code-router 支持按任务类型切换模型——规划/创作阶段用云端大模型（Kimi/Claude），机械执行阶段（文件转换、脚本调用、格式修补）用本地 Qwen3:8B，兼顾质量、成本与数据安全。
6. **图像与语音取舍**：Qwen3:8B 不出图。配图改用 `image_search.py` 网络图源（零配置可用）或用户自带素材；旁白 TTS 如需接入，PPT Master 已原生支持 Qwen TTS API（`--provider qwen`，DashScope 云端）。
7. **硬件允许时升档**：Qwen3 系列 14B / 32B 对多步 Agent 工作流的稳定性提升显著；如共享 GPU 预算允许，优先升模型而不是调流程。

---

## 4. 上手路径

```text
① 安装 Python 3.10+
② 安装任一 Agent 宿主（推荐 Claude Code，见第 3 章接入本地模型）
③ git clone https://github.com/hugohe3/ppt-master.git && pip install -r requirements.txt
④ 在宿主中打开 ppt-master 目录，把素材放入 projects/，用对话驱动
```

常用指令示例：

```text
生成：   请用 projects/q3-report/sources/report.pdf 生成一份 PPT
回填：   把 projects/新材料.md 的内容套进 projects/source/公司模板.pptx，保留原设计
提炼模板：用 /create-template 工作流，基于 projects/brand/our_deck.pptx 创建可复用模板
快速模式：用 projects/brief.md 快速生成一份 5 页 PPT，不用跟我确认
```

可选配置 `.env`：生图 API Key（`IMAGE_BACKEND`）、Pexels / Pixabay 图库 Key（提升图片质量）、Qwen TTS Key（旁白）。纯本地模型方案下这些均可跳过，不影响主流程。

---

## 5. 企业模板复用方案（核心）

PPT Master 把"复用企业模板"设计为三大主线之一，提供三条路径，按企业场景对号入座。

### 5.1 路径 A：直接回填 —— Edit Native PPTX（最快见效）

**适用**：手里已有公司 `.pptx` 模板或成品 deck，想保留原页面壳、只换内容。

**原理**：源 deck 通过 `pptx_to_svg.py --roundtrip` 导入为"保留来源的 round-trip 工作区"——每张源页面变成一个紧凑可编辑 SVG + 不可变原生背书。**未改动的页面逐字节恢复**（`passthrough`），改动的页面只重建修改过的对象（`rebuilt`），母版与版式全程原样保留。

**操作**：对话里说"把这些内容套进这份模板，保留原设计，只选适合的页面"。工作流会：

1. 导入 round-trip 工作区，生成 `authoring_summary.json`（逐页文字/图片/占位符清单）；
2. 把源页面当作"页面库"做内容映射规划（页面结构本身承载论证逻辑：总分、对比、递进），并与你确认一次；
3. 通过 `page_plan.json` 选页、重排、复用某页多次；
4. 可叠加讲稿（notes）、旁白音频、自动翻页、转场动画；
5. 导出时打印回执 `passthrough / cloned_passthrough / patched / rebuilt` 计数，逐字节保留情况可验证。

**边界**：不能新增母版/版式结构、不能改页面尺寸——需要新结构时走路径 B。

**模型压力**：低。选页与文字替换对 8B 本地模型友好，是 Qwen3:8B 环境下的主力路线。

### 5.2 路径 B：提炼成可复用模板 —— Create Template（模板资产化，重点推荐）

**适用**：想把企业 VI / 版式沉淀为"一次提炼、反复使用"的资产，供团队长期复用。

**操作入口**：对话输入 `"用 /create-template 工作流，基于 projects/brand/our_deck.pptx 创建可复用模板"`。工作流强制先确认一份模板简报（不允许跳过），确认后 AI 自动完成分析、创作、校验。

**第一步：按"真正要复用什么"选择模板类型（kind）**——这是官方强制确认的简报环节：

| Kind | 复用什么 | 典型企业场景 | 产出物 |
|---|---|---|---|
| **Brand** | 颜色 / 字体 / Logo / 语调 / 图标风格 | 只有 VI 手册，还没成型的 PPT | 仅身份 spec，无页面原型 |
| **Style** | 沟通方法 + 视觉默认值 | 复用"咨询式论证风格"但不锁页面 | 仅方法 spec，无页面原型 |
| **Layout** | 品牌中立的页面结构（母版/版式/占位符） | 一套骨架供多品牌 / 多子公司复用 | 结构 spec + SVG 页面原型 |
| **Deck** | 身份 + 结构 + 应用语境一体化 | **企业定期汇报、投标书等成体系的反复演示**（大多数企业的选择） | 完整工作区：应用契约 + 身份 + 结构 + SVG 原型 |

**第二步：理解内部实现模式**（AI 自动推导，无需用户选择）：

- `standard`：重新创作一套紧凑的可复用系统；
- `fidelity`：更广的来源对齐，保留更多来源版式；
- **`mirror`**：逐页物化来源事实，最忠实——直接给原始 `.pptx` 时导入器会读取 OOXML 里真实的 Master / Layout / placeholder / 主题 / 原生形状 / 素材，能保留的都保留。

**第三步：产出的模板工作区结构**：

```text
templates/<kind>/<id>/
├── templates/
│   ├── design_spec.md        # 模板契约：身份、结构、页面语法、槽位约定
│   ├── 01_cover.svg          # 每种页面角色一个原型
│   ├── 03_chapter.svg
│   ├── 04_content.svg
│   └── 05_ending.svg
├── images/                   # logo、位图素材
├── icons/imported/           # 导入的矢量素材唯一规范副本
└── exports/<id>_template_preview.pptx   # 审阅预览（多母版时强制生成）
```

**第四步：注册与复用**。`library` 范围（默认）写入全局库并注册进 `decks_index.json` 等索引，出现在 Stage 1 模板选择器中；`project` 范围写入项目目录，可跨项目以精确路径复用。此后每次生成新 deck，在对话里给出**精确工作区根路径**：

```text
用 projects/q4-report/sources/report.pdf 做 deck。
模板工作区：skills/ppt-master/templates/decks/<我司模板>/
```

AI 读取全部原型与当前内容，自动决定选页 / 复用 / 重排，导出**带真母版与版式继承**的 PPTX。旧版平铺模板包会被拒绝——必须经 Create Template 重建，保证资产质量一致性。

**验收标准**（在真 Microsoft PowerPoint 中执行）：

| 检查位置 | 预期结果 |
|---|---|
| 视图 → 幻灯片母版 | 能看到模板声明的 Master 与具名 Layout |
| 开始 → 新建幻灯片 | 版式选择器出现可复用 Layout 名称 |
| 选中生成页查看"版式" | 页面绑定到声明的 Layout，而非通用版式 |
| 点击可复用内容区域 | 模板槽位表现为带类型与边界的原生占位符 |
| 从导出的 Layout 新建一页 | 不复制成品页也能得到固定视觉与占位符几何 |

### 5.3 路径 C：Brand + Layout 组合（规模化变体）

模板体系成熟后，用"整段应用"规则做组合：Layout 存在时结构归 Layout，身份归 Brand，不做字段级混搭。典型玩法：

- **我司 Brand + 通用 Layout**：身份统一，结构按场景换（汇报用一套、路演用一套）；
- **通用 Layout + 各子公司 Brand**：结构统一，身份各异；
- **Deck + 外部 Brand 覆盖**：同一应用语境，不同活动主视觉。

字段级微调（如"主色改成 #FF0000"）不在安装层做，走对话确认即可生效。

### 5.4 决策速查

| 你手里有什么 / 想干什么 | 走哪条 | 模型压力 |
|---|---|---|
| 现成 deck，套壳换内容，马上要 | **A** Edit Native PPTX | 低（8B 可胜任） |
| 公司模板要长期反复用、团队共享 | **B** Create Template → Deck | 高（建议一次性用大模型完成提炼，产出物与模型无关） |
| 只有 VI 手册 / logo / 色板 | **B**（Create Brand + 现成或新建 Layout） | 中 |
| 多品牌共用一套骨架 | **C** Layout + Brand 组合 | 中 |
| 想整页重新设计（改版） | Generate PPTX 的 Beautify 路线（不要硬套 A） | 高 |

> **关键洞察**：模板提炼（路径 B）是一次性的高模型压力任务，可以借一次云端大模型完成；而模板**消费**（每天生成新 PPT、回填内容）是高频低压力任务，完全可以由本地 Qwen3:8B 承担。这个"一次贵、长期便宜"的结构，正好匹配"本地化部署 + 模板复用"的企业诉求。

---

## 6. 落地路线图

| 阶段 | 目标 | 动作 | 周期参考 |
|---|---|---|---|
| 一、环境验证 | 跑通最小闭环 | 按 3.4 节搭 Ollama + Claude Code + Qwen3:8B；用内置模板（如 `presentation_core`）生成 5 页测试 deck；跑一遍路径 A 回填测试 | 1–2 天 |
| 二、模板试点 | 沉淀第一个企业模板 | 选一套使用频率最高的公司 deck，走路径 B 提炼为 Deck 工作区（此步骤建议接入云端大模型完成）；在真 PowerPoint 中按 5.2 验收表检查 | 1 周内 |
| 三、并行对比 | 验证产出质量 | 同一批内容分别用"本地 8B"和"云端大模型"从模板生成，对比打磨工作量，确定日常路线与分层路由策略 | 2 周 |
| 四、团队推广 | 资产 git 化 + 协作规范 | 模板工作区与本文档一并纳入 git 仓库（仓库地址见《GitHub 协作规则》），按协作规则提交推送；团队成员按第 4 章上手 | 持续 |
| 五、规模化 | 多模板 / 多场景 | 按需扩展 Brand / Layout / Deck 模板族，启用路径 C 组合；评估升级模型档位 | 按需 |

## 7. 风险与验收清单

- [ ] 所有 PPTX 产出在 **Microsoft PowerPoint**（非 WPS）中完成母版/版式/图表可编辑性验收
- [ ] Ollama / vLLM 上下文长度已显式调大（≥32k），Agent 循环无"读一半忘前面"症状
- [ ] 模板提炼完成后，`design_spec.md` + SVG 原型 + 素材已按《GitHub 协作规则》提交入库
- [ ] round-trip 回填任务的导出回执中，`passthrough` 计数与预期保留页数一致
- [ ] 敏感材料场景确认：仅模型对话出网；若用云端模型处理敏感内容，先过合规评审，否则坚持纯本地路线
- [ ] 每批产出先 git 提交再迭代修改（弥补工具无自动回滚）

## 附录 A：命令速查

```bash
# 环境更新（clone 安装方式）
python3 skills/ppt-master/scripts/update_repo.py

# 查看一个 PPTX 里到底有什么（编辑前确认识别结果）
python3 skills/ppt-master/scripts/pptx_to_svg.py <deck.pptx> -o projects/<slug> --inheritance-mode both --roundtrip
python3 skills/ppt-master/scripts/pptx_template_import.py <deck.pptx> --manifest-only -o <workspace>

# 回填导出与校验
python3 skills/ppt-master/scripts/svg_to_pptx.py "projects/<slug>" --roundtrip
python3 skills/ppt-master/scripts/pptx_delivery_check.py "<output.pptx>"

# 图片检索（零配置可用）
python3 skills/ppt-master/scripts/image_search.py --query "<关键词>"
```

## 附录 B：参考资料

- 项目仓库：https://github.com/hugohe3/ppt-master （AtomGit 镜像：https://atomgit.com/hugohe3/ppt-master）
- 示例在线预览：https://hugohe3.github.io/ppt-master-examples/
- 模板架构文档：`docs/zh/templates-architecture.md`、`docs/zh/templates-guide.md`
- 三条路线规范：`skills/ppt-master/workflows/`（generate-pptx / create-template / edit-native-pptx）
- Ollama × Claude Code 官方集成：https://docs.ollama.com/integrations/claude-code
- Ollama × Codex CLI 官方集成：https://docs.ollama.com/integrations/codex
- 模型桥接：claude-code-router（https://github.com/musistudio/claude-code-router）、LiteLLM
