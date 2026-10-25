# Claude Code WSL 部署与本地模型接入方案

> 文档版本：v1.4 ｜ 整理日期：2026-10-26
> 目标环境：Windows 10/11 + WSL2（Ubuntu 22.04 / 24.04）
> 接入对象：WSL 内**现有**的 Ollama（及 vLLM），模型 **Qwen3.5:9B**（工作站已预装）
> 变更记录：v1.4 6.4 方式二补全三档模型映射与 CLAUDE_CODE_MAX_CONTEXT_TOKENS（校准 CC 对自定义模型的 200k 假设），8.2 增第四坑，10 章增对应排查行，修正 6.5 hello.txt 预期字节数；v1.3 第 10 章新增“Claude Code 多份安装（Windows interop 抢占 PATH）”排查行；v1.2 6.2 节验证步骤按实机取证重写（environ 取证、加载快照坑、api/show 前缀键与原生上限辨析），8.1 回填 qwen3.5:9b@64k+q8_0 实测显存（KV 仅 ~1 GiB，混合注意力架构）；v1.1 驱动模型定版为 qwen3.5:9b（本机已预装，不设回退策略）；6.2 节补充四个 Ollama 参数的详细释义、三种配置方式与验证方法
> 配套文档：`../评估方案/PPT-Master评估与企业模板复用方案.md`、`../GitHub协作规则.md`
> 说明：本文所有命令除特别注明"Windows 侧"外，均在 WSL 终端内执行。

---

## 1. 目标与范围

### 1.1 部署目标

本方案要达成三件事：第一，在 WSL2 的 Ubuntu 环境中完成 Claude Code 的安装与初始化；第二，把 Claude Code 的模型后端从 Anthropic 云端切换到 WSL 内**已经存在**的本地推理服务——Ollama（首选）或 vLLM（团队共享场景），全程数据不出本机；第三，以 Qwen3.5:9B 为默认驱动模型（本机已预装，无需额外拉取），形成可跑通 PPT Master 工作流的完整本地 Agent 环境。

部署完成后应达到的验收标准：`claude --version` 正常输出；Claude Code 会话内回复、文件读写、命令执行、工具调用（tool use）四项能力全部可用；`/context` 显示的上下文窗口 ≥ 32k tokens；连续 30 分钟 Agent 会话无明显降速或断连。

### 1.2 两条接入路线（先看结论）

| | **路线 A：Ollama 直连（首选）** | **路线 B：vLLM + claude-code-router 桥接** |
|---|---|---|
| 原理 | Ollama **v0.14.0 起原生支持 Anthropic Messages API**，Claude Code 零桥接直连 | vLLM 只讲 OpenAI 格式，由 claude-code-router（CCR）翻译成 Anthropic 格式 |
| 适用 | 个人工作站、单机试点 | 团队共享 GPU、多用户并发、长驻服务 |
| 复杂度 | 低（两个环境变量 + 一条模型参数） | 中（需部署 CCR 并维护配置文件） |
| 工具调用 | Ollama 对 qwen3.5 自动处理 | vLLM 必须显式开启 `--enable-auto-tool-choice --tool-call-parser hermes` |
| 上下文 | `OLLAMA_CONTEXT_LENGTH` 显式调大 | 启动参数 `--max-model-len` 控制 |
| 选型建议 | **本机已有 Ollama → 直接走 A** | 已有 vLLM 长驻服务、或多人共用一块 GPU → 走 B |

两条路线可以共存：日常单人使用走 A，需要多人并发时再叠加 B，Claude Code 侧只需改一个环境变量即可切换。

---

## 2. 前置条件与环境检查

部署前逐项核对以下清单，任何一项不满足都应先处理再继续。检查命令全部在 WSL 内执行。

### 2.1 WSL2 本体

```bash
# Windows 侧 PowerShell 执行：
wsl --version          # WSL 版本 ≥ 2.x；"默认版本: 2" 
wsl -l -v              # 确认 Ubuntu 发行版 STATE=Running，VERSION=2
```

若仍是 WSL1 或未安装：`wsl --install -d Ubuntu-24.04`（Windows 11）或按微软文档升级。WSL1 没有 GPU 直通与 systemd，无法支撑本方案。

### 2.2 GPU 直通（NVIDIA）

```bash
nvidia-smi             # WSL 内执行，能输出显卡信息即直通正常
ls /usr/lib/wsl/lib/   # 应能看到 libcuda.so 等 CUDA 注入库
```

要求 Windows 侧已安装 NVIDIA 驱动（建议 ≥ 555 版本）；**WSL 内不要安装 Linux 显卡驱动**，只需 Windows 驱动 + WSL 自带的 CUDA 注入。`nvidia-smi` 报错时先更新 Windows 驱动，再 `wsl --shutdown` 后重进。AMD/Intel 显卡用户：Ollama 支持 ROCm/SYCL，vLLM 支持有限，建议此类环境只走路线 A 并实测验证。

### 2.3 现有 Ollama（关键：版本 ≥ 0.14.0）

```bash
ollama --version       # 必须 ≥ 0.14.0（该版本起原生提供 Anthropic 兼容 API）
ollama list            # 确认 qwen3.5:9b 已存在（本机已预装）
curl -s http://localhost:11434/   # 返回 "Ollama is running"
```

