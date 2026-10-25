# PPT Master 企业级 PPT 学习路径

> 文档版本：v1.2 ｜ 整理日期：2026-10-26 ｜ 变更：v1.2 阶段 2 回填 20261026 首战复盘（输入先验/一页一会话/确认关卡/三条铁律/备注正确机制，指向测试记录）；v1.1 阶段 1 接入《冒烟测试示例选型》；v1.0 初版
> 适用读者：已完成《部署方案》路线 A 部署并通过四层验收的企业工作站用户（Claude Code + Ollama + qwen3.5:9b）
> 目标：用最短时间从"环境就绪"走到"稳定产出企业级 PPT"
> 配套文档：`../部署方案/Claude-Code-WSL部署与本地模型接入方案.md`（环境）、`../评估方案/PPT-Master评估与企业模板复用方案.md`（选型依据）

---

## 1. 文档全景与关联关系（先看地图再上路）

上游仓库的文档分三层，**"给人读的"在 `docs/`，"给 AI 读的"在 `skills/ppt-master/`**。英文 `docs/*.md` 是规范源，`docs/zh/` 是同步中文译本（内容一一对应，建议直接读中文版）。

```text
┌─ 第一层：产品文档（人读，docs/zh/）──────────────────────────┐
│                                                              │
│  理念层（为什么）          操作层（怎么做）        原理层（边界）  │
│  ├ what-is-ppt            ├ getting-started ★   ├ technical-design│
│  ├ why-ppt-master         ├ templates-guide ★   ├ powerpoint-svg-mapping│
│  ├ project-positioning    ├ faq ★               ├ templates-architecture│
│  └ roadmap                ├ animations          └ windows-installation│
│                           └ audio-narration                        │
└──────────────────────────────────────────────────────────────┘
                    │ 运行时由 AI 消费（第二、三层）
┌─ 第二层：路线判定权威（skills/ppt-master/workflows/）─────────┐
│  SKILL.md（强制加载顺序：attribution_guard → routing.md →     │
│            选定唯一路线 → 只读该路线 runtime authority）       │
│  routing.md ★（三条顶级路线的判定权威，冲突时以它为准）：      │
│    Generate PPTX   —— 从零/从材料生成新设计                    │
│    Edit Native PPTX —— 保住现有 PPTX 设计，改内容/加备注动画 ★ │
│    Create Template —— 把现有材料提炼为可复用模板资产           │
│  profiles: quick-generate（快速直出）/ beautify-pptx（1:1美化）/│
│            image-to-pptx（图片重建）—— 是剖面不是第四条路线     │
└──────────────────────────────────────────────────────────────┘
┌─ 第三层：运行时参考与内置资产（AI 按需加载）──────────────────┐
│  workflows/stages/*（topic-research / verify-charts /        │
│    visual-review / live-preview / customize-animations /     │
│    generate-audio / resume-execute 等阶段文档）               │
│  workflows/create-template/{brand,style,layout,deck}.md      │
│  references/*（约 90 篇 executor/strategist 参考，人不必读）   │
│  templates/ 内置资产库：                                      │
│    brands×21：mckinsey/bcg/bain/deloitte/pwc/accenture/      │
│      jpmorgan/ibm/google/microsoft/aws/nvidia/alibaba/       │
│      tencent/huawei/xiaomi/中国电信/中国电建/中汽研 等         │
│    styles×14：mbb-consulting/consulting-decision/            │
│      solution-proposal/investor-pitch/product-launch/        │
│      operating-review/technical-deepdive/incident-postmortem 等│
│    layouts×7：presentation_core(_43)/report_core/            │
│      editorial_bleed/story_vertical/moments_square/…         │
│    decks×2：中国电信、中汽研（整套成品模板）                    │
│    charts/tables/icons/sounds 词汇表 + 各 *_index.json        │
│  scripts/docs/*（工具手册：svg-pipeline/project/narration/    │
│    troubleshooting 等，排障时查）                             │
└──────────────────────────────────────────────────────────────┘
```

