---
name: creating-git-work-branches
description: 当用户希望从远程最新 main 创建任务分支、切换到 feature、bugfix、refactor 或 docs 分支，审阅后使用 Conventional Commits 提交改动，或将本地任务分支推送到远端同名分支并建立 upstream 时使用。
---

# 创建 Git 工作分支

## 概述

从 `origin/main` 创建命名统一且不跟踪远端 `main` 的任务分支，准备符合 Conventional Commits 的提交，并安全推送到远端同名分支。将创建分支、暂存文件、执行提交和首次推送视为独立的确认门禁；不得向用户隐藏任何会改变 Git 状态的操作。

## 操作边界

- 仅在用户明确指定的 Git 仓库内操作。
- 只允许按本 skill 的“推送任务分支”流程执行普通 push；不创建 PR，不执行 merge、rebase、stash、amend、force push、删除分支或部署。
- 不丢弃、覆盖或静默携带已有改动。
- 禁止直接在 `main` 上提交。
- 每条会改变 Git 状态的命令执行前都必须取得确认。用户最初要求使用本 skill，不代表其已跳过后续审阅门禁。

## 持续任务与确认复用

先判断当前请求是新任务，还是同一业务目标下的持续修改。用户已经明确确认“继续当前分支”后，只要业务目标、仓库集合和当前分支均未变化，该选择在持续任务中保持有效；后续追加修改只报告当前分支和同步状态，不再重复询问是否创建任务分支。

出现以下任一情况时，停止复用并重新执行分支确认：

- 用户切换到新的业务目标；
- 当前分支发生变化；
- 涉及的仓库新增或减少；
- 当前分支相对同名远端变为 `behind` 或 `diverged`；
- 用户明确要求重新创建或选择分支。

分支状态、实施方案和“创建新分支或继续当前分支”的选择可以在同一条消息中展示并一次确认。确认继续当前分支不授权暂存、提交或推送；这些改变 Git 状态的操作仍按各自门禁单独确认。

## 创建分支

1. 使用以下命令定位并检查仓库：

   ```bash
   git rev-parse --show-toplevel
   git status --short --branch
   git branch --show-current
   git remote get-url origin
   ```

2. 如果工作区存在已暂存、未暂存或未跟踪文件，立即停止。列出这些文件，并说明本 skill 不会自动 stash，也不会把它们带入新分支。等待用户自行处理工作区。
3. 如果 HEAD 处于 detached 状态、缺少 `origin`，或无法解析 `main`，立即停止并报告准确原因。不得猜测其他基线分支。
4. 使用 `git fetch origin main` 获取最新基线。如果运行环境要求授权，则将网络访问或凭证操作作为明确的权限边界。
5. 按以下格式生成分支名：

   ```text
   <type>/<project>-<MMDD>-<english-description>
   ```

   - `type`：根据任务选择 `feature`、`bugfix`、`refactor` 或 `docs`；无法确定时询问用户。
   - `project`：根据仓库或当前项目目录推断，使用 `oseflow` 这类简短的小写标识；无法可靠推断时询问用户。
   - `MMDD`：使用用户当前时区的执行日期并补齐四位，例如 8 月 11 日为 `0811`。
   - `english-description`：使用简短的小写英文 kebab-case 描述任务，不包含无意义词语或敏感信息。

6. 检查同名本地或远程分支是否已经存在。如果存在，立即停止并请用户更换描述；不得自行添加后缀。
7. 创建分支前展示：
   - 仓库根目录和 remote URL；
   - 当前分支和 HEAD SHA；
   - `origin/main` 及其 SHA；
   - 完整的建议分支名；
   - 准确命令：`git switch --no-track -c <branch> origin/main`。
8. 等待用户明确确认。确认后只执行已展示的命令，再使用 `git status --short --branch`、`git branch --show-current` 和 `git rev-parse --abbrev-ref --symbolic-full-name @{upstream}` 验证结果。最后一条命令此时必须因尚未设置 upstream 而失败；如果显示 `origin/main`，立即停止，不得继续提交或推送。

示例：

```text
feature/oseflow-0811-add-api-envelope
bugfix/oseflow-0811-fix-soft-delete-status
refactor/oseflow-0811-simplify-auth-flow
docs/oseflow-0811-update-architecture-readme
```

## 准备并提交改动

