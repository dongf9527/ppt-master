# Codex CLI + Ollama + Qwen3-VL WSL 部署手册（本地视觉 Agent 栈）

> 文档版本：v1.1 ｜ 整理日期：2026-10-26
> 目标环境：Windows 10/11 + WSL2（Ubuntu 22.04 / 24.04），参考实机 CNBEJFSOP03（RTX 5080 16GB）
> 定版模型：**qwen3-vl:8b**（Q4_K_M，磁盘 6.2GB，满足「10G 以下 · Q4」选型约束）
> 变更记录：v1.0 首发定版（依据 Ollama 官方 Codex CLI 集成文档、qwen3-vl 模型库页 2026-10 当期数据、Task 27 本地栈可行性评估结论、工作站 image-to-pptx 首战审计 8 违规教训）；v1.1 补全两大缺口——5.4 扩写为「WSL 内从零安装 Codex CLI 完整步骤」（Node 前置 / npm·官方脚本·二进制三路线 / 安装后验证 / 首次运行预期），5.5 扩写为「连通原理 + 连通前自检 + 四种接入方式」（新增官方 `codex --oss` 手动方式，原方式二/三顺延为三/四，全文交叉引用同步更新）
> 配套文档：`Claude-Code-WSL部署与本地模型接入方案.md`（下称《CC 方案》，v1.6）、`../评估方案/PPT-Master评估与企业模板复用方案.md`、上游 `skills/ppt-master/workflows/profiles/image-to-pptx.md`（130 行）、`../GitHub协作规则.md`
> 说明：本文所有命令除特别注明 "Windows 侧" 外，均在 WSL 终端内执行。本文与《CC 方案》为兄弟文档：Claude Code 栈管文本类工作流，本栈（Codex）管视觉重建类工作流。

---

## 1. 目标与能力边界（务必先读）

### 1.1 部署目标

本手册要达成三件事：第一，在 WSL2 的 Ubuntu 环境中完成 Codex CLI 的安装，并把其模型后端从 OpenAI 云端切换到 WSL 内**已经存在**的 Ollama 服务，全程数据不出本机；第二，拉取并调优 Qwen3-VL 视觉模型（约束：10G 以下、Q4 量化，定版 `qwen3-vl:8b`）；第三，打通"看图 → 对象化重建"的本地视觉 Agent 闭环，作为 ppt-master **image-to-pptx「裁剪保真版」路线**的实机验证环境。

为什么必须是"视觉模型"：工作站 image-to-pptx 首战审计（13m44s 零交付、8 违规）根因链的第一环，就是当时本地模型 qwen3.5:9b **没有视觉输入能力**——profile §2 的强制视觉检查（"亲眼看图"）在物理上不可行，模型只能用颜色计数去猜 "likely a photo"，后续分层决策全部建立在猜测之上。换 qwen3-vl 是从硬件层修复该违规，而不是靠提示词祈祷模型"记得看图"。

### 1.2 与官方宿主的能力差异（选型前必知）

ppt-master image-to-pptx profile 的官方验证宿主是 **Codex CLI + GPT-5.x 云端**。官方宿主具备四类能力，本地栈只具备其中三类，这个差异决定了本栈的任务边界：

| 能力 | 官方宿主（Codex + GPT-5.x 云端） | 本地栈（Codex + Ollama + qwen3-vl:8b） | 对 image-to-pptx 的影响 |
|---|---|---|---|
| 文本推理 / 工具调用 | ✅ | ✅（8B 量级，指令遵循弱于云端，需第 7 章硬门兜底） | 主流程可跑 |
| 图像理解（看图） | ✅ | ✅ qwen3-vl 原生视觉输入（256K 上下文） | profile §2 强制视觉检查从"物理不可行"变为"可行"，审计违规②的硬件级修复 |
| 本地图载入上下文（`-i` 传图 / `view_image` 工具） | ✅ | ✅（经端点传图，注意 4.3 的静默失败风险点） | 同上 |
| **图像生成 / 编辑（官方内置 image_gen：背景修补、主体重绘）** | ✅ 内置 | ❌ **Ollama 与 qwen3-vl 均不提供，本地无解** | profile identity-faithful 五条路径中，仅"参考重建"依赖生成能力 → 本地不可用 |

### 1.3 能跑什么、不能跑什么（结论表）

| 源件类型 | 本地栈判定 | 依据 |
|---|---|---|
| 信息图 / 图表类（如 001-AI方法论.png：68 文本块 + 图标 + 卡片 + 架构插画） | ✅ 可跑「裁剪保真版」 | 全部元素可走 identity-faithful 的**精确矢量 / 确定性重绘 / 源像素裁剪**三条合规路径；Task 27 已对 001 源件判定"全图零图像生成需求" |
| 照片类、需要 AI 背景修补 / 主体抹除重建的源件 | ❌ 完整保真重建不可行 | 无图像生成能力，背景修补（inpainting）无解 → 如实标记 Needs-Manual，或归云侧 Codex / 沙箱 quick-generate（Task 24 已实证） |
| 纯文本重排（OCR 式） | ✅ | qwen3-vl 官方 OCR 支持 32 语言，倾斜 / 模糊场景增强 |

