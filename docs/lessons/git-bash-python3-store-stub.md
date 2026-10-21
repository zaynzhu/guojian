# Windows 下 python3 是 Microsoft Store stub（无输出、exit 49）

## 结论速览

- **最终方案**：本机跑 Python 脚本一律用 `python`（`E:/program/tool/python`，3.12.8），不用 `python3`。
- **适用条件**：**仅 Windows**（本机为临时环境，用户主力开发在 macOS——该环境下 `python3` 通常正常，此坑不存在）；Windows 11，PATH 含 `C:\Users\<用户>\AppData\Local\Microsoft\WindowsApps`（Store stub 所在目录）；已验证 Git Bash，PowerShell 走同一 PATH 大概率相同（未单独验证）。

## ✅ python3 跑脚本静默失败 exit 49，换 python 即通（2026-10-21）

- **为何值得记**：失败时**零输出**——没有 Python 报错、没有 traceback，只有 exit code 49，极难定位；本项目 AGENTS.md 的收尾命令原文写的 `python3 scripts/finish_task.py`，照抄执行必踩，每次任务收尾都可能再遇。
- **最终方案**（原文）：
  ```bash
  python scripts/finish_task.py --message "..."   # 用 python，不用 python3
  which python3    # -> /c/Users/OMEN/AppData/Local/Microsoft/WindowsApps/python3（Store stub）
  which python    # -> /e/program/tool/python
  ```
- **为什么这样做**：Windows 上 `python3` 不是真解释器，是 Microsoft Store 的安装引导 stub（App Execution Alias）；非交互环境（重定向/管道/工具调用）下它不弹 Store 窗口，静默退出 49。`python` 才指向真实解释器。
- **适用条件**：Windows 11 特有坑（Store App Execution Alias 是 Windows 机制）；本机 `python` = `E:/program/tool/python/python.exe`（Python 3.12.8）。macOS 主力开发环境下 `python3` 一般直接可用，此条不适用。换机器先 `which python3` 确认指向再决定。
- **验证证据**（2026-10-21 实跑）：
  - `python3 scripts/finish_task.py --message "..."` → 无任何输出，`Exit code 49`
  - `python3 --version` → 同样 `exit=49`，无版本输出
  - 换 `python scripts/finish_task.py --message "..."` → 正常执行（同步 337 文件、生成源码 zip）
  - `python --version` → `Python 3.12.8`
- **交叉验证**：单 agent 单次跑通，无多轮审查——如实标注，非"精粹"。
- **关键步骤**：脚本无输出 + 非常规 exit code → 先 `which <命令>` 看是否指向 `WindowsApps` → 换真实命令重跑。
- **易错点**：不要被项目文档里写的 `python3` 字样锚定，照抄命令也必须本地代换成 `python`；stub 不打印任何错误是最迷惑的点。