# PPT Master 冒烟测试示例选型

> 文档版本：v1.0 ｜ 整理日期：2026-10-26
> 适用读者：正在执行《学习路径》阶段 1 或《部署方案》9.2 冒烟测试的工作站用户（Claude Code + Ollama + qwen3.5:9b）
> 数据基线：`hugohe3/ppt-master-examples` @ `3b8ce2c`（examples.json v1，72 个含导出件的示例，全库 1.6 GB）
> 配套文档：`PPT-Master企业级PPT学习路径.md`（阶段 1 任务书）、`../部署方案/Claude-Code-WSL部署与本地模型接入方案.md`（9.2 冒烟流程）

---

## 1. 这份文档解决什么问题

冒烟测试只需要"一份合适的现有 PPTX"，但 examples 仓库 72 个示例的规格差异极大：页数从 6 页到 31 页不等，导出件体积从 0.04 MB 到 10.95 MB 横跨三个数量级，画幅既有标准 16:9 也有微信横幅、朋友圈卡片、小红书图文等特殊规格，内容语言覆盖中英日韩印地等近十种。如果随手选了超大图卷、带音频的叙述版或非标准画幅示例，9B 本地模型很可能在第一次真实任务上就撞上上下文溢出或管线异常——问题不在你的部署，而在示例本身不适合当第一块试金石。

本文从 72 个含导出 PPTX 的示例中，按统一标准筛出**主推三例（按"最稳 → 最快 → 最衔接企业场景"排序）与专项候选四例**，并给出首轮应避开的类型清单。全部结论基于对每个示例的页数（`svg_final/` 计数）、导出件体积、首页实际文字（语言核对）、资产构成（音频/源件/图表变体）的逐项扫描，非主观推荐。选定示例后，冒烟提示词与验收标准直接沿用《学习路径》阶段 1 原版，本文不重复展开流程细节。

## 2. 选型标准（五个维度与依据）

| 维度 | 门槛 | 依据 |
|------|------|------|
| 页数 | ≤ 16 页为宜 | 《部署方案》8.2 实测：65.5k 窗口扣除 autocompact buffer 后真实对话余量约 32k，Edit Native PPTX 单页 SVG 可达数千 token，卷越小往返越稳 |
| 导出件体积 | ≤ 1 MB 为宜 | 体积近似线性决定解包、SVG 提取、重打包三段耗时，也间接反映图片/媒体数量 |
| 内容语言 | 中文优先 | 验收时肉眼对照直观：标题是否换成"部署验收测试"一目了然，无需翻译回查 |
| 画幅 | 标准 16:9 | 主流程与内置布局词汇表均以 16:9 为一等公民；横幅/卡片类走特殊画幅，不适合首战 |
| 媒体复杂度 | 无音频、少动效依赖 | 叙述型（narrated）示例带 TTS 音频产物，图表原生版含 charts/tables 变体——首轮引入这些只增加变量，不增加信息量 |

一句话概括：**冒烟要的是最小变量，不是最漂亮的 deck**。功能特性（图表、音频、超链接、长卷）留给第二轮之后的专项验证，每个特性单独一卷，出了问题才能快速定位到特性本身。

## 3. 主推三例

### 3.1 首选：都江堰（最稳）

| 项 | 值 |
|----|-----|
| 路径 | `examples/ppt169_dujiangyan_zh_sparse/` |
| 导出件 | `exports/ppt169_dujiangyan_zh_sparse.pptx`（14 页，0.73 MB） |
| 内容 | 中文科普《都江堰：一座还在上班的两千岁工程》，Default 路线 |
| 特点 | 仓库首个"稀疏输入"示例：设计刻意克制、页面装饰元素少 |

**推荐理由**：稀疏设计意味着单页 SVG 的对象数量少，往返（pptx→SVG→pptx）时 9B 模型需要"看懂再改"的文本量最小，是把"解析 → 选页 → 回填 → 导出"闭环跑通成功率最高的一卷。中文科普题材独立成篇，不依赖企业语境，标题页主标题"都江堰"三个字位置醒目，替换后极易验收。如果你只测一卷，测它。

**适配提示词**（沿用《学习路径》阶段 1 模板，仅替换路径）：