**版本不足是路线 A 的头号失败原因**。低于 0.14.0 时 Claude Code 无法直连（会报 `Unable to connect to Anthropic services` 或 404），执行下面命令升级后重试（模型与配置不受影响）：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### 2.4 现有 vLLM（仅路线 B 需要）

```bash
pip show vllm | head -2        # 确认已安装及版本（建议 ≥ 0.8）
nvidia-smi                     # 确认显存余量 ≥ 14 GiB（9B 模型 + 32k 上下文）
curl -s http://localhost:8000/v1/models   # 若 vLLM 已在跑，返回模型列表 JSON
```

### 2.5 基础工具

```bash
git --version    # ≥ 2.30（协作规则要求）
python3 --version # ≥ 3.10（ppt-master 要求，非本次部署硬依赖）
```

Node.js 仅在"npm 方式安装 Claude Code"或"部署 CCR"时需要（≥ 18，建议 22 LTS），见第 5、7 章；使用官方原生安装脚本且不走路线 B 时可不装 Node。

---

## 3. 架构与数据流

部署完成后的目标架构如下，所有组件同处一台机器的两个平面（Windows 宿主 + WSL），推理流量全程不出 WSL：

```text
┌─────────────────────────── Windows 11 宿主 ───────────────────────────┐
│                                                                        │
│  ┌────────────────────────── WSL2 (Ubuntu) ─────────────────────────┐  │
│  │                                                                  │  │
│  │  ┌──────────────────┐   Anthropic Messages API (HTTP)   ┌──────┐ │  │
│  │  │  Claude Code     │ ─────────────────────────────────► │Ollama│ │  │
│  │  │  (Agent 宿主)    │   http://localhost:11434           │:11434│ │  │
│  │  │  + ppt-master    │ ◄──────── 路线A：直连 ────────────  │qwen  │ │  │
│  │  │    skill         │                                    │3.5:9b│ │  │
│  │  └────────┬─────────┘                                    └──────┘ │  │
│  │           │            ┌──────────────────────┐                   │  │
│  │           │  路线B：    │  claude-code-router  │  OpenAI 格式      │  │
│  │           └──────────► │  (CCR, :3456)        │ ──────────────┐   │  │
│  │                        └──────────────────────┘               ▼   │  │
│  │                                                ┌─────────────────┐│  │
│  │                                                │ vLLM :8000      ││  │
│  │                                                │ Qwen3.5-9B 长驻 ││  │
│  │                                                └─────────────────┘│  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                   GPU（WSL 直通，CUDA on WSL）          │
└────────────────────────────────────────────────────────────────────────┘
```

要点三条：①Claude Code 与推理服务同在 WSL 内，走 `localhost` 回环，不涉及 WSL 端口转发问题；②路线 A 中 Ollama 同时提供 Anthropic 兼容端点（`/v1/messages`）与 OpenAI 兼容端点，Claude Code 只用前者；③路线 B 中 CCR 监听 3456 端口，对 Claude Code 伪装成 Anthropic API，对后端 vLLM 发 OpenAI 格式请求。
## 4. 阶段一：WSL 基础环境准备

### 4.1 启用 systemd（推荐）

WSL 内以 systemd 托管 Ollama / vLLM 服务可实现开机自启与 `systemctl` 管理。编辑 `/etc/wsl.conf`（无则创建）：

```ini
[boot]
systemd=true

[network]
generateResolvConf=true
```

Windows 侧 PowerShell 执行 `wsl --shutdown`，等待 8 秒后重新进入 WSL。验证：`systemctl list-units --type=service | grep ollama`。若 Ollama 是在启用 systemd 之前安装的，重跑一次官方安装脚本即可注册 systemd 服务。

### 4.2 资源配额（Windows 侧 `~/.wslconfig`）

WSL2 默认最多吃掉宿主 50% 内存。对 9B 模型场景，在 Windows 用户目录创建 `C:\Users\<你>\.wslconfig`：

```ini
[wsl2]
memory=24GB          # 建议宿主内存的一半以上；跑 64k 上下文时模型+KV cache 需 ~10GB
processors=8
swap=16GB

# 可选：镜像网络模式（Windows 11 22H2+），WSL 与宿主共享网络栈
# networkingMode=mirrored
```

改后需 `wsl --shutdown` 生效。若 GPU 推理正常则无需动 GPU 相关项。

### 4.3 时区与中文（可选，减少后续排障噪音）

