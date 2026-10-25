# GitHub 协作规则

> 文档版本：v1.0 ｜ 生效日期：2026-10-26
> 适用范围：本项目全体成员及其驱动的 AI Agent（Claude Code、Codex CLI 等）
> 关联文档：`PPT-Master评估与企业模板复用方案.md`
>
> **仓库地址**：`https://github.com/dongf9527/ppt-master.git`
> 协作仓库为新建空仓库；上游 `hugohe3/ppt-master` 保持只读参考，不直接向上游推送。

---

## 1. 规则总表

| # | 规则 | 说明 |
|---|---|---|
| 1 | **唯一同步方式** | 所有代码/文档改动通过 git commit + git push 推送到 GitHub，不在对话中凭空产生代码 |
| 2 | **仓库位置** | 严格使用 `https://github.com/dongf9527/ppt-master.git`，不在其他目录创建 `.git` |
| 3 | **提交规范** | 小步提交，commit message 使用 conventional commits 风格（如 `feat:`、`fix:`、`docs:`） |
| 4 | **会话记录** | 分析/评估/测试记录写入 `docs/` 目录，以 Markdown 形式提交推送 |
| 5 | **禁止 force push** | 绝不使用 `git push --force`；同理禁止对已推送 commit 执行 `--amend` 后再推送（等效 force，同样触犯本条）；未推送 commit 的 amend 属修正笔误常规手段，不受限 |
| 6 | **Rebase 前置** | rebase 前先评估替代方案（revert/merge），必要时提前告知确认 |
| 7 | **bug 修复不重命名变量** | 修复 bug 时保持原变量名，仅在 refactor 时重命名 |

---

## 2. 条款细则

### 规则 1 —— 唯一同步方式

**动机**：对话内容易失联、不可审计；git 历史是唯一的可信事实源。

**正确做法**：

- 任何有价值的改动（代码、模板工作区、文档、测试记录）在产生后立即 `git add` → `git commit` → `git push`；
- 对话中讨论出的结论、AI 生成的代码片段，必须先落到工作区文件里，再走 git 提交，不允许"只存在于聊天记录中"；
- AI Agent 执行的每一次变更，由操作者负责确认 diff 后提交。

**违规示例**：❌ 在聊天里贴一段 SVG 代码说"用这个"但从未写入文件提交；❌ 把产出只存在本地 `projects/` 而不推送。

### 规则 2 —— 仓库位置

**动机**：防止在错误目录初始化仓库，导致把 `ppt-master` 上游源码、临时分析目录或个人文件误推到协作仓库。

**正确做法**：

- `.git` 只允许存在于 `https://github.com/dongf9527/ppt-master.git` 对应的本地克隆目录中；
- 协作仓库与本仓库的关系建议：上游 `hugohe3/ppt-master` 保持只读参考（或以 fork/子模块方式挂接），协作仓库承载**我们的模板工作区（`templates/`）、项目产出（`projects/`）、文档（`docs/`）**；
- 克隆前先确认当前目录：`git remote -v` 输出必须指向 `https://github.com/dongf9527/ppt-master.git`。

**违规示例**：❌ 在 `repo-analysis/`、`/tmp` 或下载解压目录里 `git init`。

### 规则 3 —— 提交规范

**动机**：小步提交便于回滚与评审；conventional commits 便于自动生成 changelog。

**commit message 格式**：`<type>(<scope>): <简短描述>`

| type | 用途 | 示例 |
|---|---|---|
| `feat` | 新增功能/模板/能力 | `feat(deck): 新增中国电信风格 deck 模板工作区` |
| `fix` | 修复问题 | `fix(template): 修正 design_spec 主色 HEX 值` |
| `docs` | 文档变更 | `docs: 补充 Ollama 接入配置说明` |
| `refactor` | 重构（不改行为） | `refactor(layout): 抽取公共槽位约定` |
| `test` | 测试记录/用例 | `test: 记录 qwen3:8b 回填路径验证结果` |
| `chore` | 杂项维护 | `chore: 更新 .gitignore 排除 exports/` |

**细则**：一次提交只做一件事；模板工作区整体入库时可在同一 commit 内包含其全部文件（spec + SVG + 素材）。

### 规则 4 —— 会话记录

**动机**：AI 协作产生的分析、评估、测试结论是团队资产，必须可追溯。

**正确做法**：