### 1.1 关联关系五条主线

1. **操作总入口**：`getting-started.md` 是第一份 deck、模板用法、快速模式、实时预览、动画、旁白的总教程；它在"用模板"一节指向 `templates-guide.md`。
2. **路线判定双轨**：给人看的判定速查表在 `technical-design.md`（路线判定速查表）与 `templates-guide.md`（60 秒选对模板路线）；**权威版本**是 `skills/ppt-master/workflows/routing.md`——两者冲突时 routing.md 赢，且 Agent 每次任务都强制先读它。
3. **模板三文档递进**：`templates-guide.md`（怎么选、怎么派生）→ `templates-architecture.md`（Brand/Style/Layout/Deck 四分类与 design_spec 结构）→ `routing.md` §5（四分类判定硬规则：身份稳定选 Brand、沟通方法可迁移选 Style、结构品牌无关选 Layout、结构承载身份选 Deck）。
4. **能力边界双权威**：某个 PowerPoint 功能管线支不支持，查 `powerpoint-svg-mapping.md`（逐项映射表，含表格/图表/公式/动画）；整个功能做不做，查 `roadmap.md`（能力覆盖地图与"明确不做"清单）与 `project-positioning.md`（能力准入判据）。
5. **排坑三级跳**：日常问题先查 `faq.md`（真实用户反馈持续更新）→ 脚本级问题查 `skills/ppt-master/scripts/docs/troubleshooting.md` → 本地环境问题查《部署方案》第 10 章故障排查速查表。

### 1.2 你只需要精读的 5 篇

其余文档当字典查。精读清单（按阅读顺序）：`getting-started` → `templates-guide`（§60 秒 + §选用）→ `what-is-ppt`（§5–§7）→ `faq` → `technical-design`（§路线判定速查表）。原理层（powerpoint-svg-mapping / templates-architecture）与运行时层（skills/）在学习路径各阶段按需带入。

---

## 2. 学习路径总览（任务驱动，四阶段）

| 阶段 | 时间 | 目标 | 主读文档 | 实战任务 |
|---|---|---|---|---|
| 阶段 0 | 已完成 | 环境就绪 | 部署方案 v1.6 | 四层验收全绿 |
| **阶段 1** | 第 1 天，约 1–2 小时 | 跑通最小闭环 | getting-started；technical-design §速查表 | Edit Native PPTX 冒烟：改标题页 |
| **阶段 2** | 第 2–3 天 | 企业模板 + 真实材料主力路线 | templates-guide §选用；what-is-ppt §5–7；edit-native-pptx §2/§4 | 公司模板产出 5–8 页完整 deck（含备注、删页/排序） |
| **阶段 3** | 第 4–5 天 | 企业级质量进阶 | faq；animations；audio-narration；powerpoint-svg-mapping §7–8 | 转场+动画+旁白；内置模板快速生成对照版 |
| **阶段 4** | 按需 | 模板资产化 | templates-guide §派生；templates-architecture；routing §5 | 企业 .pptx 提炼为模板资产入库 |

每阶段结构：**读什么（精确到节）→ 做什么（提示词模板见 §3）→ 验收标准 → 产物**。逐阶段通过再进入下一阶段，与部署方案"分层验证"同一哲学。

---

## 3. 阶段详解与提示词模板

### 阶段 1 · 最小闭环（第 1 天）

**读**：`docs/zh/getting-started.md` 全文（重点：配置安装目录与工作目录、做出第一份 deck、快速模式三节，约 15 分钟）；`technical-design.md` 的"路线判定速查表"一节（约 5 分钟）。建立三个核心概念：`projects/<name>` 是每个任务的工作区、最终交付在 `projects/<name>/exports/`、三条路线按"要不要保留原设计"来分。