```bash
sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

---

## 5. 阶段二：安装 Claude Code

### 5.1 方式一：官方原生安装（推荐，零依赖）

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

脚本会把二进制装到 `~/.local/bin/claude` 并自动写入 PATH。**不需要 Node.js**。安装后新开终端或 `source ~/.bashrc`，验证：

```bash
claude --version    # 输出版本号即成功
claude doctor       # 环境自检，确认安装健康
```

### 5.2 方式二：npm 安装（需要与 CCR 共存时可选）

```bash
# 需要 Node.js ≥ 18（建议 22 LTS，用 nvm 安装）
nvm install 22 && nvm use 22
npm install -g @anthropic-ai/claude-code
```

两种方式任选其一即可；已用原生方式安装的无需再装 npm 版本。后续升级统一用 `claude update`。

### 5.3 配置文件体系（认识目录结构）

Claude Code 的配置三级级联，优先级从低到高：`~/.claude/settings.json`（用户级）→ `.claude/settings.json`（项目级，入 git）→ `.claude/settings.local.json`（项目级个人覆盖，不入 git）。本方案把模型接入配置写在**用户级** `settings.json`（见 6.4 节），使所有项目默认走本地模型；涉及密钥的内容只放 `settings.local.json`，并遵守协作规则将 `.claude/` 加入 `.gitignore`。

---

## 6. 阶段三：路线 A —— Claude Code 直连 Ollama（首选路线）

### 6.1 原理回顾

Ollama 自 v0.14.0（2026-01-16 发布）起内置 Anthropic Messages 兼容层：`http://localhost:11434/v1/messages` 接收 Anthropic 格式请求，支持消息与多轮对话、流式输出、系统提示、工具调用、扩展思考与图像输入。Claude Code 不需要任何翻译层，改两个环境变量即可把"大脑"换成 qwen3.5:9b。

### 6.2 配置 Ollama 服务（关键参数一次到位）

Ollama 的**默认上下文只有 4096 tokens**，Claude Code 作为 Agent 工具一轮对话就可能消耗数千 tokens，不调大必然出现"模型读到一半忘前面"的症状。方案为 Ollama 设定以下 4 个核心环境变量，先弄清每个参数"控制什么、为什么这么设"：

| 环境变量 | 建议值 | 控制什么 | 为什么要设 |
|---|---|---|---|
| `OLLAMA_CONTEXT_LENGTH` | `65536` | 模型单次推理可见的**上下文窗口**（tokens） | 默认仅 4096；Claude Code 的系统提示 + 工具定义本身就要上万 tokens，超出的历史会被**静默截断**（不报任何错误），表现为"上一轮交代的事下一轮就忘"。四个参数中唯一"不设必出问题"的一个 |
| `OLLAMA_FLASH_ATTENTION` | `1` | 启用 **Flash Attention** 注意力算子 | 长上下文推理更快、显存峰值更低；同时是 KV cache 量化的**前置开关**——不开启时下一行 `q8_0` 的设定不生效 |
| `OLLAMA_KV_CACHE_TYPE` | `q8_0` | **KV cache 的量化精度**（默认 `f16`） | 64k 上下文的 fp16 KV cache 约 9 GiB，`q8_0` 直接压到约 4.5 GiB，回答质量损失几乎不可感知——这是 16 GB 显卡能开 64k 的关键。可选 `f16` / `q8_0` / `q4_0`；`q4_0` 再省一半但长文召回质量明显下降，Agent 场景不建议 |
| `OLLAMA_KEEP_ALIVE` | `2h` | 模型在**无请求后**的显存驻留时长（默认仅 5 分钟） | Claude Code 会话中请求间隔经常超过 5 分钟（人在阅读产出、思考下一步），超时后模型被卸出显存，下一次请求要重新加载数十秒；设 `2h` 让模型整个工作会话常驻显存，`-1` 表示永不卸载 |

另有一个可选项 `OLLAMA_NUM_PARALLEL=1`：并发请求处理数。每多一路并发，KV cache 显存近似翻倍；单机单人设 1，把显存全部留给上下文窗口。

**第一步｜确认 Ollama 装在哪一侧**（决定配置入口）：

```bash
which ollama    # WSL 内执行
```

- 输出 `/usr/local/bin/ollama` 等 Linux 路径 → 装在 **WSL 内**，用方式一；
- 输出以 `/mnt/c/…ollama.exe` 结尾 → 装在 **Windows 侧**（WSL 经 localhost 直通访问），用方式二。

**第二步｜按宿主位置选择配置入口（三选一）**

**方式一：WSL 内 systemd 服务（推荐，永久生效）**

```bash
sudo systemctl edit ollama
```

在打开的编辑器中写入（该命令生成 override 覆盖文件，不改动原始服务定义）：

```ini
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=65536"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_KEEP_ALIVE=2h"
Environment="OLLAMA_NUM_PARALLEL=1"
```

保存后重载生效：`sudo systemctl daemon-reload && sudo systemctl restart ollama`。

**方式二：Windows 侧 Ollama 桌面版**

PowerShell 执行（或用图形界面："系统属性 → 环境变量 → 用户变量"逐条新建）：

```powershell
setx OLLAMA_CONTEXT_LENGTH 65536
setx OLLAMA_FLASH_ATTENTION 1
setx OLLAMA_KV_CACHE_TYPE q8_0
setx OLLAMA_KEEP_ALIVE 2h
```

设置完成后**从系统托盘退出 Ollama 并重新启动**——Windows 下环境变量只对新启动的进程生效，不重启不生效。

**方式三：手动 `ollama serve`（临时 / 无 systemd）**

```bash
export OLLAMA_CONTEXT_LENGTH=65536   # 官方对 Agent 场景建议 64k+，最低 32k
export OLLAMA_FLASH_ATTENTION=1      # 开启 Flash Attention（KV 量化的前提）
export OLLAMA_KV_CACHE_TYPE=q8_0     # KV cache 8bit 量化，显存占用减半
export OLLAMA_KEEP_ALIVE=2h          # 模型常驻内存，避免 Agent 循环中反复重载
ollama serve
```