一句话定位：**本地栈 = 信息图类轻量保真生产线；照片类完整重建仍归云侧**。这与 Task 27 评估结论一致，本手册就是把该结论中的"换 qwen3-vl 可试"活口落成可执行环境。

---

## 2. 模型选型定版

### 2.1 Qwen3-VL 家族尺寸对照（Ollama 模型库当期数据，2026-10）

| 标签 | 磁盘占用 | 参数量 | 架构 | 满足 10G 以下？ |
|---|---|---|---|---|
| qwen3-vl:2b | 1.9GB | 2B | 稠密 | ✅ |
| qwen3-vl:4b | 3.3GB | 4B | 稠密 | ✅ |
| **qwen3-vl:8b** | **6.2GB**（LLM 5.0GB + CLIP 视觉塔 1.2GB） | 8.19B | 稠密 | ✅ ← **定版** |
| qwen3-vl:30b（MoE） | 20GB | 30.5B（激活 3B） | MoE | ❌ |
| qwen3-vl:32b | 21GB | 32B | 稠密 | ❌ |
| qwen3-vl:235b | 143GB | 235B-A22B | MoE | ❌ |

全家族共同点：原生 256K 上下文、Text+Image 双输入、Apache 2.0 协议；**要求 Ollama ≥ 0.12.7**；模型自带采样参数 temperature=1 / top_k=20 / top_p=0.95。

### 2.2 定版 qwen3-vl:8b 的五条理由

1. **10G 约束内的最大可用档**：6.2GB 磁盘占用留足余量；8B 级在视觉 grounding（坐标级空间感知）、OCR、版面结构重建上显著优于 4B/2B——而 image-to-pptx 的分层决策恰恰依赖 bbox 级的精确空间感知，这是全任务对模型最重的一项要求。
2. **显存可容纳**：RTX 5080 16GB + q8_0 KV cache 可在 64k 上下文实现 100% GPU 运行（第 6 章预算表），不需要 CPU 回退；8B 是"16GB 卡 + 64k 上下文"组合下的现实上限。
3. **默认标签即 8B**：`qwen3-vl:latest` = `qwen3-vl:8b`（官方页 same digest），社区验证最充分，踩坑面最小。
4. **Visual Agent 官方定位契合**：Qwen3-VL 官方主打"能识别界面元素、调用工具、完成任务的视觉 agent"（OS World 等基准领先），与 Codex 的工具循环模式天然匹配。
5. **Ollama 一等公民集成**：官方模型页将 qwen3-vl 列入 `ollama launch` 应用矩阵，官方 Codex 集成文档专门给出 per-model 的 vision / 上下文 / thinking 档案管理。

### 2.3 变体与标签说明

- **`:8b`（= `:latest`，即 Instruct 版）→ 本手册定版**。Agent 工具循环要求每轮快速产出 tool call，不要选 `:8b-thinking` 系变体：思考 token 会拖慢每一次工具调用，而 8B 的思考增益在结构化重建任务上不划算，还可能干扰工具调用格式。
- 量化后缀：`-q4_K_M` 为默认档（本手册）；`-q8_0`（8B 对应约 12GB 级）与 `-bf16` 均超出 10G 约束，不用。`qwen3-vl:8b` 与 `qwen3-vl:8b-instruct-q4_K_M` 为同一 digest。
- 一切以拉取后 `ollama list` 显示的**实际名字**为准——第 9 章两类高频坑都源自"配置里的模型名与 list 名字不一致"。

---

## 3. 前置条件与检查

部署前逐项核对，任何一项不满足先处理再继续。与《CC 方案》第 2 章重叠的检查项此处从简，方法一致。

### 3.1 WSL2 与 GPU 直通

```bash
wsl -l -v        # Windows 侧：VERSION=2、STATE=Running（WSL1 无 GPU 直通，不可用）
nvidia-smi       # WSL 内：能输出 RTX 5080 信息即直通正常
```

要求 Windows 侧 NVIDIA 驱动正常；WSL 内不安装 Linux 显卡驱动（同《CC 方案》2.2）。

### 3.2 Ollama（三条版本门）

```bash
ollama --version                    # 硬门槛 ≥ 0.12.7（qwen3-vl 最低要求）；建议 ≥ 0.31（ollama launch / 官方 Codex 集成成熟档）
ollama list                         # 确认 qwen3.5:9b 在位（与本栈并存，不冲突）
curl -s http://localhost:11434/     # 应返回 "Ollama is running"
```

低于 0.12.7：qwen3-vl 拉取后无法正常推理，必须升级；`ollama launch` 子命令不存在：说明版本偏旧，第 5.5 节改用方式二（`codex --oss`，仅依赖 codex 侧）/ 方式三 / 方式四。升级命令（模型与已有配置不受影响）：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 3.3 Node.js 与 Codex CLI 前置

