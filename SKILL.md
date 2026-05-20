---
name: weekly-report
description: 根据 git commit 记录生成标准格式周报。当用户说"生成周报"、"写周报"、"整理本周工作"、"帮我写周报"时触发。扫描 WEEKLY_GIT_SCAN_REPOS 环境变量指定目录下所有 git 仓库，收集当前 git 用户本周提交，智能归纳成四段式周报格式。
---

## 目标

扫描指定根目录下所有 git 仓库，收集**当前 git 用户**本周一至今的提交记录，归纳整理成标准四段式周报。

## 环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `WEEKLY_GIT_SCAN_REPOS` | 要扫描的代码库根目录，多个目录用 `,` 分隔，**必填** | `/Users/me/java,/Users/me/go` |
| `GIT_AUTHOR` | 指定作者（可选，默认取 `git config --global user.name`） | `zhangsan` |

如果 `WEEKLY_GIT_SCAN_REPOS` 未设置，**停止执行**并提示：
```
请先设置环境变量：
export WEEKLY_GIT_SCAN_REPOS=/your/code/root                    # 单个目录
export WEEKLY_GIT_SCAN_REPOS=/path/one,/path/two                # 多个目录用逗号分隔
然后重新运行。
```

## 执行步骤

### Step 1：准备参数

用 Bash 工具运行以下命令，收集必要信息：

```bash
# 1. 确认扫描根目录（按逗号拆分为多个路径）
IFS=',' read -ra SCAN_DIRS <<< "$WEEKLY_GIT_SCAN_REPOS"
echo "扫描目录：${SCAN_DIRS[@]}"

# 2. 确定作者（优先用 GIT_AUTHOR，否则读全局 git 配置）
AUTHOR="${GIT_AUTHOR:-$(git config --global user.name)}"
echo "AUTHOR=$AUTHOR"

# 3. 计算本周一日期
MONDAY=$(python3 -c "
from datetime import date, timedelta
today = date.today()
monday = today - timedelta(days=today.weekday())
print(monday.strftime('%Y-%m-%d'))
")
echo "MONDAY=$MONDAY"
```

### Step 2：扫描所有 git 仓库

```bash
# 按逗号拆分 WEEKLY_GIT_SCAN_REPOS，对每个目录递归查找 git 仓库（最多 4 层深度）
IFS=',' read -ra SCAN_DIRS <<< "$WEEKLY_GIT_SCAN_REPOS"
for dir in "${SCAN_DIRS[@]}"; do
  find "$dir" -maxdepth 4 -name ".git" -type d 2>/dev/null | sed 's|/.git$||'
done | sort
```

### Step 3：收集每个仓库的提交

对 Step 2 找到的每个仓库路径执行，获取提交 hash 和 message：

```bash
git -C "$REPO_PATH" log \
  --author="$AUTHOR" \
  --since="$MONDAY 00:00:00" \
  --pretty=format:"%H | %ad | %s" \
  --date=format:"%m-%d(%a)" \
  --no-merges 2>/dev/null
```

记录仓库名（取路径最后一段）和对应的提交列表。跳过没有任何提交的仓库。

### Step 3.5：读取模糊提交的 diff

对于 commit message 过于简短或含义不明确的提交（如仅有 `fix`、`update`、`feat: xxx新增`等笼统描述），
读取该提交的代码 diff 以理解具体改动：

```bash
git -C "$REPO_PATH" diff --stat "$HASH^" "$HASH"
git -C "$REPO_PATH" diff "$HASH^" "$HASH" -- '*.java' '*.go' '*.ts' '*.py' \
  | grep '^[+-]' | grep -v '^[+-][+-][+-]' | grep -v '^[+-]\s*//' | head -60
```

通过 diff 理解实际的业务行为变化，而不是照搬 commit message 的字面描述。

### Step 4：扫描未提交改动

对每个仓库执行，获取工作区中尚未提交的变更文件列表：

```bash
git -C "$REPO_PATH" status --short 2>/dev/null
```

若有未提交文件，进一步读取 diff 理解改动内容：

```bash
git -C "$REPO_PATH" diff HEAD -- '*.java' '*.go' '*.ts' '*.py' \
  | grep '^[+-]' | grep -v '^[+-][+-][+-]' | grep -v '^[+-]\s*//' | head -60
```

将有意义的未提交改动归纳后加入周报，标注 **（开发中）**。
忽略配置文件、日志、临时文件等无意义改动。

### Step 5：生成周报

将已提交工作和未提交改动合并，归纳后**严格按以下格式**输出：

---

## 输出格式

```
一、本周重点工作：
1. <主题，概括一组相关工作>
   - <具体子项>
   - <具体子项>
2. <主题>（开发中）          ← 未提交改动用此标注
   - <具体子项>
...


```

---

## 安全限制

**严禁执行任何写操作**，只允许只读的 git 命令：

| 禁止 | 允许 |
|------|------|
| `git commit` | `git log` |
| `git add` | `git diff` |
| `git stash` | `git status` |
| `git rm` | `git show` |
| `git push` | `git config --get` |
| `git reset` | `find` |

如果执行过程中需要暂存工作区以查看某些内容，**不得使用 `git stash`**，直接对当前工作区执行 `git diff HEAD` 即可。

## 归纳原则

- **合并同类**：同一仓库多条提交合并归纳为 1-2 条，突出核心工作
- **业务语言**：将技术 commit 转为业务描述，例如 `fix: NPE in UserService` → `修复用户服务空指针问题`
- **忽略噪音**：跳过纯辅助性提交（`chore`、`style`、`bump version`、`merge` 等），除非是本周唯一工作
- **中文输出**：所有描述用中文，不保留项目名

## 无提交时的处理

如果所有仓库都没有找到本周提交，输出：

```
未在 $WEEKLY_GIT_SCAN_REPOS 下找到「$AUTHOR」本周（$MONDAY 至今）的任何提交。

请确认：
1. WEEKLY_GIT_SCAN_REPOS 路径是否正确（当前：$WEEKLY_GIT_SCAN_REPOS）
2. 作者名是否匹配 git log 中的实际 author（当前：$AUTHOR）
   可通过 git log --pretty="%an" 查看仓库中的实际作者名
   如需指定，设置：export GIT_AUTHOR=<实际作者名>
```