> ⚠️ 写在 `~/.bashrc` 里的 `export` 只对"手动敲 `ollama serve`"启动的进程生效，**不会**作用于 systemd 托管的 ollama 服务——这是"明明配了却没生效"的最常见原因。方式一与方式三二选一，不要混用。

**第三步｜验证配置已生效（按 ①→④ 顺序，① 是取证）**：

```bash
# ① 取证：变量是否真的进入了服务进程（最可靠的一步；无输出 = 没进服务进程，配置白设）
sudo cat /proc/$(pidof ollama)/environ | tr '\0' '\n' | grep OLLAMA

# ② 改完必须重载并冷启动：ollama ps 的 CONTEXT 是"加载时快照"，
#    不重启服务就复验，会被 4096 的旧实例假象骗到
sudo systemctl daemon-reload && sudo systemctl restart ollama
ollama run qwen3.5:9b "hi" >/dev/null 2>&1 && ollama ps
#    预期：CONTEXT = 65536，UNTIL ≈ 2 hours（KEEP_ALIVE 生效）

# ③ 日志核对：找 llama-server 启动参数与 KV cache 两行
journalctl -u ollama -n 400 --no-pager | grep -iE 'n_ctx_slot|type_k|type_v|q8_0|flash'
#    预期：-c 65536 --cache-type-k q8_0 --cache-type-v q8_0 --flash-attn on
#           llama_context: flash_attn = enabled
#           llama_kv_cache: ... K (q8_0): ... MiB, V (q8_0): ... MiB
#           n_ctx_slot = 65536

# ④ 模型元数据核对：model_info 的键带架构前缀（如 qwen35.context_length），
#    旧的 grep -o '"context_length":[0-9]*' 会因前缀漏匹配，改用 json 展开后模糊 grep
curl -s localhost:11434/api/show -d '{"model":"qwen3.5:9b"}' \
  | python3 -m json.tool | grep -iE 'context_length|num_ctx|"parameters"'
#    注意：context_length 在这里是 GGUF 原生上限（qwen3.5 = 262144），
#    不代表运行时生效值——运行时以 ② 的 CONTEXT 列与 ③ 的 n_ctx_slot 为准。
#    若 "parameters" 字段出现 num_ctx，说明 Modelfile 钉死了上下文，会盖过环境变量，
#    处理方式见 6.3 派生模型。
```

> 实测参考（RTX 5080 工作站，2026-10）：`ollama ps` → `6.9 GB ｜ 100% GPU ｜ 65536 ｜ 2 hours`；KV cache 实测 `llama_kv_cache: size = 1088.00 MiB (65536 cells, 8 layers), K (q8_0): 544.00 MiB, V (q8_0): 544.00 MiB`。

> ⚠️ 两条易误读的日志：
> ① `warmup: flash attention is enabled`——新版 Ollama 对支持的模型**默认开启 FA**，四个参数全部未生效时这条日志也会出现，**不能单独作为 OLLAMA_FLASH_ATTENTION 已生效的证据**，请以 ②③ 为准；
> ② `KV cache shifting is not supported` / `the context does not support partial sequence removal`——qwen3.5 是**混合注意力（全注意力层与线性/滑窗层交替）架构**，llama.cpp 对这类缓存不做平移与部分移除，改由 context checkpoints 机制兜底，属架构固有提示，**不是配置错误**。

### 6.3 确认模型就绪

```bash
ollama list                 # 确认 qwen3.5:9b 在列（本机已预装，无需拉取）
ollama show qwen3.5:9b      # 查看参数与上下文配置
```

如希望**按模型固定上下文**（而非全局环境变量），可用 Modelfile 派生一个专用模型：

```bash
cat > Modelfile <<'EOF'
FROM qwen3.5:9b
PARAMETER num_ctx 65536
EOF
ollama create qwen3.5:9b-64k -f Modelfile
# 之后 claude --model qwen3.5:9b-64k 使用
```

### 6.4 接入 Claude Code

**方式一：环境变量直连（先用手动方式验证）**

```bash
export ANTHROPIC_BASE_URL=http://localhost:11434
export ANTHROPIC_AUTH_TOKEN=ollama      # 客户端要求非空即可，Ollama 不校验值
claude --model qwen3.5:9b
```

> ⚠️ 认证变量只设 `ANTHROPIC_AUTH_TOKEN`，**不要**再设 `ANTHROPIC_API_KEY`，两者共存会触发 Auth conflict 报错。若之前配置过官方 Key，先 `claude auth logout` 并清掉旧变量。

**方式二：写入 settings.json（推荐，永久生效）**

编辑 `~/.claude/settings.json`：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:11434",
    "ANTHROPIC_AUTH_TOKEN": "ollama",
    "ANTHROPIC_MODEL": "qwen3.5:9b",
    "ANTHROPIC_SMALL_FAST_MODEL": "qwen3.5:9b",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "qwen3.5:9b",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "qwen3.5:9b",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "qwen3.5:9b",
    "CLAUDE_CODE_MAX_CONTEXT_TOKENS": "65536",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
