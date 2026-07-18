# AutoVibe

给 CLI coding agent（Claude Code、Codex、Grok CLI…）套一层「自动点 yes」的壳，
让小型任务不被确认提问打断，一路跑到完成。

```bash
autovibe -- claude
autovibe -- codex
autovibe --max 50 --log ~/autovibe.log -- grok
```

## 先看这里：很多 CLI 自带自动模式

在套壳之前，优先用官方开关——它们比任何外壳都稳，因为不依赖「识别屏幕文字」：

| CLI | 自动化方式 |
|---|---|
| **Claude Code** | `claude --permission-mode acceptEdits`（自动接受文件编辑）；`claude --dangerously-skip-permissions`（全自动，跳过所有确认）；会话内也可用 `Shift+Tab` 切换 auto-accept；无头模式 `claude -p "任务描述"` |
| **Codex** | `codex --full-auto`（工作区内自动执行）；`codex --yolo` / `--dangerously-bypass-approvals-and-sandbox`（全自动）；或在 `~/.codex/config.toml` 里设 `approval_policy = "never"` |
| **Grok CLI** | 查看 `grok --help` 是否有 auto-approve / yolo 类开关，版本迭代较快 |

这些开关的共同点：**在源头关掉提问**，而不是事后帮你按键。全自动模式请只在
可信目录、有 git 兜底的仓库里用。

## 为什么 `yes 1 | claude` 不行

这些 CLI 是 TUI 程序：它们检测 stdin 是否为终端、直接以 raw 模式从 tty 读键盘。
用管道喂输入会导致 TTY 检测失败（进入非交互模式或直接报错），所以经典的
`yes` 命令帮不上忙。正确做法是用**伪终端（PTY）**包一层——这正是 `autovibe`
做的事。

## autovibe 的工作原理

1. 用 PTY 启动目标 CLI，你看到的界面、能敲的键盘和平时完全一样；
2. 壳持续镜像输出，并维护一份去掉 ANSI 转义码的「屏幕尾部」文本；
3. 当输出**静默一小段时间**（默认 1 秒，说明界面停在提问上）且尾部命中已知
   提问模式时，自动写入答案：
   - `❯ 1. Yes` 这类编号菜单 → 按 `1`
   - `(y/n)` / `[Y/n]` → 发 `y⏎`
   - `Press enter to continue` → 发 `⏎`
4. **安全护栏**：提问上下文若命中危险模式（`rm -rf`、`git push --force`、
   `git reset --hard`、`sudo`、`drop table`、"don't ask again" 选项等），
   壳会响铃并把提问留给你，绝不自动作答；
5. 任何时候按 **Ctrl-]** 可以现场开/关自动模式；你手动敲键后壳会短暂让路，
   不和你抢同一个提问。

## 安装

只依赖 Python 3 标准库，macOS / Linux 通用（Terminal、Ghostty、iTerm2 都可以）：

```bash
git clone https://github.com/djzoom/AutoVibe.git
chmod +x AutoVibe/autovibe
ln -s "$PWD/AutoVibe/autovibe" /usr/local/bin/autovibe   # 或加进 PATH
```

## 用法与选项

```
autovibe [选项] -- <命令> [参数...]

--quiet SEC        输出静默多久才认定「停在提问上」（默认 1.0 秒）
--cooldown SEC     两次自动作答之间的最小间隔（默认 2.0 秒）
--max N            最多自动作答 N 次后转手动（0 = 不限）
--rule REGEX=SEND  追加自定义规则，优先于内置规则；SEND 里可写 \r 表示回车（可重复）
--deny REGEX       追加危险模式：命中则不自动作答（可重复）
--config PATH      JSON 配置文件（默认 ~/.config/autovibe.json）
--no-default-rules 只用 --rule/--config 里的规则
--no-default-deny  去掉内置危险模式（不建议）
--log PATH         把每次自动作答记录到文件
--dry-run          只记录“本来会答什么”，不真正发送——首次用于新 CLI 时先跑这个
```

示例：

```bash
# 第一次包一个新 CLI，先观察它的提问长什么样
autovibe --dry-run -- grok

# 给 Codex 的确认提示加一条“直接回车选默认项”的规则
autovibe --rule 'Allow command\?=\r' -- codex

# 最多自动确认 30 次，超过就交还人工，防止彻底失控
autovibe --max 30 -- claude
```

### 配置文件（`~/.config/autovibe.json`）

```json
{
  "rules": [
    { "name": "codex-allow", "pattern": "Allow command\\?", "send": "\r" }
  ],
  "deny": [
    "(?i)git\\s+push\\s+origin\\s+main"
  ]
}
```

## 其他备选方案

- **expect**：`expect -c 'spawn claude; expect -re {1\. Yes} {send 1}; interact'`
  ——适合一次性脚本，但规则多了难维护，且 `interact` 前的窗口容易漏提问。
- **tmux**：把 agent 跑在 tmux 会话里，另开脚本轮询
  `tmux capture-pane -p` 找提问、`tmux send-keys 1` 作答。思路和 autovibe
  相同，但多了 tmux 依赖；好处是可以远程/后台管理多个会话。
- **官方无头模式**：`claude -p`、`codex exec` 等本身就是为无人值守设计的，
  配合 `--dangerously-skip-permissions` / `--full-auto` 是最省事的“连贯跑完”
  方案，代价是放弃逐步确认。

## 注意

自动确认等于把决策权交给 agent。建议：只在有 git 版本控制的目录里用、
保留默认 deny 规则、加上 `--max` 和 `--log`，并且永远不要让它替你选
"don't ask again"。