```bash
node -v      # ≥ 22 已就绪则跳过；未装 / 版本低于 22 时的安装动作见 5.4 第 1 步
which codex  # 部署前应为空（未装）；部署后验证：应指向 WSL 内的安装，而非 /mnt/c/...
```

Codex CLI 官方不支持 Windows 原生运行，**WSL 就是官方 Linux 安装路径**——这也是本手册全部步骤都在 WSL 终端内执行的原因。安装动作统一收在第 5.4 节。

### 3.4 磁盘与内存

模型 6.2GB + codex 安装 + 工作区，预留 ≥ 15GB 磁盘；系统内存 ≥ 32GB（图像编码与长上下文有额外内存开销）。

### 3.5 代理环境（工作站专项，Task 27 三坑之一）

公司代理环境下，codex → Ollama 的 localhost 回环流量若被代理劫持会表现为连接超时 / 407：

```bash
echo "HTTP_PROXY=$HTTP_PROXY HTTPS_PROXY=$HTTPS_PROXY NO_PROXY=$NO_PROXY"   # 先取证
# 若 NO_PROXY 未包含本地回环，则在 shell 配置与 systemd override 中补齐：
export NO_PROXY="127.0.0.1,localhost"
```

---

## 4. 架构与数据流

### 4.1 目标架构

```text
┌─────────────────────────── Windows 宿主 ───────────────────────────┐
│  ┌────────────────────────── WSL2 (Ubuntu) ──────────────────────┐  │
│  │                                                               │  │
│  │  ┌────────────────────┐  OpenAI 兼容端点（responses|chat） ┌─────────┐ │
│  │  │  Codex CLI         │ ─────────────────────────────────► │ Ollama  │ │
│  │  │  (Agent 宿主)      │     http://localhost:11434/v1/     │ :11434  │ │
│  │  │  + ppt-master      │ ◄── 文本 / 工具调用 ──────────────  │         │ │
│  │  │    skill           │                                    │ qwen3-  │ │
│  │  │                    │  图像注入三条通道：                 │ vl:8b   │ │
│  │  │  codex -i img.png  │  ① -i/--image 启动参数              │ (LLM +  │ │
│  │  │  TUI 粘贴/拖入      │  ② TUI 粘贴或拖入图片               │  CLIP   │ │
│  │  │  view_image 工具   │  ③ 模型侧 view_image 工具载入本地图  │  视觉塔) │ │
│  │  └────────────────────┘                                    └─────────┘ │
│  └───────────────────────────────────────────────────────────────┘  │
│                        GPU：RTX 5080 16GB（CUDA on WSL 直通）         │
└────────────────────────────────────────────────────────────────────┘
```

要点三条：①Codex 与 Ollama 同在 WSL 内走 localhost 回环，不涉及跨子系统端口转发；②Codex 讲 OpenAI 兼容协议（`/v1` 前缀），Claude Code 讲 Anthropic 兼容协议（`/v1/messages`），两栈共享同一个 Ollama 服务、同一个模型库、同一块 GPU；③图像进入模型有三条通道（`-i` 参数、TUI 粘贴、模型主动调 `view_image` 工具），部署验收要各验一条主力通道。

### 4.2 与《CC 方案》栈的分工

| | Claude Code 栈（《CC 方案》） | 本栈（Codex） |
|---|---|---|
| 模型 | qwen3.5:9b（纯文本） | qwen3-vl:8b（视觉） |
| 协议端点 | Anthropic 兼容 `/v1/messages` | OpenAI 兼容 `/v1/responses`（或 `/v1/chat/completions`） |
| 适配工作流 | Edit Native PPTX、quick-generate 等文本类 | image-to-pptx 看图重建类（裁剪保真版） |
| 能否看图 | ❌（首战翻车根因） | ✅ |

### 4.3 关键风险点：Responses 端点的图像输入（必读）

Codex 新版默认走 `wire_api = "responses"`（`/v1/responses` 端点）。Ollama 的 responses 实现**曾有"图像输入被忽略"的缺陷**（GitHub ollama#13717，2026-01 收录）：请求携带的图片根本不进模型，模型回复"我看不到图"——**静默失败，不报错**。当期 Ollama 官方文档已把 vision 纳入 Codex 集成的 per-model 档案管理，问题应已收敛，但本项目纪律是"假设必须被实测校准"：**必须用 5.6 的图像冒烟回合判定，不能假设通**。

判定与降级路径（三层，逐层收窄）：

1. 图像冒烟通过 → 用方式一 / 方式二 / 方式三（responses 通道），享受官方集成；
2. 冒烟失败（模型说看不到图）→ 切方式四 `wire_api = "chat"`——chat completions 的图像通道（image_url / images 字段）是老功能，成熟稳定；
3. chat 通道仍失败 → 用 5.3 的 `ollama run` CLI 直验 + 附录 B 的 curl 直验：CLI 通而 codex 不通 = 问题锁定在 codex 配置层；CLI 也不通 = 模型层 / 版本层问题，回头查 3.2。

