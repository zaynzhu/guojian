# GitHub 推送 Connection was reset 时走本地代理

## 结论速览

- **最终方案**：`git -c http.proxy=http://127.0.0.1:7897 -c https.proxy=http://127.0.0.1:7897 push origin main`（`-c` 一次性参数，不写入 git 配置；本机代理软件监听 7897）。
- **适用条件**：直连 GitHub 报 `Recv failure: Connection was reset` 且本机代理软件在运行；端口不保证恒定，每次先探测。本条在 **Windows 临时环境**实测；用户主力开发在 macOS，其网络/代理状态未核实，跨环境需重新探测再套用。

## ✅ 直连 reset，代理 7897 临时参数推送成功（2026-10-21）

- **为何值得记**：本仓库（guojian）推 GitHub 依赖网络状态，必再遇；且本机 git 与环境变量**都没有**代理配置（`git config --get http.proxy` 为空、`env | grep -i proxy` 为空），不知道"代理软件在跑但 git 没配"这一点就无从下手。
- **最终方案**（原文）：
  ```bash
  # 1) 确认 git 侧确实没配代理
  git config --get http.proxy; git config --get https.proxy; env | grep -i proxy
  # 2) 探测本地代理端口（Clash 系 7890/7897、v2rayN 10808/10809、SSR 1080）
  for port in 7890 7897 10809 10808 1080; do
    code=$(curl -s --connect-timeout 3 -x http://127.0.0.1:$port -o /dev/null -w "%{http_code}" https://github.com)
    echo "http 代理 127.0.0.1:$port -> $code"   # 200 即可用
  done
  # 3) 临时走代理推送（不污染配置）
  git -c http.proxy=http://127.0.0.1:7897 -c https.proxy=http://127.0.0.1:7897 push origin main
  ```
- **为什么这样做**：`Recv failure: Connection was reset` 是直连 GitHub 被网络层重置，git 原样重试无意义，必须走代理；用 `-c` 传一次性参数而不是 `git config --global` 写死，是因为本地代理端口可能变化，写进全局配置会过期成毒药、且毒化所有 git 网络操作。
- **适用条件**：Windows 11 临时环境 + 本机代理软件（当前监听 7897，Clash Verge 默认混合端口）在运行；代理软件没开时此路不通，需先启动它。macOS 主力环境的代理软件与端口未核实，需先探测再套用。
- **验证证据**（2026-10-21 实跑，guojian 仓库）：
  - 直连两次均 `fatal: unable to access 'https://github.com/zaynzhu/guojian/': Recv failure: Connection was reset`
  - 探测结果：`7897 -> 200`；`7890 / 10809 / 10808 / 1080 -> 000`（不可用）
  - 代理推送 → `To https://github.com/zaynzhu/guojian`，`b0b56ce..4e0400b  main -> main`
- **交叉验证**：单 agent 单次跑通，无多轮审查——如实标注，非"精粹"。
- **关键步骤**：确认 git 无代理配置 → 探测端口拿 200 → `-c` 临时参数推送。
- **易错点**：探测时 000 表示该端口连不上（可能协议不符，如 socks-only 端口用 http 探测），200 才可用；不要把代理端口写进任何持久配置；代理软件未运行时先解决"开代理"再谈推送。