```

说明：`ANTHROPIC_MODEL` 设默认主模型后，日常直接 `claude` 启动即可，不必每次带 `--model`；`ANTHROPIC_SMALL_FAST_MODEL` 与 `ANTHROPIC_DEFAULT_OPUS/SONNET/HAIKU_MODEL` 三档全部映射到本地模型——Claude Code 内部的后台任务、子代理会按这三档请求模型，不映射则可能落到云端或报模型不存在；`CLAUDE_CODE_MAX_CONTEXT_TOKENS=65536` 校准上下文窗口——CC 不识别自定义模型名时会默认按 **200k** 管理上下文（启动告警 `isn't recognized · context assumed to be 200k`），而真实窗口仅 64k，超限时 Ollama 会静默截断，此变量让 `/context` 与自动压缩阈值回归真实；`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` 关闭遥测与非必要流量，本地化场景建议开启。

**方式三：Ollama 官方快捷方式**

```bash
ollama launch claude    # 自动设置环境变量并启动 Claude Code，在交互选择器中选 qwen3.5:9b
```

### 6.5 分层验证（逐层通过再进入下一层）

```bash
# ① 服务层：Anthropic 端点连通性（直接以 Anthropic 格式发一条消息）
curl -s http://localhost:11434/v1/messages \
  -H 'content-type: application/json' \
  -d '{"model":"qwen3.5:9b","max_tokens":64,
       "messages":[{"role":"user","content":"只回答两个字：就绪"}]}'
# 期望返回 JSON，content 内含模型回复

# ② 接入层：Claude Code 能否走通
claude "请回复：连接成功"

# ③ Agent 层：工具调用是否生效（让 Claude Code 真的动手创建文件）
claude "在 /tmp 下创建 hello.txt，内容为 ppt-master-ok，然后告诉我文件大小"
ls -l /tmp/hello.txt   # 存在且 14 字节（ppt-master-ok 13 字符 + 换行）即工具链正常

# ④ 上下文层：会话内执行 /context，确认上下文窗口 ≥ 32k
```

四层全部通过，路线 A 部署完成。
## 7. 阶段四：路线 B —— vLLM + claude-code-router 桥接（团队共享 GPU）

### 7.1 启动 vLLM 服务（参数不可省）

```bash
vllm serve Qwen/Qwen3.5-9B \
  --host 0.0.0.0 --port 8000 \
  --served-model-name qwen3.5-9b \
  --max-model-len 32768 \
  --enable-auto-tool-choice --tool-call-parser hermes \
  --reasoning-parser qwen3 \
  --gpu-memory-utilization 0.90
```

| 参数 | 作用 | 缺省后果 |
|---|---|---|
| `--max-model-len 32768` | 上下文长度上限；显存富余可提至 65536（视模型的 YaRN 扩展支持） | 默认取模型配置，小显存机器可能 OOM |
| `--enable-auto-tool-choice` | 允许模型自动发起工具调用 | **Claude Code 的工具调用全部失效**，模型把工具调用当文本输出 |
| `--tool-call-parser hermes` | 指定 Qwen3 系列的工具调用解析器（hermes 格式） | 同上 |
| `--reasoning-parser qwen3` | 分离思考内容与正文，避免 thinking 混入回复 | CCR/客户端可能收到混杂思维链的输出 |
| `--served-model-name qwen3.5-9b` | 对外暴露的模型名，后续配置要与它一致 | 模型名过长易在 CCR 配置中写错 |

首次启动会从 HuggingFace 下载权重（9B 模型 bf16 约 18 GiB），国内网络建议先 `export HF_ENDPOINT=https://hf-mirror.com`。验证：`curl -s http://localhost:8000/v1/models`。

### 7.2 systemd 常驻（可选，团队服务器建议）

`/etc/systemd/system/vllm-qwen35.service`：

```ini
[Unit]
Description=vLLM Qwen3.5-9B server
After=network.target

[Service]
User=<你的用户名>
Environment="HF_ENDPOINT=https://hf-mirror.com"
ExecStart=/home/<你的用户名>/.local/bin/vllm serve Qwen/Qwen3.5-9B --host 0.0.0.0 --port 8000 --served-model-name qwen3.5-9b --max-model-len 32768 --enable-auto-tool-choice --tool-call-parser hermes --reasoning-parser qwen3 --gpu-memory-utilization 0.90
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

`sudo systemctl daemon-reload && sudo systemctl enable --now vllm-qwen35`。

### 7.3 安装并配置 claude-code-router

```bash
npm install -g @musistudio/claude-code-router
```

编辑 `~/.claude-code-router/config.json`：

```json
{
  "LOG": true,
  "API_TIMEOUT_MS": 600000,
  "Providers": [
    {
      "name": "vllm-qwen35",
      "api_base_url": "http://localhost:8000/v1/chat/completions",
      "api_key": "vllm",
      "models": ["qwen3.5-9b"]
    }
  ],
  "Router": {
    "default": "vllm-qwen35,qwen3.5-9b",
    "background": "vllm-qwen35,qwen3.5-9b",
    "think": "vllm-qwen35,qwen3.5-9b",
    "longContext": "vllm-qwen35,qwen3.5-9b",
    "longContextThreshold": 60000
  }
}
```

配置要点：`api_base_url` 必须是**完整的** `chat/completions` 地址；`api_key` 填任意非空串（本地 vLLM 不校验）；`Router` 把默认、后台、推理、长上下文四类场景全部路由到 qwen3.5-9b——本地只有这一个后端，若未来引入云端大模型，可把 `think`/`longContext` 档位指到更强模型，实现"规划用大模型、执行用本地小模型"的分层路由。

### 7.4 启动与使用（务必用 ccr 启动）

```bash
ccr start        # 启动路由网关（监听 127.0.0.1:3456）
ccr code         # 用 CCR 启动 Claude Code（关键！直接敲 claude 仍会连官方）
ccr ui           # 可选：浏览器打开 http://localhost:3456 可视化管理配置
```

> ⚠️ 最常见的"路线 B 失败"：装完 CCR 后仍然用 `claude` 命令启动，结果请求直连 Anthropic 官方。**必须 `ccr code`**；或执行 `eval "$(ccr activate)"` 后在当前终端用 `claude` 代替。

会话内可用 `/model vllm-qwen35,qwen3.5-9b` 动态切换模型；改完 `config.json` 用 `ccr restart` 生效。

### 7.5 验证

与 6.5 节相同的四层验证，仅第 ① 层改为验证 CCR 与 vLLM：

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"qwen3.5-9b","messages":[{"role":"user","content":"只回答两个字：就绪"}],"max_tokens":64}'
ccr status        # 路由网关运行中
ccr code          # 进入后执行 /context 与工具调用测试
```