```text
用 ppt-master 的 Edit Native PPTX 流程处理 projects/ppt169_dujiangyan_zh_sparse.pptx：
只把第 1 页（标题页）的主标题文字替换为"部署验收测试"，
其余所有页面逐字节保留，导出为新文件，不要覆盖源文件。
完成后告诉我工作区路径和导出文件路径。
```

### 3.2 次选：WebGPU 入门（最快）

| 项 | 值 |
|----|-----|
| 路径 | `examples/ppt169_webgpu_intro_quick/` |
| 导出件 | `exports/ppt169_webgpu_intro_quick.pptx`（12 页，0.07 MB） |
| 内容 | 中文技术入门《WebGPU 是什么》，Quick 路线，暗色科技风 |
| 特点 | 全库体积最小的 16:9 标准卷之一，解包重打包秒级完成 |

**推荐理由**：0.07 MB 的体积让管线三段（解包/提取/重打包）都快到可以忽略，适合在部署验收当天反复跑多轮——第一次失败后改完配置马上重试，不用等。暗色科技风附带 7 段等宽代码块，顺带验证等宽字体在 roundtrip 中的保真，属于"免费附赠"的检查点。

### 3.3 第三：中国电信业绩解读（最衔接阶段 2）

| 项 | 值 |
|----|-----|
| 路径 | `examples/ppt169_china_telecom_2025/` |
| 导出件 | `exports/ppt169_china_telecom_2025.pptx`（14 页，0.39 MB）；另有 `_native_charts_tables` 变体 |
| 内容 | 中文商务《中国电信 2025 年全年业绩解读》，Quick 路线，套用内置"中国电信" Deck 模板 |
| 特点 | 内置 21 个企业品牌模板的实际应用样本，含原生图表版变体 |

**推荐理由**：它是《学习路径》阶段 2"企业模板 + 真实材料"的预演——用的正是内置品牌模板体系。冒烟时先用基础版跑标题替换；通过后再用 `_native_charts_tables` 变体跑一遍，顺带观察含原生图表的页面在 roundtrip 后是否原样。一步过渡到阶段 2，不用换示例库。

## 4. 专项候选（第二轮起按特性单测）

| 示例 | 专项 | 规格 | 说明 |
|------|------|------|------|
| `ppt169_china_economy_2025_briefing` | 原生图表/表格 | 17 页 0.11 MB | 10 张原生图表（6 种类型）+ 3 张原生表，数据简报风格，验证 charts/tables 的 SVG 往返保真 |
| `ppt43_fourier_transform_formulas_zh` | 4:3 画幅 + 公式 | 15 页 0.15 MB | 全库首个原生公式示例，4:3 画布，验证非 16:9 画幅与公式对象的编辑表现 |
| `ppt169_jilin_geohazard_rules_beautify` | 有源文件美化 | 20 页 0.59 MB | 全库唯一带 `sources/` 源 PPTX 的示例，对应"用户已有 PPTX 美化"的真实业务场景 |
| `ppt169_infosec_training_zh_long` | 长卷一致性 | 28 页 0.17 MB | 仓库最长卷，5 章结构 + 目录页码 + 随堂自测，压测多页批量改动时的一致性 |
| `ppt169_sanxingdui_kiosk_zh` | 超链接往返 | 16 页 1.33 MB | 仓库首个原生超链接示例（导览菜单跳转），验证 hlinkClick 在 roundtrip 后仍可点 |

## 5. 首轮请避开的类型

以下三类示例本身质量没有问题，只是不适合当第一块试金石，建议按需后置：

- **非标准画幅**：`banner_autumn_solar_terms`、`custom_1920x600_autumn_web_banner`、`wechat_typst_series_banners`、`wechat_tea_header_set`、`moments_*`、`xiaohongshu_*`、`story_*`、`a4_*` 等横幅、朋友圈卡片、小红书图文与 A4 备忘录。它们的画幅逻辑与 16:9 主流程不同，首轮引入只会混淆"是流程问题还是画幅问题"。
- **超大图卷**：`ppt169_qianli_jiangshan_scroll`（10.95 MB）、`moments_nine_heritage_goods`（10.58 MB）、`ppt169_mid_autumn_papercut`（6.66 MB）、`ppt169_pixel_breakfast_atlas`（6.81 MB）等。图片资产密集，解包与重打包慢，且 9B 模型的上下文会被无关图像数据挤占。
- **带音频的叙述版**：`ppt169_beidou_navigation_narrated`、`ppt169_pixel_breakfast_atlas` 含 TTS 旁白产物。音频不属于冒烟范围，混入只会多一个失败源。

