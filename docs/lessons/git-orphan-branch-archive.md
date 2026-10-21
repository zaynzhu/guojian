# 在同一仓库归档外部源码为对照用孤儿分支（git）

## 结论速览

- **最终方案**：`git mktree </dev/null` 造空树 → `git commit-tree` 造占位提交 → `git branch <名> <commit>` 建"空"孤儿分支；源码到手后 `git switch <名>` → 复制 → `git add -A` → `git commit` → `git switch main`。
- **适用条件**：Windows Git Bash；需要长期保留一份与主线历史完全无关的外部源码（作者原版/上游基准）供 `git diff` 对照；工作区有 ignored 缓存时先 `git status --ignored --short` 排查。

## ✅ 不动工作区建孤儿分支并归档作者原版源码（2026-10-21）

- **为何值得记**：`git switch --orphan` 会把主线 index 全部带入新分支，要先清 index 才能得到"空"分支，切回主线还可能撞 untracked 覆盖拒绝；当时比选三条路线后选定 plumbing 零接触方案。本项目"原始"分支即按此法创建，作者发布新版需重新归档时直接复用。
- **最终方案**（原文）：
  ```bash
  # 1) 不碰 index/工作区，直接造"空占位"孤儿分支（git 无法建无提交的分支）
  empty_tree=$(git mktree </dev/null)
  commit=$(git commit-tree "$empty_tree" -m "chore: 建立原始分支占位，等待作者原版源码")
  git branch 原始 "$commit"
  # 2) 源码 zip 先解压到项目外临时目录并预检（无 .git/密钥/apk/构建产物）
  # 3) 确认工作区无 ignored 残留
  git status --ignored --short   # 无输出才安全
  # 4) 放入源码并提交
  git switch 原始
  # PowerShell: Copy-Item -Path "$env:TEMP\<解压目录>\guoapp\*" -Destination "<仓库根>\" -Recurse -Force
  git add -A
  git commit -m "chore: 归档作者原版源码"
  git switch main                 # 工作区自动恢复主线内容
  ```
- **为什么这样做**：孤儿分支使归档历史与主线彻底独立；`mktree`/`commit-tree`/`branch` 三个 plumbing 命令只创建对象和引用，不触碰 index 与工作区，主线零风险；先占位后填源码，两步各自原子。归档内容放仓库根并去掉 zip 顶层文件夹，与 main 目录结构对齐，`git diff main 原始 -- <path>` 逐文件对照时路径才一致。
- **适用条件**：Windows 11 / Git Bash（`git mktree </dev/null` 依赖 Unix 重定向；PowerShell 下的等价写法如 `'' | git mktree` 未验证）。中文分支名（"原始"）在 Windows Git Bash 下创建/切换/日志全程正常。
- **验证证据**（2026-10-21 实跑，guojian 仓库）：
  - `git log --oneline 原始` → `e657e79 chore: 归档作者原版源码` / `9f8fb63 chore: 建立原始分支占位，等待作者原版源码`
  - `git show --stat HEAD` → `373 files changed, 72460 insertions(+)`（与 zip 条目数一致，无多余文件混入）
  - 切回 main 后 `git status --short` 干净，`git branch -v` 两分支各自指向正确提交
- **交叉验证**：单 agent 单次跑通，无多轮/多 agent 审查——如实标注，非"精粹"。
- **关键步骤**：zip 预检 → 项目外临时目录解压 → 查 ignored 残留 → plumbing 建占位分支 → switch → 去掉 zip 顶层文件夹复制到仓库根 → add/commit → switch 回 main → 清理临时目录。
- **易错点**：
  - `git switch --orphan` 不清 index，直接用会把主线文件全部带入归档提交；
  - 工作区有 `.dart_tool/`、`build/` 等 ignored 缓存时，切到归档分支后 `git add -A` 可能按归档源码自带的 .gitignore 规则把残留误提交——先 `git status --ignored --short` 确认无输出；
  - Windows 下 `git add` 满屏 `warning: LF will be replaced by CRLF` 只是 checkout 预告，实测入库 blob 保留作者原字节（CRLF 文件 CRLF 入库），不是错误；
  - **验证归档保真不要用 `git archive` 导出对比**：archive 会按仓库 .gitattributes 的 `text=auto`/eol 规则转换导出内容，曾把 373 个文件中的 78 个制造出假哈希差异（去 CR 后字节全同）；权威方法是用 `git ls-tree -r <分支>` 的 blob 哈希对比 `git hash-object --no-filters <原文件>` 的原字节哈希（2026-10-21 实测 373/373 字节一致）；
  - 归档分支只留本地仓库：本项目 `scripts/sync_source.py` 的 `SOURCE_DIRECTORIES` 白名单不含 `docs`，镜像与压缩包天然不外发；`git switch main` 后工作区即恢复主线，收尾同步脚本照常可用。
- **本仓库背景**："原始"分支归档的是作者原版 `duanju_app` `0.2.55+61`（当时 main 为 `0.2.73+88`），来源 `D:\下载\真果·鉴.zip`（zip 顶层文件夹 `guoapp/` 已去掉）。

> 修订 2026-10-21：哈希级复验确认归档字节保真（`git ls-tree` blob 哈希 vs 源文件 `git hash-object --no-filters`，373/373 一致）；修正初版"仓库内按 LF 规范存储"的错误推断——实测 blob 保留作者原字节；新增 git archive 导出假差异坑与权威验证方法。