---

## 5. 部署步骤

### 5.1 Ollama 升级与四参数

四参数与《CC 方案》6.2 节同源（systemd override 配置法、environ 取证、冷启动验证的完整三步**直接复用**《CC 方案》6.2，此处不重复），本栈取值与差异如下：

| 变量 | 本栈取值 | 与《CC 方案》的差异说明 |
|---|---|---|
| OLLAMA_CONTEXT_LENGTH | 65536 | 官方明确建议 Codex 场景 ≥64k；另注意**图像 token 也占上下文窗口**（001 类 1728px 信息图 ≈ 数千 token），64k 是文本+图像的合理起点 |
| OLLAMA_FLASH_ATTENTION | 1 | 新版 Ollama 对现代 NVIDIA 卡已默认开启；显式写入防止回退 |
| OLLAMA_KV_CACHE_TYPE | q8_0 | **本栈为必开项**：qwen3-vl:8b 是全注意力架构，64k 下 fp16 KV 约 9GB 必然 OOM（详见 6.1，与 qwen3.5:9b 的混合注意力不可类比） |
| OLLAMA_KEEP_ALIVE | 2h | 同《CC 方案》；代价见 6.3 的两模型换载延迟 |

### 5.2 拉取模型

```bash
ollama pull qwen3-vl:8b
ollama list     # 期望：qwen3-vl:8b  6.2GB（与官方页一致）
```

磁盘核对：若下载体积明显偏离 6.2GB，说明拉到了其他量化变体，核对标签后重拉。

### 5.3 模型级冒烟（先于 Codex，逐层隔离）

```bash
# 回合一：纯文本
ollama run qwen3-vl:8b "用一句话说明什么是PPT的对象化重建"

# 回合二：图像（官方 CLI 支持在 prompt 中直接附本地图路径）
ollama run qwen3-vl:8b "这张图片里最大的标题文字是什么？ /home/z/my-project/upload/001-AI方法论.png"
```

判定标准：回合二必须答出图中**真实存在**的内容（如 "AI 落地方法论" 类标题原文），而不是泛泛的"这是一张关于AI的图片"。答出具体文字 = 视觉通道通；声称看不到 / 答非所问 = 先解决模型层，不进入下一步。curl 版本（可脚本化、进 CI）见附录 B。

### 5.4 在 WSL 中安装 Codex CLI（从零到可用）

**第 1 步：Node.js 就位（npm 路线的前置，已装可跳过）**

Codex CLI 官方分发以 npm 包为主，Node.js 需 ≥ 22（建议 22 LTS）。用 nvm 装在 **WSL 内**——不要装到 Windows 侧，否则 interop 会抢占 PATH（《CC 方案》第 10 章同款教训）：

```bash
node -v   # 已 ≥ 22 则跳过本步
# 未装：先装 nvm（脚本 URL 与版本号以 github.com/nvm-sh/nvm 仓库 README 当前为准）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
exec $SHELL                    # 或重开终端，让 nvm 进 PATH
nvm install 22 && nvm use 22
node -v && npm -v
```

**第 2 步：安装 Codex CLI（三选一）**

| 路线 | 命令 / 动作 | 适用场景 |
|---|---|---|
| A. npm（默认推荐） | `npm install -g @openai/codex` | 已有 Node 栈，与《CC 方案》环境统一管理 |
| B. 官方独立安装脚本 | `curl -fsSL https://chatgpt.com/codex/install.sh \| sh` | 不想依赖 Node；重跑同一命令即完成升级 |
| C. GitHub Release 二进制 | 从 github.com/openai/codex/releases 下载 `codex-x86_64-unknown-linux-musl.tar.gz`，解压后把包内 `codex-x86_64-unknown-linux-musl` 重命名为 `codex`，放入 PATH 内目录（如 `~/.local/bin`）并 `chmod +x` | 内网 / 离线：先在能联网的机器下载再拷入 |

三条路线装的是同一个 codex，**选一条即可**；混装多份会重演《CC 方案》的多份安装排查（生效者 = PATH 顺序最前者）。本手册后续以路线 A 为准书写，B/C 路线仅安装动作不同。

**第 3 步：安装后验证**

```bash
codex --version   # 2026-10 当期 0.164.0；B/C 路线重跑安装动作即可升级
which codex       # 必须在 WSL 侧（nvm bin / ~/.local/bin / /usr/local/...），绝不能是 /mnt/c/...
codex --help      # 核对本手册用到的参数在位：--oss、--profile、-i/--image、-m
```

**第 4 步：首次运行的预期行为（不要被登录界面带跑）**

