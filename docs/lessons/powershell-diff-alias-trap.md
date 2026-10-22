# PowerShell 的 diff 是 Compare-Object 别名，验证文件一致会得假结论

## 结论速览

- **最终方案**：验证两文件内容一致用 Git Bash 的 `diff <a> <b>`（无输出 = 一致）加 `cmp <a> <b>`（无输出 = 字节一致）双重确认；PowerShell 内用 `Get-FileHash <a>,<b>` 比哈希或 `fc.exe`（真文件比较器）。
- **适用条件**：Windows PowerShell / pwsh 7（`diff` 别名至今仍指向 Compare-Object，先 `Get-Alias diff` 确认）。

## ⛔ 用 diff 验证文件一致性，输出怪表格差点误判"字节级一致"（2026-10-21）

- **失败现场（原文摘录）**：
  ```
  PS> diff CLAUDE.md AGENTS.md; if ($?) { "diff 无输出=字节级一致" }

  InputObject SideIndicator
  ----------- -------------
  AGENTS.md   =>
  CLAUDE.md   <=
  diff 无输出=字节级一致
  ```
- **报错稳定片段**：无报错——这正是最险之处；可识别的稳定特征是输出表头 `InputObject SideIndicator` 与成对的 `<=` / `=>` 标记。
- **错误原因**：PowerShell 的 `diff` 是 `Compare-Object` 的别名（`Get-Alias diff` 可查）；它按对象比较、需要 `-ReferenceObject`/`-DifferenceObject` 参数或管道输入，直接传两个字符串路径时比较的是**这两个字符串字面量**（"CLAUDE.md" ≠ "AGENTS.md"，于是 `<=`/`=>` 各一行），完全没有读文件内容。另外 `$?` 只是"上一命令成功与否"，不是"无差异"标志，不能用来判一致性。
- **为何不可再采用**：任何在 PowerShell 下用 `diff <文件> <文件>` 做"文件内容对比"的场景，比较的都是路径字符串本身，结论与文件内容无关，必然错误。
- **替代方案**：Git Bash `diff <a> <b>`（POSIX diff，无输出 = 一致，退出码 0）与 `cmp <a> <b>`（字节级，无输出 = 一致）双重验证；或 PowerShell `Get-FileHash <a>,<b>` 比哈希、`fc.exe <a> <b>`。
- **判定时效**：pwsh 7.x（2026-10-21 实测）；`diff` → Compare-Object 别名是 PowerShell 长期行为，升级后用前仍应 `Get-Alias diff` 确认。

## ✅ Git Bash diff + cmp 双重验证文件字节一致（2026-10-21）

- **为何值得记**：与上方 ⛔ 同一场景的可靠做法；当时用于 neat-freak 的规则文件配对验收（CLAUDE.md vs AGENTS.md 全文 diff 必须为零）。
- **最终方案**（原文）：
  ```bash
  diff <a> <b> && echo "=== 全文 diff 为零 ==="
  cmp <a> <b> && echo "=== 字节级一致 ==="
  ```
- **为什么这样做**：Git Bash 的 `diff` 是 POSIX diff，传文件路径即读文件内容、无输出且退出码 0 表示一致；`cmp` 逐字节比较作双重确认，两者无输出共同构成"一致"的可靠判据。
- **适用条件**：Windows Git Bash（含 git for Windows 自带工具链）。
- **验证证据**（2026-10-21 实跑）：`diff CLAUDE.md AGENTS.md` 与 `cmp CLAUDE.md AGENTS.md` 均无差异输出，echo 到达两条一致标记；此前 PowerShell 侧用 `Copy-Item` 字节级复制后以此法验收配对通过。
- **交叉验证**：单 agent 单次跑通，无多轮审查——如实标注，非"精粹"。
- **关键步骤**：优先 Git Bash 验证 → 两条命令都无输出才算一致 → 有输出时先看是行尾差异（`od -c` / `--strip-trailing-cr`）还是内容差异。
- **易错点**：不要在 PowerShell 里敲同样的 `diff` 命令（见上方 ⛔ 条目）；文件行尾 LF/CRLF 不同会让 diff 全文报差异，但可能只是行尾问题（先 `diff --strip-trailing-cr` 排除）。