**做**：准备任意一份源 PPTX——**不必海选**，直接用《冒烟测试示例选型》主推三例之一（首选 `ppt169_dujiangyan_zh_sparse`，14 页/0.73 MB，最稳），获取方式见该文档第 6 节。复制进 `projects/` 后在 claude 会话输入：

```text
用 ppt-master 的 Edit Native PPTX 流程处理 projects/demo.pptx：
只把第 1 页（标题页）的主标题文字替换为"部署验收测试"，
其余所有页面逐字节保留，导出为新文件，不要覆盖源文件。
完成后告诉我工作区路径和导出文件路径。
```

**验收标准**：① 导出 PPTX 能在 PowerPoint 正常打开；② 标题已改且文字可点选编辑（原生文本框而非图片）；③ 其余页母版、配色、动画原样未动；④ 工作区出现 `authoring-svg-flat/`（可编辑 SVG）、`exports/`（成品）、`sources/source.pptx`（原包只读留存）。

**这一步建立的关键认知**：Edit Native PPTX 用 `pptx_to_svg.py --roundtrip` 把源 deck 变成"可编辑 SVG + 不可变原生底座"工作区——没动的页逐字节还原，改动的页只重建改过的对象。这就是"企业级不跑版"的机制保证。

### 阶段 2 · 企业模板 + 真实材料（第 2–3 天，主力路线）

**读**：`templates-guide.md` 的"60 秒选对模板路线"与"一、选用已有模板"两节；`what-is-ppt.md` 第 5–7 节（PowerPoint 原生对象模型、什么是模板、什么是好 PPT——理解管线为什么这样设计）；`skills/ppt-master/workflows/edit-native-pptx.md` 的 §2 输入门槛与 §4 页面规划（人读是为了知道怎么喂数据）。

**门槛须知（§2 的硬规则）**：Edit Native PPTX 有事实硬规则——**页面上的每个实质论断都必须来自你给的材料**，只给标题不给材料它会反过来问你。材料支持 PDF/DOCX/图片/Markdown/网址（管线用 `source_to_md.py` 统一转换）。**20261026 首战实测（见 `../测试记录/测试记录-20261026-真实任务首战.md`）：9B 在输入缺失时不会问你，而是自拟材料自造源件一路跑到底**——所以输入核实必须由你做在会话之前，并用提示词明写"缺即停"。

**做**（真实任务模板）——**第一步永远是眼见为实**：

```bash
ls -l projects/real/sources/   # 确认 公司模板.pptx 与 需求.md 都在，再开会话
```

输入就位后按"一页一会话"拆分（5 页任务 = 5 轮，每轮 `/clear`）：

```text
用 Edit Native PPTX 流程处理 projects/real/sources/公司模板.pptx，材料在 projects/real/sources/需求.md：
① 先 ls projects/real/sources/ 并把结果贴给我；任一文件缺失立即停下问我，禁止代写材料或源件
② 本轮只做第 1 页（封面）：保留模板版式配色母版，改标题与副标题，备注写入 notes/slide_01.md
③ 改动前先给我本页的材料来源对照；禁止修改 skills/ 下任何脚本；导出到 exports/，不覆盖源文件
```

每轮替换"第 N 页"与对应要点即可。**三条铁律**（首战教训，逐条对应一次失控）：禁代写材料／备注一律写 `notes/slide_XX.md` 文件（管线不存在 SVG 内嵌备注机制，嵌入必丢）／禁改 `skills/` 下任何脚本。**不要**在提示词里写"不用跟我确认"——工作流 §4.3 的确认关卡是 9B 场景下最后一道安全网，首轮必须产出页面规划表供人工点头。

第二遍加练一个"结构手术"需求（page_plan 的能力）：`只保留源文件的第 1、3、5、7 页并按 7→1→3→5 排序，第 5 页后复制一份第 3 页改标题`——理解"源 deck 是页面素材库，输出叙事决定取舍与顺序"。若你用的是多页整卷一次交（而非一页一会话），必须盯住两点：每轮会话 `/context` 不超 ~45k；输出页数与内容结构与你的规划表逐页核对——首战中 9B 把 5 页规划跑成了模板 6 页填空、数据页无声消失，就是发生在整卷一次交的场景里。