- 目录约定：`docs/` 下按主题存放 Markdown 文件，例如：
  - `docs/ppt-master评估与企业模板复用方案.md`
  - `docs/GitHub协作规则.md`（本文档）
  - `docs/测试记录-<日期>-<主题>.md`
- 每次实质性分析/评估/测试后，将结论整理成 Markdown 并提交推送；
- 测试记录建议包含：日期、环境（模型/宿主/版本）、操作步骤、结果、结论与下一步。

**违规示例**：❌ 测试结论只写在聊天里；❌ 把记录散落在个人目录。

### 规则 5 —— 禁止 force push

**动机**：force push 会不可逆地丢弃远端历史，破坏他人本地状态。

**细则**：

- 绝不执行 `git push --force` / `git push -f`（含 `--force-with-lease` 的滥用场景——正常协作中同样避免）；
- **等效行为同样禁止**：对**已推送**的 commit 执行 `git commit --amend` 再推送、`git rebase` 已推送分支后推送、`git reset --hard <旧提交>` 后推送；
- **例外（不受限）**：commit **尚未 push** 时，`--amend` 修正笔误是常规手段，允许使用；
- 误操作需要改远端历史时，停下来在会话中与团队确认，走例外审批，而不是自行 force。

**自查命令**：

```bash
git log origin/<branch>..<branch>   # 列出未推送的提交——对以下列出的提交 amend 是安全的
```

### 规则 6 —— Rebase 前置

**动机**：rebase 改写历史，是协同分支上冲突与丢失提交的高发源。

**决策顺序**：需要整合分支时，按以下顺序评估：

1. `git revert` —— 已推送到共享分支的错误提交，首选 revert（新增反向提交，不改历史）；
2. `git merge` —— 常规整合，保留真实历史；
3. `git rebase` —— 仅在以上方案不适用（如需要线性历史的功能分支尚未被他人拉取）时使用，且**使用前在会话中告知相关成员并确认**分支确实无他人基于其开发。

### 规则 7 —— bug 修复不重命名变量

**动机**：让 fix 提交的 diff 只包含"修复"本身，评审者一眼看清改动意图；混入重命名会让 `git blame` 与 `git log -S` 追溯失效，也抬高合并冲突概率。

**正确做法**：

- bug 修复提交中：只改动与缺陷直接相关的行，变量名、函数名、文件名一律保持原样；
- 确实需要重命名时：先提交 `fix`，再单独提交 `refactor`（两步、两个 commit），使每个 commit 的意图单一；
- AI Agent 生成修复代码时同样受此约束——操作者在提交前检查 diff 是否夹带重命名。

**示例**：

```text
❌ fix(login): 修正空指针并重命名 user → userInfo        # 混合意图
✅ fix(login): 修正登录空指针判断                          # 第一步
✅ refactor(login): user 重命名为 userInfo                # 第二步（可选，另行提交）
```

---

## 3. 标准工作流参考

```bash
# 0. 克隆（仅首次；确认 remote 指向协作仓库）
git clone https://github.com/dongf9527/ppt-master.git && cd ppt-master

# 1. 同步最新
git pull --rebase=false          # 或按规则 6 评估后的整合方式

# 2. 在分支上工作（推荐）
git checkout -b feat/corp-deck-template

# 3. 小步提交
git add templates/decks/<模板id>/
git commit -m "feat(deck): 新增企业 deck 模板工作区"

# 4. 推送（绝不使用 -f）
git push origin feat/corp-deck-template
```

## 4. 提交前自查清单

- [ ] 改动是否已全部落到文件（而非仅存在于对话中）？
- [ ] `git remote -v` 是否指向 `https://github.com/dongf9527/ppt-master.git`？
- [ ] commit message 是否符合 conventional commits？
- [ ] 是否为小步提交、单一意图？
- [ ] 分析/评估/测试记录是否已写入 `docs/` 并包含在本次提交中？
- [ ] 本次推送**不含** force、不含对已推送 commit 的 amend/rebase 改写？
- [ ] 若为 bug 修复：diff 是否保持了原变量名？
- [ ] 若执行过 rebase：是否已按规则 6 完成事前评估与确认？

## 5. 附则

- 本规则对人类成员与 AI Agent 一体适用；由 AI 执行的操作，其发起人对合规性负责。
- 规则修订通过本文档的 `docs:` 类型提交完成，修订本身同样遵守本规则。
- 仓库地址已于 2026-10-26 确定为 `https://github.com/dongf9527/ppt-master.git`，并在团队内公告。