另提示一点：本库示例与主仓库模板资产（`templates/brands/` 21 个品牌、`styles/` 14 套风格）是一一对应关系的活样本——读到《学习路径》阶段 4"模板资产化"时，可以把示例当"这套模板长什么样"的实景参照，不必再凭文档想象。

## 6. 示例获取方式

沙箱与工作站的处理策略不同，按需选择：

**工作站已整库克隆**（1.6 GB，75 个项目全部就位）：

```bash
cd ~/sourcecode/ppt-master-examples && git pull
cp -r examples/ppt169_dujiangyan_zh_sparse/exports/*.pptx  ~/sourcecode/ppt-master/projects/
```

**工作站只需个别示例**（推荐：单项目即取即用，免下整库）：

```bash
# 稀疏检出单个项目目录（约几 MB）
git clone --filter=blob:none --no-checkout https://github.com/hugohe3/ppt-master-examples.git
cd ppt-master-examples
git sparse-checkout set examples/ppt169_dujiangyan_zh_sparse
git checkout
cp examples/ppt169_dujiangyan_zh_sparse/exports/*.pptx ../ppt-master/projects/
```

复制时注意《部署方案》9.2 的约定：**源 PPTX 复制进 `projects/` 后保持只读心理——任何编辑产出都导出新文件，不覆盖源件**。这是 Edit Native PPTX 工作流的第一守则，从冒烟就养成。

## 7. 附录：轻量示例速查表

页数 ≤ 16 且导出件 ≤ 1 MB 的示例全收录（按体积升序），供第二轮自选。语言为首页文字实核结果：

| 示例 | 页数 | MB | 语言 | 题材备注 |
|------|-----|------|------|----------|
| `ppt169_webgpu_intro_quick` ★ | 12 | 0.07 | 中 | WebGPU 技术入门，暗色+代码块 |
| `ppt169_apple_fy2025_review` | 14 | 0.09 | 中 | 苹果 FY2025 经营复盘，财务数据 |
| `ppt169_cloudflare_outage_postmortem` | 15 | 0.09 | 英 | 故障复盘报告 |
| `ppt169_telford_feedback_2023_24_quick` | 13 | 0.10 | 英 | 政务反馈简报 |
| `ppt169_solar_cheapest_power` | 15 | 0.10 | 中 | 光伏成本科普，数据图文 |
| `ppt169_uv_launch` | 14 | 0.11 | 中 | Python 包管理工具发布 |
| `ppt169_ev_market_priorities` | 14 | 0.13 | 中 | 电动车市场决策讨论稿 |
| `ppt169_kubernetes_blueprint_2026` ★ | 13 | 0.13 | 中 | K8s 集群架构图解，等距技术风 |
| `ppt169_china_economy_2025_briefing` ★ | 17 | 0.11 | 中 | 经济数据简报，图表专项（第 4 节） |
| `ppt43_fourier_transform_formulas_zh` ★ | 15 | 0.15 | 中 | 傅里叶公开课，4:3+公式（第 4 节） |
| `ppt169_every_day_counts_proposal` | 14 | 0.26 | 英 | 教育提案 |
| `ppt169_swiss_grid_systems` | 14 | 0.26 | 中 | 瑞士网格系统讲座 |
| `ppt169_sqlite_everywhere` | 12 | 0.31 | 英 | SQLite 技术分享，图纸风 |
| `ppt169_china_telecom_2025` ★ | 14 | 0.39 | 中 | 企业模板业绩汇报（主推第三） |
| `ppt169_glassmorphism_demo` | 12 | 0.65 | 中 | AI Agent 工程化，毛玻璃风 |
| `ppt169_dujiangyan_zh_sparse` ★ | 14 | 0.73 | 中 | 都江堰科普，稀疏设计（主推第一） |
| `ppt169_fire_control_room_duty_manual` | 16 | 0.76 | 中 | 消防值班手册，培训课件 |

★ = 本文正文推荐的卷目。超出本表的大卷、非标准画幅与多语言卷（日/韩/泰/越/阿拉伯/印地/繁中）可按第 5 节原则自行排期。