---

## 8. Qwen3.5:9B 专项调优

### 8.1 显存预算表（决定你能开多大上下文）

Qwen3.5:9B 量化权重（q4_K_M）约 5.8 GiB；KV cache 每 token 约 144 KiB（fp16，按 36 层 × 8 组 × 128 维 × K/V 估算；不同架构层数/分组有差异，最终以 `ollama ps` 实测为准）：

| 配置 | 权重 | KV cache | 合计 | 建议显存 |
|---|---|---|---|---|
| 32k 上下文，fp16 KV | ~5.8 GiB | ~4.5 GiB | ~10.5 GiB | 12–16 GB |
| 32k 上下文，q8_0 KV | ~5.8 GiB | ~2.3 GiB | ~8 GiB | 12 GB |
| 64k 上下文，fp16 KV | ~5.8 GiB | ~9.0 GiB | ~15 GiB | 24 GB |
| 64k 上下文，q8_0 KV | ~5.8 GiB | ~4.5 GiB | ~10.5 GiB | **16 GB（推荐档位）** |
| 128k 上下文（YaRN），q8_0 KV | ~5.8 GiB | ~9.0 GiB | ~15 GiB | 24 GB |

结论：**16 GB 显存 + q8_0 KV 量化开 64k 上下文**是性价比最高的档位（本机 RTX 5080 16 GB 正处该档位）；12 GB 显存建议保守用 32k；24 GB 可尝试 128k 上下文或换 14B 级模型。

> **实测校准（RTX 5080 工作站，qwen3.5:9b q4_K_M + 64k + q8_0 KV，2026-10）**：KV cache 实测仅 **1088 MiB**（65536 cells × 8 层，K/V 各 544 MiB），进程总量实测 **6.9 GB**——qwen3.5 为混合注意力架构，全量 KV 只由少数全注意力层承载，上表按"36 层全注意力"的估算明显偏保守，16 GB 卡实际占用约 43%，余量比上表更大。GGUF 元数据原生上限 262144，后续如需扩到 128k 显存依然充裕；但 64k 已覆盖 Claude Code + ppt-master 场景且 prefill 更快，推荐档位维持不变。

### 8.2 上下文相关的四个坑

1. **Ollama 默认 num_ctx=4096**：症状是"上一轮交代的事下一轮就忘"，且不报任何错误（旧上下文被静默丢弃）。解法见 6.2/6.3 节，设完用会话内 `/context` 复核。
2. **32k 是及格线不是目标**：Claude Code 一轮 Agent 会话（系统提示 + 文件内容 + 工具输出）轻松消耗上万 tokens，跑 ppt-master 时还要叠加技能文档，64k 起步体验才完整。
3. **上下文吃满的行为**：接近上限时 Claude Code 会自动压缩（compact）历史，压缩本身也消耗一次推理；模型越小压缩损失越大，重要任务前先 `/clear` 开新会话。
4. **Claude Code 对自定义模型名按 200k 假设**：启动告警 `Model "qwen3.5:9b" isn't recognized · context assumed to be 200k tokens`，`/context` 显示的 200k 是假象——真实仅 64k，按假象用下去必然超限截断（又回到“忘事”症状）。解法：settings.json 的 env 加 `"CLAUDE_CODE_MAX_CONTEXT_TOKENS": "65536"`（6.4 方式二已含），重启后 `/context` 的 Auto-compact window 应显示 65536。

### 8.3 思考模式与响应速度

Qwen3.5 沿用 Qwen3 系列的思考（thinking）/非思考双模式设计，Ollama 默认开启思考。思考模式提升推理质量但显著拉长首 token 延迟，PPT Master 的模板消费（Edit Native PPTX 路线）建议保持默认；若追求响应速度或模型出现"卡在思考态"（小模型偶发），可在对话中让模型关闭思考（提示词加 `/no_think`），或在 CCR 层统一处理。

### 8.4 与 PPT Master 配合的边界（引自评估文档 3.5 节）