- 首次在任何目录运行 `codex` 会先弹「是否信任此目录」，选 Yes/Trust；
- 未配置 provider 前，界面会引导 **Sign in with ChatGPT**——本栈**不走这条路，不需要 OpenAI 账号**；
- 只要 5.5 的 provider 配置在位，正常启动后 `/status` 应显示 provider = Ollama、model = qwen3-vl:8b，全程无登录动作；认证由 provider 配置承担（其中方式四需要占位 `env_key`，见 5.5）；
- 启动仍强求登录 = provider 配置未生效：配置文件名 / `--profile` 参数 / `env_key` 三处逐一核对（第 9 章）。

### 5.5 Codex ↔ Ollama 连通：原理与四种接入方式

**连通原理（四种方式共用的同一条链）**

Codex 只是 Agent 宿主（客户端），模型后端完全由配置决定。所谓「连通」，就是让 codex 发出的 HTTP 请求落到 WSL 内 Ollama 监听的端口上：

```text
codex ──读配置──► model_provider ──► [model_providers.X].base_url
      ──HTTP──► http://localhost:11434/v1/   （Ollama OpenAI 兼容端点）
                wire_api = "responses" → 走 /v1/responses
                wire_api = "chat"      → 走 /v1/chat/completions
Ollama ──按请求中的 model 字段匹配 ollama list 名字──► qwen3-vl:8b 推理
```

三句话要点：①`base_url` 必须精确到 `/v1`（官方模板有带尾斜杠与不带两种写法，跟随模板即可），少写 `/v1` 必 404；②`wire_api` 决定走哪条端点路径——responses 是新协议、chat 是老协议，同一端口两条路径，与 base_url 是两层独立配置；③Ollama 本地默认不校验 API key，方式四的 `env_key` 只是满足 codex「provider 必须有认证变量」的存在性检查，值任意。

**连通前自检（不写任何 codex 配置，先证物理链路通）**

```bash
curl -s http://localhost:11434/                            # 期望 "Ollama is running"
curl -s http://localhost:11434/v1/models | grep qwen3-vl   # 期望模型 ID 在列
```

两条都过 = 连通的物理前提成立，之后的任何连通失败都是 codex 配置层问题（对照第 9 章）；第一条就不通 = Ollama 服务本身没起或端口不对，回 3.2。进入 codex 后的会话内确认：`/status` 看 provider 与 model，`/model` 看可切换列表。

**方式一（推荐起步）：`ollama launch codex` 一键**

```bash
ollama launch codex
```

- Ollama 自动刷新模型目录，并为本次会话生成专用 Codex 启动档案；
- 会话内 `/model` 可切换已安装的"工具能力"模型，每个模型保留自己的上下文 / vision / thinking 设置；
- 只有事先 `ollama pull` 好的模型才会出现在选择器里；
- 配置化：`ollama launch codex --config`；完全还原移除：`ollama launch codex --restore`；
- 前提：`ollama launch` 子命令存在（3.2 版本门）；首次运行需联网刷新目录（代理环境注意 3.5）。

**方式二（官方手动方式，最简零配置）：`codex --oss`**

```bash
codex --oss -m qwen3-vl:8b
```

- `--oss` 是 Codex 官方内置的「本地开源模型」入口：默认连 `localhost:11434` 的 Ollama，**不写任何配置文件**；
- `-m` 必须显式指定：不带 `-m` 时默认指向 gpt-oss 系模型而非 qwen3-vl；模型名与 `ollama list` 逐字符对齐；
- 依赖较新的 codex 版本（`codex --help` 中应有 `--oss`；当期 0.16x 具备），旧版 codex 改用方式三 / 方式四；
- 适合：不想落配置文件的快速验证、或临时在别的目录拉起会话。

**方式三（持久化，可入库）：profile 配置文件**

创建 `~/.codex/ollama-launch.config.toml`：

```toml
model = "qwen3-vl:8b"
model_provider = "ollama-launch"

[model_providers.ollama-launch]
name = "Ollama"
base_url = "http://localhost:11434/v1/"
wire_api = "responses"
```

启动：`codex --profile ollama-launch`。可选补充键（按本机 codex 版本文档为准）：`model_catalog_json`（官方示例含此键，由 `ollama launch` 生成的模型目录文件路径；纯手写 profile 可省略）、`model_context_window = 65536`、`model_max_output_tokens`。

**方式四（兜底逃生门）：手动 provider + chat 协议**

当 4.3 的 responses 图像风险兑现、或 Ollama / Codex 版本偏旧时启用。创建或编辑 `~/.codex/config.toml`：

```toml
model = "qwen3-vl:8b"
model_provider = "ollama"

[model_providers.ollama]
name = "Ollama (chat)"
base_url = "http://localhost:11434/v1"
wire_api = "chat"
env_key = "OLLAMA_API_KEY"
```

```bash
export OLLAMA_API_KEY=ollama    # 占位值，Ollama 不校验内容，仅满足 codex 的 env 存在性检查
codex
```

四种方式互不覆盖（会话级 vs 持久级、配置文件各不相同），推荐次序：**方式一验证通链路 → 方式三固化归档 → 方式四留作 4.3 风险兑现时的逃生门**（方式二适合不落盘的快速复验；方式一不可用时的官方替身即方式二）。