**验收标准**：① 材料中每个数字/论断都能在产出页溯源；② 删除/重排/复制页后母版与未动页不跑版；③ 备注出现在 PowerPoint 备注栏（验证 `notes/*.md` 机制生效）；④ 单任务会话 `/context` 未超 ~45k；⑤ 收工后 `git status --short skills/` 无输出——管线源码与开工时一致。

### 阶段 3 · 企业级质量进阶（第 4–5 天）

**读**：`faq.md` 全文（源文件格式、projects 目录、AI 配图边界等真实排坑）；`animations.md` 的"默认行为"与"常用操作"；`audio-narration.md` 的"你会得到什么/两条嵌入路径/怎么触发"；`powerpoint-svg-mapping.md` 第 7、8 节（表格、图表哪些能力原生保留——汇报级数据呈现的判断依据）。

**做三件事**：

1. **动效**：给阶段 2 的 deck 加页间转场与两处元素动画：

```text
给 projects/real 工作区导出的 deck 加：全 deck 淡入式页间转场，
封面标题一个出现动画，其余页面不动。导出后告诉我在 PowerPoint 里如何核对。
```

2. **旁白**：备注齐全的 deck 可直接触发旁白（本地 qwen3.5 不产 TTS，走 edge-tts 免 Key 或已配置的 Qwen TTS，见 `audio-narration.md` §2D）：

```text
用 Edit Native PPTX 的旁白模块给 projects/real 的 deck 逐页生成中文旁白，
用 edge-tts，导出嵌入音频的 PPTX。
```

3. **对照生成**：用内置模板资产走一次 Generate 快速模式，建立"Edit Native 保设计 vs Generate 新设计"的手感：

```text
用内置 styles 里的 operating-review 风格，快速生成一份 5 页的
《季度运营复盘》PPT，不用跟我确认。
```

**验收标准**：① 动画兼容性校验通过、PowerPoint 播放正常；② 旁白时序与页面匹配；③ 数据图表与材料数字一致（Agent 会在导出前跑 verify-charts 质量门，理解它的作用）；④ 能说出三种"企业模板来源"的取舍：企业现成 PPTX（Edit Native）/ 内置品牌风格（Generate）/ 自建资产（阶段 4）。

### 阶段 4 · 模板资产化（按需）

**读**：`templates-guide.md` 的"二、派生新模板（重点）"与"三、模板的边界"；`templates-architecture.md` 第一、二节；`routing.md` §5 四分类判定。

**做**：把企业 .pptx 提炼为可复用模板工作区（四选一按判据：只沉淀 VI 身份 → brand；沉淀"沟通方法+设计方向" → style；品牌无关的页面结构 → layout；整套可复用成品 → deck）：

```text
用 Create Template 流程，把 projects/assets/公司模板.pptx 提炼为 deck 模板，
工作区放到 templates-workspaces/ 下，按 create-deck 的校验走完并导出预览。
```

**验收标准**：① 产出的模板工作区通过 Create Template 校验；② 新会话中 Generate 路线能显式挂载该模板根并按其 design_spec 出页；③ 形成一张企业内部"模板选型速查卡"（内置 21 品牌/14 风格 + 自建资产）。

> 💡 模板提炼是四类任务里对模型要求最高的一次性任务（理解设计意图、抽象 design_spec），本地 9B 建议只跑简单的 brand/layout；**style/deck 提炼建议临时把 `ANTHROPIC_BASE_URL` 切到云端强模型一次性完成**（部署方案 §8.4 的既定边界），产出入库后日常消费回到本地。

---

## 4. 与本地 qwen3.5:9b 的配合边界（全程适用）