- 优先走 **Edit Native PPTX** 路线，对 9B 级本地模型最友好；
- 单次任务控制在 5–8 页，长材料先分段提炼；
- 本地 9B 模型不产图，配图用网络图源或自带素材；
- 模板提炼（Create Template）这类一次性的高难度任务，建议临时把 `ANTHROPIC_BASE_URL` 切到云端大模型完成，产出入库后日常消费仍走本地。

---

## 9. 部署验收清单

### 9.1 分层验收表

| 层级 | 检查项 | 命令 / 方法 | 通过标准 |
|---|---|---|---|
| 环境 | WSL2 + systemd | `wsl -l -v`；`systemctl is-system-running` | VERSION=2；返回 running/degraded |
| 环境 | GPU 直通 | `nvidia-smi` | 正常输出显卡与显存 |
| 服务 | Ollama 版本 | `ollama --version` | ≥ 0.14.0 |
| 服务 | 上下文配置 | `curl -s localhost:11434/api/show -d '{"model":"qwen3.5:9b"}'` | context_length ≥ 32768 |
| 服务 | 模型驻留 | `ollama ps` | qwen3.5:9b 加载中，Processor 显示 GPU 占比 |
| 接入 | Anthropic 端点 | 6.5 节 curl | 返回含回复的 JSON |
| 接入 | 认证变量唯一 | `env \| grep ANTHROPIC` | 只有 AUTH_TOKEN，无 API_KEY |
| Agent | 工具调用 | 6.5 节第 ③ 层 | 文件真实创建 |
| 业务 | ppt-master 冒烟 | 9.2 节 | 产出 .pptx 可在 PowerPoint 打开 |

### 9.2 ppt-master 冒烟测试（最终业务验收）

```bash
# 在 WSL 内
git clone https://github.com/hugohe3/ppt-master.git && cd ppt-master
pip install -r requirements.txt
claude    # 在 ppt-master 目录启动（配置已指向本地模型）
```

会话内输入：`用 Edit Native PPTX 流程，把 projects/ 下任一示例 PPTX 的标题页文字替换为"部署验收测试"`。能走完"解析 → 选页 → 回填 → 导出"闭环且产出文件可正常打开，即整条链路验收通过。建议按协作规则把测试记录写入 `docs/测试记录-<日期>-WSL部署.md` 并提交。

---

## 10. 故障排查速查表

| 症状 | 原因 | 处置 |
|---|---|---|
| `Unable to connect to Anthropic services` | 未设 `ANTHROPIC_BASE_URL`；或 Ollama < 0.14.0 无 Anthropic 端点；或 CCR 场景没用 `ccr code` 启动 | 检查环境变量；升级 Ollama；改用 `ccr code` |
| `Auth conflict` 报错 | `ANTHROPIC_API_KEY` 与 `ANTHROPIC_AUTH_TOKEN` 同时存在 | `claude auth logout`，清掉 `ANTHROPIC_API_KEY`，只留 `AUTH_TOKEN` |
| 401/403/连接被拒 | 端口/地址错；Ollama 未监听 | `curl -s localhost:11434`；`lsof -i :11434` 查占用 |
| **响应越来越慢（10s→80s 递增）** | 已知 bug（Ollama #13949）：Claude Code 周期性调用 `/v1/messages/count_tokens`，旧版 Ollama 不支持且反复重试拖垮请求处理 | 重启 `ollama serve` 可临时恢复；升级 Ollama 至修复版本 |
| 模型"忘事"、答非所问 | 上下文仍为默认 4096 | 按 6.2 设 `OLLAMA_CONTEXT_LENGTH`，Modelfile 方式二选一 |
| **四参数全部未生效**（ps 仍 4096、UNTIL 仍 5 分钟） | 环境变量没进 systemd 服务进程（bashrc/终端 export 对服务无效，或 override 漏了 daemon-reload/restart） | 取证：`sudo cat /proc/$(pidof ollama)/environ \| tr '\0' '\n' \| grep OLLAMA`；再按 6.2 方式一写 override.conf 并 `daemon-reload + restart`；旧实例是加载时快照，须重载后再验 |
| `claude` 执行的版本不对 / doctor 报 Multiple installations（Invoked 指向 /mnt/c/...） | Windows 侧 npm 装过 Claude Code，其 shim 经 WSL interop 抢占 PATH | 统一保留 native：`echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc`，再 `claude install` 对齐 install method；Windows PowerShell 删 `%APPDATA%\npm\node_modules\@anthropic-ai\claude-code` 及 `claude`、`claude.cmd`、`claude.ps1` 三个 shim（npm 不可用时直接删文件等价） |
| 启动告警 `Model ... isn't recognized · context assumed to be 200k`，/context 显示 200k | Claude Code 不识别自定义模型名，默认按 200k 管理上下文，而真实窗口仅 64k，超限即被 Ollama 静默截断 | `~/.claude/settings.json` env 加 `"CLAUDE_CODE_MAX_CONTEXT_TOKENS": "65536"` 后重启会话；顺带把 `ANTHROPIC_DEFAULT_OPUS/SONNET/HAIKU_MODEL` 三档映射到本地模型，防后台任务落到云端 |
| 推理极慢、CPU 拉满 | GPU 未生效 | `ollama ps` 看 Processor 列；检查 `nvidia-smi`；更新 Windows 显卡驱动后 `wsl --shutdown` |
| vLLM 工具调用失效（模型把工具调用当文字输出） | 缺 `--enable-auto-tool-choice --tool-call-parser hermes` | 补参数重启 vLLM |
| CCR 报 upstream 超时 | vLLM 长思考超时 | `config.json` 中调大 `API_TIMEOUT_MS`（如 600000） |
| vLLM 下载权重缓慢 | HuggingFace 直连慢 | `export HF_ENDPOINT=https://hf-mirror.com` |
| WSL 重启后服务全掉 | systemd 未启用或服务未设自启 | 4.1 节启用 systemd；`systemctl enable --now ollama` |
| 提示 prompt too long | 超过 `--max-model-len` 或 Ollama num_ctx | 调大上限；或 `/compact`、`/clear` 后重试 |