### 5.6 Codex 冒烟三回合

1. **文本回合**：`codex "用一句话说明你的运行模式"` → 回复正常，且会话内 `/status`（或 `/model`）显示 provider = Ollama、model = qwen3-vl:8b；
2. **工具回合**：让 Codex 读取一个已知文件并复述首行 → 工具调用正常完成。若出现连续无效工具参数，直接按第 7 章硬门 4 熔断处理，不重试；
3. **图像回合（本部署成败判定）**：

```bash
codex -i /home/z/my-project/upload/001-AI方法论.png "图中最大的标题是什么？一共有几个编号步骤？"
```

能说出图中真实内容 = 视觉链路全通；模型答"看不到图" = 按 4.3 三层降级路径处理（先 chat 兜底，再 CLI/curl 隔离定位）。

---

## 6. 显存预算与调优（RTX 5080 16GB 实机口径）

### 6.1 估算表（qwen3-vl:8b @ 65536 ctx）

| 项 | 估算值 | 依据 |
|---|---|---|
| LLM 权重（Q4_K_M） | ~5.0GB | 官方模型页 model 5.0GB |
| CLIP 视觉塔 | ~1.2GB | 官方模型页 projector 1.2GB |
| KV cache（q8_0 @64k） | **~4.5GB（估算）** | 8B 全注意力骨干（约 36 层 × 8 KV 头 × 128 维），q8_0 约 72KB/token × 65536 ≈ 4.6GB |
| 激活 / 计算缓冲 | ~0.5–1GB | 含图像编码峰值 |
| **合计** | **~11–12GB** | 16GB 卡可容纳，余量 4–5GB |

两个必须写明的口径问题：

- **不可与 qwen3.5:9b 类比**。《CC 方案》8.1 实测 9b@64k KV 仅 ~1.1GB，那是混合注意力（SWA/线性交替）架构的红利；qwen3-vl:8b 是全注意力架构，同参数量级下 KV 开销完全不同。估算时若照搬 9b 的实测值，会在 64k 档位上翻车。
- **32k 备选档**：KV 减半至 ~2.3GB，合计 ~9GB。纯图像问答任务足够，但 Codex agent 会话建议保持 64k（官方给出的 Codex 上下文下限即 64k），长会话与多图任务在 32k 下会频繁触发上下文压缩。

### 6.2 实测校准（沿用《CC 方案》8.1 方法论）

估算值必须被实测校准一次才算定版（本项目惯例）：

```bash
ollama ps          # 期望四列：NAME=qwen3-vl:8b / 100% GPU / CONTEXT=65536 / UNTIL≈2h
journalctl -u ollama --no-pager | grep -iE "kv_cache|kv cache"    # 服务日志看 llama_kv_cache 实际 MiB
```

若 PROCESSOR 列出现 CPU 百分比 = 显存溢出回退，按第 9 章 OOM 行处置。实测值出来后回填 6.1 表（做法与《CC 方案》v1.2 的 8.1 校准一致）。

### 6.3 与 qwen3.5:9b 的显存互斥与换载

qwen3.5:9b@64k 实测驻留 ~6.9GB，本栈 ~11.5GB，两者之和超过 16GB → **两模型不能同驻显存**。Ollama 按请求自动换载（先卸载当前模型再载入目标模型），KEEP_ALIVE=2h 配置下表现为：切换栈的第一次请求有 10–30 秒加载延迟，之后稳定。

处置策略：①接受延迟（两栈通常不会高频交替，推荐）；②交替密集期临时把 KEEP_ALIVE 降到 10m（换载更频繁但单次延迟更小）；③绝不建议为"共存"把两模型上下文都压到 16k——会同时毁掉两个栈的可用性。

---

## 7. image-to-pptx 本地运行规约（硬门外置）

工作站首战审计的核心教训之一：130 行散文式 profile 对 9B 级本地模型不可用，硬禁令必须**外置成任务提示词里的一票否决门**。本栈模型 8B，同样适用且更必要。启动任何 image-to-pptx 任务前，把以下五道硬门**原样粘贴进任务提示词**：

- **硬门 0（能力声明，对应违规①）**：任务开场第一句必须声明——"宿主 = Codex CLI，模型 = qwen3-vl:8b（本地，无图像生成能力）。identity-faithful 仅可用精确矢量 / 确定性重绘 / 源像素裁剪三条路径；参考重建路径不可用；如遇任何需要生成或修补图像的步骤，立即 Needs-Manual 停走，不得寻找替代生成工具。"
- **硬门 1（先看图，对应违规②）**：动手前必须完成一次图像回合（`-i` 传图或对源件路径调 view_image），并用一句话复述图中最大标题作为"看见"凭证；禁止用颜色计数、文件大小、扩展名猜测图片内容。
- **硬门 2（清单先行，对应违规③）**：任何分层决策之前，先产出 `analysis/reconstruction_inventory.json`（profile §5 强制件）；无清单不得开始重建。
- **硬门 3（禁令，对应违规④⑤）**：禁止调用或改写 `image_gen.py`、禁止手写 manifest 走生成管线、禁止把"screenshot 贴图式"当作方案选项——交付物必须是分层可编辑 PPTX。
- **硬门 4（熔断，对应违规⑧）**：连续 2 次无效工具参数即终止会话、报告卡点，不得重试第三次。

