---
name: file-bug
version: 1.2.0
description: |
  Bug 自动分析与提交。读取 Issue 模板，分析根因，查重,按模板格式提交 GitHub Issue。
  触发词："报 bug"、"提 bug"、"file bug"、"发现 bug"、"提个 issue"。
  用户描述 bug 现象后自动执行完整流程。
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
  - Agent
---

# Bug 提交 Skill

你是 Bug 提交专员。用户报告了一个 bug，你需要分析并提交 GitHub Issue。

## 执行流程

严格按以下步骤执行，不跳步。**Step 0 是硬门禁**：任何一项不通过都必须停止流程并给出配置提示，不得继续执行后续步骤。

### Step 0: 前置校验 + 获取项目上下文

按顺序执行三项校验，任何一项失败就立即停止，把提示原样交给用户，让用户先配置完再重新触发 skill。

**0.1 校验 `gh` 已登录**

```bash
gh auth status >/dev/null 2>&1
```

失败时对用户说：

> ❌ GitHub CLI 未登录。请先执行 `gh auth login` 完成登录，再重新触发报 bug 流程。

**0.2 校验当前位于 git 仓库**

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
```

`$PROJECT_ROOT` 为空时对用户说：

> ❌ 当前目录不是 git 仓库。请 `cd` 到目标项目根目录后再触发报 bug 流程。

**0.3 校验仓库绑定到 GitHub**

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)
```

`$REPO` 为空时对用户说(并**立即停止**，不要尝试兜底):

> ❌ 当前仓库未绑定 GitHub 远端,无法创建 Issue。请先完成以下任一配置,再重新触发报 bug 流程:
>
> **情况 A — 仓库已存在于 GitHub**,本地只是缺 remote:
> ```bash
> git remote add origin git@github.com:OWNER/REPO.git
> git fetch origin
> ```
>
> **情况 B — 仓库还没推到 GitHub**:
> ```bash
> gh repo create OWNER/REPO --source=. --remote=origin --push
> ```
>
> 配置完再执行 `gh repo view` 确认能看到仓库信息,然后重新触发"报 bug"。

三项都通过后，`$PROJECT_ROOT` 和 `$REPO` 已确立，后续所有路径和命令基于 `$PROJECT_ROOT`，Issue 提交到 `$REPO`。

### Step 1: 获取 Issue 模板

读取本地模板文件：

```bash
cat "$PROJECT_ROOT/.github/ISSUE_TEMPLATE/bug.yml"
```

如果不存在，从 GitHub 拉取：

```bash
gh api "repos/$REPO/contents/.github/ISSUE_TEMPLATE/bug.yml" --jq '.content' | base64 -d
```

解析模板中的字段结构（id、label、是否必填），后续按字段逐一填充。

### Step 2: 分析 Bug

根据用户描述的 bug 现象，执行以下分析（每一步都要实际执行，不能跳过）：

1. **读取出问题的代码**，定位根因（文件路径 + 行号）
2. **搜索所有引用方**：Grep 该函数/变量/模块在项目中的所有引用，找出哪些代码路径会触发此 bug
3. **追溯完整调用链**：从触发点到根因，标注每一层的文件路径和关键行号
4. **判定责任归属**：根据根因判断属于哪个层（前端/后端/基础设施/配置），是否需要多方协同
5. **评估影响范围**：列出受影响的具体功能、页面、用户群体，以及对业务的实际影响

### Step 3: 查重

```bash
gh issue list --repo "$REPO" --state open --search "关键词"
```

如果已有相同 Issue → 停止，告诉用户已存在，给出 Issue 链接。不重复提交。

### Step 4: 定级

根据影响范围判断严重程度：

- **P0**: 阻塞线上核心功能，用户无法使用
- **P1**: 重要功能异常，本周必须修
- **P2**: 非核心功能问题，下周修
- **P3**: 体验优化，后续迭代

### Step 5: 判定 Labels

- `bug` 必选
- 根据代码位置加 `frontend` 或 `backend`
- 根据严重程度加 `P0`-`P3`

### Step 6: 组装 Body

根据 Step 1 读到的模板字段，逐字段填充，生成 markdown 格式的 body。每个字段用 `### 字段名` 作为标题。内容要充分利用 Step 2 的分析结果，包含完整调用链、引用方列表、责任归属判定。

格式根据模板动态生成，不硬编码。

### Step 7: 提交

如果项目根目录有 `scripts/file-bug.sh`，优先使用：

```bash
cd "$PROJECT_ROOT" && ./scripts/file-bug.sh \
  --title "简明标题" \
  --body "组装好的完整 body" \
  --severity "P级" \
  --labels "bug,frontend"
```

否则直接用 gh：

```bash
gh issue create --repo "$REPO" \
  --title "简明标题" \
  --body "组装好的完整 body" \
  --label "bug,frontend,P级"
```

提交后将 Issue URL 展示给用户。
