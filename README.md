# weekly-report

一个 Claude Code skill，自动扫描 git 仓库提交记录，生成标准四段式周报。

## 功能

- 扫描指定目录下所有 git 仓库（最多 4 层深度）
- 收集当前 git 用户本周一至今的提交
- 对模糊 commit 自动读取 diff 理解实际改动
- 识别未提交改动并标注「开发中」
- 将技术提交归纳为业务语言，输出中文周报

## 触发方式

在 Claude Code 中说：「生成周报」、「写周报」、「整理本周工作」

## 环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `WEEKLY_GIT_SCAN_REPOS` | 要扫描的代码库根目录，**必填**，多个目录用 `,` 分隔 | `/Users/me/java,/Users/me/go` |
| `GIT_AUTHOR` | 指定 git 作者（可选，默认取 `git config --global user.name`） | `zhangsan` |

```bash
export WEEKLY_GIT_SCAN_REPOS=/Users/me/projects           # 单个目录
export WEEKLY_GIT_SCAN_REPOS=/Users/me/java,/Users/me/go  # 多个目录
export GIT_AUTHOR=zhangsan                                 # 可选，指定作者
```

## 输出格式

```
一、本周重点工作：
1. <主题>
   - <具体子项>
2. <主题>（开发中）
   - <具体子项>


```

## 安装

将 `SKILL.md` 放入 Claude Code 的 skills 目录（`~/.claude/skills/`）即可。