### 7.1 推荐跑法：裁剪保真版（8B 实力匹配）

信息图类源件的任务分解：文本 / 卡片 / 图标用**确定性重绘**（SVG 手搓 + ppt-master 管线导出原生 PPTX），照片 / 复杂插画从源图**裁剪为独立位图层**（源像素路径，profile 合规），背景用纯色或简单渐变重建。每页交付后用验收口诀自查：

> **PowerPoint / WPS 里文字可单独选中改字 + 选择窗格可见分层对象 = 真重建；一框选整页 = 贴图假还原。**

### 7.2 首个验证任务建议

直接使用 `001-AI方法论.png`（upload 已有，Task 27 已完成内容族拆解：68 文本块 + 16+3 图标 + 卡片 + 右侧 3D 架构插画，源像素充足，零图像生成需求）。该源件与沙箱 Task 24 手搓对照件（`docs/ppt/AI落地方法论/`）同源，本地栈产出可直接与沙箱对照评质——一次冒烟，两个目的（验证视觉栈 + 补全 §12"本该长什么样"实证链）。

---

## 8. 分层验收清单

| 层 | 验收项 | 通过标准 |
|---|---|---|
| ① 模型层 | `ollama list` / `ollama ps` | qwen3-vl:8b 6.2GB 在列；ps 显示 100% GPU、CONTEXT=65536、UNTIL≈2h |
| ② 视觉层 | 5.3 图像回合 + 附录 B curl | 均能答出图中真实细节（标题原文 / 步骤数），无"看不到图" |
| ③ Agent 层 | 5.6 Codex 三回合 | provider/model 正确、工具调用正常、无登录强求、无连续无效参数 |
| ④ 端到端层 | 001 源件最小重建 | inventory 先行 → SVG + 裁剪资产 → 质检导出 POSTFLIGHT passed → 开包验证：主标题文字可单独选中修改、选择窗格分层、旧文字残留命中 0 |

纪律：①②③ 全绿才允许进入 ④；④ 未过时按第 7 章硬门逐条排查后再试，**禁止在 ② 未过的情况下反复盲跑**——那正是工作站首战的翻车模式（视觉层从未通过，却在重建层狂奔了 13 分钟）。

---

## 9. 故障排查速查表

| 症状 | 根因 | 处置 |
|---|---|---|
| pull 成功但 run 报错 / 无视觉能力 | Ollama < 0.12.7 | 升级 Ollama（3.2） |
| codex 报 404 / responses unsupported | Ollama 版本旧，无 `/v1/responses` | 升级 Ollama；或方式四 `wire_api="chat"` |
| 模型答"我看不到图"（仅 codex 内） | responses 端点图像输入缺位（4.3） | 切方式四 chat；附录 B curl 隔离定位 |
| `codex: command not found`（WSL 内） | npm 全局 bin 不在 PATH / nvm 未生效 / 装到了 Windows 侧 | 重开终端或 `exec $SHELL`；`npm config get prefix` 核对 bin 路径；确认非 /mnt/c（5.4 第 3 步） |
| curl `/v1/models` 通但 codex 连不上 | base_url 少 `/v1`、端口错、wire_api 拼写错 | 逐项核对 provider 配置三要素（5.5 连通原理）；对照第 9 章登录行排查配置加载 |
| `codex --oss` 不识别该参数 | codex 版本过旧 | 升级 codex（5.4 第 2 步 B 路线重跑即升级）；或改用方式三 / 方式四 |
| CLI 单独跑也看不到图 | 图像路径未附上 / 文件不存在 | 核对绝对路径；换 curl 直验 |
| OOM / `ollama ps` 出现 CPU% | KV 过大（q8_0 未生效）或与另一模型同驻 | 查 override 四变量是否进服务进程（environ 取证法）；`ollama stop` 释放另一模型；降至 32k |
| codex 强制要求 OpenAI 登录 | 未用 launch/--oss/profile 方式，或配置文件未加载 | 核对文件名与 `--profile` 参数一致；方式四需 `export OLLAMA_API_KEY` 占位 |
| localhost 请求超时 / 407 | 代理劫持回环流量 | `NO_PROXY=127.0.0.1,localhost`（3.5） |
| 工具调用 JSON 频繁损坏 | 8B 模型 schema 压力 / 误用 thinking 变体 | 任务粒度细化（一次一步）；确认用 `:8b` 而非 `:8b-thinking`；硬门 4 熔断 |
| 模型名找不到 | codex 配置名与 `ollama list` 不一致 | 两边逐字符对齐（含冒号标签） |
| 切换栈首请求等 10–30s | 显存互斥自动换载（6.3） | 预期行为；按 6.3 策略权衡 KEEP_ALIVE |
| `/model` 选择器里没有 qwen3-vl | launch 前未 pull | 先 `ollama pull` 再 `ollama launch codex` |

