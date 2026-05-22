# crs-statusline

[English](./README.en.md)

一个供 [Claude Relay Service](https://github.com/zhengyb/claude-relay-service) 用户使用的 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 双行状态栏插件。第一行显示本地会话信息,第二行显示上游 Claude 账号的 OAuth 配额与你的 API Key 当日费用。

```
Sonnet · my-project · $0.35 · 12m12s
Upstream Usage: 5h 16% (3h29m), 7d 43% (3d), sonnet 21% (3d); My Daily Usage: $14.85/$200
```

- **顶部行** —— 取自 Claude Code 注入到 statusline 脚本 stdin 的 JSON,每次渲染实时计算:
  - `model.display_name` · `workspace.current_dir`(只取末级目录名) · `$cost.total_cost_usd` · `cost.total_duration_ms`(自动格式化为 `Xs` / `XmYs` / `XhYm`)
  - 任一字段缺失时跳过该段;四个全缺则不输出顶部行。
- **底部行** —— 取自 relay 端 `/v1/session-usage` 端点,本地缓存 10 秒:
  - **Upstream Usage**:当前上游 Claude 账号的 5h / 7d / sonnet 三个 OAuth 窗口利用率与重置剩余时间(数据由 relay 转发自 `api.anthropic.com/api/oauth/usage`)
  - **My Daily Usage**:当日(按倍率计算)费用 / API Key 的日限额(`$NA` 表示未设限额)

## 依赖

- **Node.js 18+ 和 npm**:必须先在系统全局安装好(脚本本身零 npm 依赖,只用 Node 内置模块,但 Claude Code 启动脚本需要 `node` 命令)。可在终端执行 `node -v && npm -v` 确认。安装命令`sudo apt install nodejs npm`
- Claude Code 2.1+
- [Claude Relay Service](https://github.com/zhengyb/claude-relay-service) 后端,且 `/v1/session-usage` 端点已启用(在 relay 的 `.env` 加 `STATUSLINE_USAGE_ENABLED=true` 并重启 relay)

## 安装

### 通过 Claude Code 插件(推荐)

在 Claude Code 里依次执行:

```
/plugin marketplace add zhengyb/Claude-Code-my-statusline
/plugin install crs-statusline@crs-marketplace
/reload-plugins
/crs-statusline:setup
```

`setup` 命令会把脚本下载到 `~/.claude/crs-statusline.js`,并自动改写 `~/.claude/settings.json` 的 `statusLine` 字段。装完后重启 Claude Code 即可。

### 手动安装

```bash
mkdir -p ~/.claude
curl -fsSL -o ~/.claude/crs-statusline.js \
  https://raw.githubusercontent.com/zhengyb/Claude-Code-my-statusline/main/crs-statusline.js
```

然后在 `~/.claude/settings.json` 里加(或更新):

```json
{
  "statusLine": {
    "type": "command",
    "command": "node ~/.claude/crs-statusline.js"
  }
}
```

重启 Claude Code。

## 环境变量

脚本继承 Claude Code 进程的环境变量,需要:

| 变量 | 用途 |
|------|------|
| `ANTHROPIC_BASE_URL` | relay 地址,**须以 `/api` 结尾**(例如 `http://your-relay:3000/api`) |
| `ANTHROPIC_AUTH_TOKEN` 或 `ANTHROPIC_API_KEY` | 你的 `cr_` 前缀 API Key |

通常你在 Claude Code 用 relay 时这两个变量已经设过了,本插件不需要任何额外配置。

## 各种显示状态

| 场景 | 输出 |
|------|------|
| 上游 + 费用都有 | `Upstream Usage: 5h 42% (2h13m), 7d 18% (4d), sonnet 9% (4d); My Daily Usage: $1.23/$10` |
| 未设日限额 | `… ; My Daily Usage: $121.10/$NA` |
| 服务端返回旧快照 | `Upstream Usage: ~5h 90% (1m), …`(`~` 前缀表示数据陈旧) |
| API Key 没解析到 Claude 账号 | `Upstream Usage: (暂无数据); My Daily Usage: …` |
| 解析到的账号不是 OAuth(Setup Token / Console) | `Upstream Usage: (账号无配额数据); …` |
| relay 不可达或环境变量缺失 | `Claude —` |

## 工作原理

1. Claude Code 每次刷新状态栏都会启动 `node ~/.claude/crs-statusline.js`,把一段 JSON(`session_id` / model / workspace / cost)喂到脚本 stdin。
2. 脚本解析 stdin 拼出顶部行。
3. 查本地缓存(按 `session_id` 分文件,位于 `${tmpdir}/claude-relay-statusline-*.json`),10 秒内命中直接打印。
4. 缓存未命中则 `GET {ANTHROPIC_BASE_URL}/v1/session-usage?session={session_id}`(超时 2 秒),格式化结果并写缓存。
5. **任何异常都被吞掉**,脚本永远 `exit 0`、永远输出一行有意义的文本,以保证状态栏永远不会拖垮 Claude Code。

完整源码就一个文件 —— [`crs-statusline.js`](./crs-statusline.js),建议安装前先扫一眼再装。

## 许可

MIT —— 见 [LICENSE](./LICENSE)。