1. 使用以下命令进行只读检查：

   ```bash
   git status --short --branch
   git branch --show-current
   git diff --stat
   git diff
   git diff --cached --stat
   git diff --cached
   ```

2. 如果当前分支是 `main`、HEAD 处于 detached 状态或没有任何改动，立即停止。
3. 区分已暂存、未暂存和未跟踪文件。建议提交前必须阅读相关 diff。标记生成物、环境文件、凭证、无关修改和异常的大文件；默认不得暂存它们。
4. 每个 Commit 只包含一个逻辑改动。如果存在无关修改，提出不同的文件分组，并且一次只审阅一组。
5. 按以下格式建议 Commit Message：

   ```text
   <type>(<scope>): <subject>
   ```

   使用 `feat`、`fix`、`refactor`、`docs`、`test`、`build` 或 `chore` 等 Conventional Commit 类型。根据受影响项目或模块推断 `scope`。`subject` 使用用户的工作语言，内容必须具体，末尾不加句号。
6. 暂存前展示：
   - 建议提交的准确文件列表；
   - 排除或存在疑问的文件；
   - 简短 diff 摘要；
   - 完整 Commit Message；
   - 准确的 `git add -- <paths>` 和 `git commit -m <message>` 命令。
7. 等待用户明确确认文件列表和 Commit Message。
8. 只暂存用户确认的准确路径。禁止使用 `git add .`、`git add -A` 或宽泛 glob。
9. 重新运行 `git diff --cached --stat` 和 `git diff --cached`。如果暂存内容与已确认的文件集合不一致，立即停止并重新请求审阅。
10. 如果暂存内容与已确认文件集合一致，展示简短的最终暂存摘要、校验结果和 Commit Message，不重复完整文件列表；再次等待用户明确确认后才能执行 `git commit`。如果集合发生变化，必须重新展示准确文件列表并请求审阅。
11. 执行用户确认过的提交命令，再使用以下命令验证：

    ```bash
    git status --short --branch
    git show --stat --oneline --decorate --no-renames HEAD
    ```

12. 报告新 Commit SHA、subject、包含的文件和剩余未提交改动。提交完成不等于已经批准推送；继续推送前必须进入下一节的独立确认门禁。

## 推送任务分支

1. 使用以下命令进行只读检查：

   ```bash
   git status --short --branch
   git branch --show-current
   git remote get-url origin
   git rev-parse --abbrev-ref --symbolic-full-name @{upstream}
   ```

2. 如果当前分支为空、是 `main`，或不符合 `<type>/<project>-<MMDD>-<english-description>` 格式，立即停止。
3. 如果当前 upstream 是 `origin/main`，立即停止并警告目标错误。不得运行普通 `git push`，也不得推送到 `main`。
4. 如果尚未设置 upstream，首次推送命令必须是：

   ```bash
   git push -u origin <current-branch>
   ```

   `<current-branch>` 必须替换为 `git branch --show-current` 返回的完整分支名。禁止省略分支名，禁止使用 `HEAD:main` 或其他 refspec。
5. 如果 upstream 已经是 `origin/<current-branch>`，后续推送可以使用 `git push`；仍须在执行前展示目标 remote、当前分支和准确命令。
6. 推送前等待用户明确确认。网络访问或凭证操作按运行环境要求申请授权。
7. 推送后使用以下命令验证 upstream 和远端分支：

   ```bash
   git rev-parse --abbrev-ref --symbolic-full-name @{upstream}
   git status --short --branch
   ```

   upstream 必须严格等于 `origin/<current-branch>`。否则报告异常并停止，不自动修改 Git 配置。

## 确认用语

当一条消息中只有一个明确的待确认门禁时，用户直接回复“确认”即可视为对该门禁的有效授权；存在多个门禁或范围可能混淆时，要求用户给出“确认创建分支”“确认暂存这些文件”“确认提交”或“确认推送”等明确回复。疑问、局部修改意见、沉默或泛泛的“继续”都不代表用户批准了暂存、提交或推送。

## 失败处理

- 命令失败时，报告命令和相关 error，然后停止，不得改用另一条会改变状态的命令继续尝试。
- 用户解决问题后，重新执行只读检查；不得假设仓库状态没有变化。
- 仓库规则与本 skill 冲突时，遵循优先级更高的仓库规则或用户指令，并明确说明冲突。