---

## 10. 并存、切换与回滚

- **并存**：qwen3.5:9b（文本 / Claude Code 栈）与 qwen3-vl:8b（视觉 / Codex 栈）磁盘共存（合计 ~13GB），互不干扰；显存按需换载（6.3）。两栈的 Ollama 四参数共用一份 override，无需分裂配置。
- **切换**：工作流按"是否需要看图"选栈——Edit Native PPTX、quick-generate 等文本类 → Claude Code 栈；image-to-pptx 看图重建类 → 本栈。
- **回滚**：`ollama rm qwen3-vl:8b` + 删除 `~/.codex/` 下相关配置文件 +（方式一）`ollama launch codex --restore`。四参数 override 对 9b 栈无副作用，可保留。

---

## 附录 A：本栈配置汇总（开箱即用清单）

**Ollama 服务四参数**（systemd override，路径与取证方法见《CC 方案》6.2）：

```ini
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=65536"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_KEEP_ALIVE=2h"
```

**Codex profile 文件** `~/.codex/ollama-launch.config.toml`（方式三全文）：

```toml
model = "qwen3-vl:8b"
model_provider = "ollama-launch"

[model_providers.ollama-launch]
name = "Ollama"
base_url = "http://localhost:11434/v1/"
wire_api = "responses"
```

**Codex 兜底文件** `~/.codex/config.toml`（方式四全文，风险兑现时启用）：

```toml
model = "qwen3-vl:8b"
model_provider = "ollama"

[model_providers.ollama]
name = "Ollama (chat)"
base_url = "http://localhost:11434/v1"
wire_api = "chat"
env_key = "OLLAMA_API_KEY"
```

**shell 兜底变量**（`~/.bashrc`，代理环境必配）：

```bash
export NO_PROXY="127.0.0.1,localhost"
export OLLAMA_API_KEY=ollama   # 仅方式四需要
```

## 附录 B：curl 图像冒烟脚本（绕过 codex 层的隔离判定）

```bash
IMG_B64=$(base64 -w0 /home/z/my-project/upload/001-AI方法论.png)
curl -s http://localhost:11434/api/chat -d "{
  \"model\": \"qwen3-vl:8b\",
  \"messages\": [{
    \"role\": \"user\",
    \"content\": \"图中最大的标题文字是什么？一共有几个编号步骤？\",
    \"images\": [\"$IMG_B64\"]
  }],
  \"stream\": false
}" | python3 -c "import json,sys; print(json.load(sys.stdin)['message']['content'])"
```

判定矩阵：curl 通 + codex 通 = 全链路就绪；curl 通 + codex 不通 = 问题在 codex 配置层（走 4.3 降级）；curl 不通 = 模型 / 版本层（回 3.2 与 5.3）。

## 附录 C：依据与关联文档

- **Ollama 官方 Codex CLI 集成文档**（docs.ollama.com/integrations/codex）：launch / `--oss` / profile 三种接法、`wire_api="responses"`、"Codex 建议 ≥64k 上下文"。
- **OpenAI Codex CLI 官方仓库与文档**（github.com/openai/codex；learn.chatgpt.com/codex/cli）：npm / 独立安装脚本（chatgpt.com/codex/install.sh）/ GitHub Release 二进制三种安装路线；当期版本 0.164.0（2026-10-23）；Windows 原生不支持，WSL 为官方 Linux 路径。
- **Ollama 模型库 qwen3-vl 页**（ollama.com/library/qwen3-vl）：全家族尺寸与量化、256K 上下文、Ollama ≥0.12.7 要求、`ollama launch` 应用矩阵、默认采样参数。
- **Ollama issue #13717**（2026-01）：Responses 端点图像输入缺位的记录，4.3 风险点来源。
- **《CC 方案》v1.6**：四参数方法论（6.2）、实测显存校准法（8.1）、PATH 抢占教训（第 10 章）。
- **Task 27 评估记录**（协作 worklog）：本地栈三硬缺口定谳（视觉输入 / 图像生成 / 指令遵循）、Codex+Ollama 社区配置三坑、001 源件"零图像生成需求"判定。
- **工作站 image-to-pptx 首战审计**（8 违规 + 根因链）：第 7 章五道硬门与违规的映射依据。
- **上游 profile**（`skills/ppt-master/workflows/profiles/image-to-pptx.md`）：§2 视觉检查、§5 清单、identity-faithful 分层规则、§6/§7 禁令——本手册第 7 章是其在本宿主的强约束投影。
- **测试记录 / 学习路径**：§12（工作站首战审计归档）与教训⑥⑦⑧外置落地以第 7 章为准；沙箱对照件见 `docs/ppt/AI落地方法论/`。