以下边界来自部署方案 §8.3/§8.4 与四层验收实测，学习各阶段都要遵守：

1. **路线优先级**：优先 Edit Native PPTX（对 9B 最友好，管线只要求"改对局部"，不要求"凭空设计"）；Generate 默认管线的多轮 spec/lock/confirm 对 9B 偏重，日常用 quick-generate 快速模式。
2. **单次任务 5–8 页**：长材料先自己分段提炼成 Markdown 再喂，不要把 50 页 DOCX 原样丢进去。
3. **配图**：本地 9B 不产图。企业模板场景本来就不该动图（保设计）；Generate 场景配图用网络图源或自带素材。
4. **思考开销是常态**：Thought for 10–60s 正常；别在等待中重复提问打断它。真跑偏了 `Esc` 打断补一句纠正。
5. **上下文纪律**：每个任务一个会话，任务完成 `/clear`；`/context` 超过 ~45k 就收尾（autocompact buffer 占半窗，实际余量 ~32.5k，见部署方案 §8.2 第四坑）；关键约定（公司模板路径、命名规范）写进项目 `CLAUDE.md`，跨压缩存活。

## 5. 提示词速查卡（复制即用）

| 场景 | 提示词要点 | 出处 |
|---|---|---|
| 冒烟 | Edit Native PPTX + "只改第 1 页标题 + 其余逐字节保留 + 不覆盖源文件" | 阶段 1 |
| 主力任务 | 指定基底 PPTX + 材料 md + 页数与每页要点 + 加备注 + 导出不覆盖 | 阶段 2 |
| 结构手术 | "只保留第 1/3/5/7 页，按 7→1→3→5 排序，第 5 页后复制第 3 页改标题" | 阶段 2 |
| 加动效 | "全 deck 淡入转场 + 封面标题出现动画 + 其余不动" | 阶段 3 |
| 加旁白 | "逐页中文旁白，用 edge-tts，导出嵌入音频的 PPTX" | 阶段 3 |
| 快速对照 | "用内置 styles 的 operating-review 快速生成 5 页，不用跟我确认" | 阶段 3 |
| 模板提炼 | "用 Create Template 流程把 XX.pptx 提炼为 deck 模板" | 阶段 4 |
| 沉淀记录 | "把本次任务过程与结果写入 docs/测试记录-<日期>.md，conventional commit，只 commit 不 push" | 协作规则 |

通用句式规律：**点名路线（Edit Native PPTX / Generate / Create Template）→ 指定输入（源 PPTX + 材料）→ 说清约束（保什么、改什么、页数、导出到哪、覆盖与否）→ 声明确认偏好（要不要逐步确认）**。点名路线对 9B 尤其重要，等于替它做了 routing。

## 6. 遇到问题去哪查（三级排障）与阶段自查清单

**排障顺序**：① `docs/zh/faq.md`（材料格式、projects/exports、AI 配图等日常问题）；② `skills/ppt-master/scripts/docs/troubleshooting.md`（管线/脚本级问题）；③ `docs/部署方案/...md` 第 10 章（本地环境：四参数、200k 假设、多份安装、count_tokens 降速等）。

**阶段自查清单**：

- [ ] 阶段 1：冒烟导出件可打开、标题可编辑、其余页未动、roundtrip 工作区四件套齐全
- [ ] 阶段 2：事实可溯源、删排页不跑版、备注入栏、会话上下文未超限
- [ ] 阶段 3：动画校验通过、旁白时序正确、图表数字一致、三种模板来源取舍能口述
- [ ] 阶段 4：模板工作区过校验、Generate 可挂载、选型速查卡成型
- [ ] 全程：每个任务一个会话、任务做完 `/clear`、测试记录按协作规则入 `docs/`

路径走完的标志：拿到一份新的真实 PPT 需求时，你能在 10 分钟内判断该走哪条路线、需要准备什么输入、并直接用速查卡开跑——这时"尽快入手"的目标就达成了。