---

## 11. 附录

### 11.1 附录 A：路线 A 一键脚本（可保存为 setup-claude-wsl.sh）

```bash
#!/usr/bin/env bash
set -euo pipefail
echo "== [1/5] 安装 Claude Code =="
command -v claude >/dev/null || curl -fsSL https://claude.ai/install.sh | bash
echo "== [2/5] 确保 Ollama >= 0.14.0 =="
ver=$(ollama --version 2>/dev/null | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1 || echo 0)
[[ "${ver%%.*}" -ge 14 ]] || curl -fsSL https://ollama.com/install.sh | sh
echo "== [3/5] 写入 Ollama 配置（systemd 服务则写 override，否则写 bashrc）==="
if systemctl is-active --quiet ollama 2>/dev/null; then
  mkdir -p /etc/systemd/system/ollama.service.d
  cat <<'EOF' | sudo tee /etc/systemd/system/ollama.service.d/override.conf >/dev/null
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=65536"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_KEEP_ALIVE=2h"
EOF
  sudo systemctl daemon-reload && sudo systemctl restart ollama
else
  grep -q OLLAMA_CONTEXT_LENGTH ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export OLLAMA_CONTEXT_LENGTH=65536
export OLLAMA_FLASH_ATTENTION=1
export OLLAMA_KV_CACHE_TYPE=q8_0
export OLLAMA_KEEP_ALIVE=2h
EOF
fi
echo "== [4/5] 写入 Claude Code 用户级配置 =="
mkdir -p ~/.claude
python3 - <<'EOF'
import json, pathlib
p = pathlib.Path.home()/'.claude'/'settings.json'
cfg = json.loads(p.read_text()) if p.exists() else {}
cfg.setdefault('env', {}).update({
    "ANTHROPIC_BASE_URL": "http://localhost:11434",
    "ANTHROPIC_AUTH_TOKEN": "ollama",
    "ANTHROPIC_MODEL": "qwen3.5:9b",
    "ANTHROPIC_SMALL_FAST_MODEL": "qwen3.5:9b",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"})
p.write_text(json.dumps(cfg, indent=2, ensure_ascii=False))
EOF
echo "== [5/5] 确认模型并提示验证 =="
ollama pull qwen3.5:9b    # 本机已预装时为秒级校验
echo "完成。请运行: claude '请回复：连接成功' 验证；再执行 /context 复核上下文窗口。"
```

### 11.2 附录 B：环境变量速查

| 变量 | 侧 | 示例值 | 作用 |
|---|---|---|---|
| `ANTHROPIC_BASE_URL` | Claude Code | `http://localhost:11434` | 指向本地 Ollama（路线 A）；CCR 场景指向 `http://localhost:3456` |
| `ANTHROPIC_AUTH_TOKEN` | Claude Code | `ollama` | 非空即可，本地服务不校验 |
| `ANTHROPIC_MODEL` | Claude Code | `qwen3.5:9b` | 默认主模型，须与 `ollama list` 完全一致 |
| `OLLAMA_CONTEXT_LENGTH` | Ollama | `65536` | 默认上下文长度，Agent 场景 64k+ |
| `OLLAMA_FLASH_ATTENTION` | Ollama | `1` | Flash Attention，KV 量化前提 |
| `OLLAMA_KV_CACHE_TYPE` | Ollama | `q8_0` | KV cache 量化，显存减半 |
| `OLLAMA_KEEP_ALIVE` | Ollama | `2h` | 模型驻留时长，避免反复重载 |
| `OLLAMA_HOST` | Ollama | `0.0.0.0:11434` | 仅当需从其他机器访问 WSL 服务时设置 |
| `HF_ENDPOINT` | vLLM | `https://hf-mirror.com` | 国内加速权重下载 |

> Ollama 侧四个参数（`OLLAMA_CONTEXT_LENGTH` / `OLLAMA_FLASH_ATTENTION` / `OLLAMA_KV_CACHE_TYPE` / `OLLAMA_KEEP_ALIVE`）的详细含义与三种配置方式见 6.2 节。

### 11.3 附录 C：参考资料

- Ollama 官方博客《Claude Code with Anthropic API compatibility》（v0.14.0 特性公告）
- Claude Code 官方安装文档：`https://claude.ai/install.sh`
- claude-code-router：`https://github.com/musistudio/claude-code-router`
- vLLM 文档：Tool Calling / Qwen3 支持页
- 本团队《PPT-Master评估与企业模板复用方案》第 3 章（选型依据）与《GitHub 协作规则》（提交规范）


