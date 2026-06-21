# Nexus 4.5.0 稳定性修复 — 2026-05-01

## 问题现象

服务每 2-3 天出现一次故障：
- 端口正常监听，网页能打开
- 终端显示 "Connecting..." 或持续加载
- 偶尔表现为画面冻结（能打字但看不到输出）

## 根因分析

### P0：execSync 阻塞事件循环（主因）

`server.js` 中有 43 处 `execSync`/`execFileSync` 同步调用 tmux 命令。

Node.js 是单线程的，同步调用会阻塞整个事件循环。当 tmux server 因 session 累积变慢时，这些调用阻塞数秒，导致：
- HTTP 请求挂起（页面持续加载）
- WebSocket 升级无法完成（"Connecting..."）
- 已连接的 WebSocket 停止推送数据（终端冻结）

### P1：PTY 退出后客户端被遗弃

`ensureWindowPty` 中的 `tmux attach-session` 进程退出后：
1. `ptyMap.delete()` 销毁了 entry 及其 clients Set
2. 重建 PTY 时创建了空的 clients Set
3. 旧的 WebSocket 连接仍在，但收不到任何输出
4. 表现：终端画面冻结

### P2：WebSocket 无心跳

通过代理/NAT 远端访问时，空闲连接会被静默丢弃。没有 ping/pong 检测，浏览器不知道连接已断，不会触发重连。

## 修复内容

### 1. execSync → async（41 处）

新增两个 helper 函数将同步调用包装为 Promise：

```javascript
const execAsync = (cmd, opts) => new Promise(...)
const execFileAsync = (cmd, args, opts) => new Promise(...)
```

所有 API handler 改为 async，调用改为 `await execAsync(...)`。

保留 2 处启动时的 execSync（不影响运行时）。

涉及 handler：
- `POST /api/windows`
- `POST /api/sessions`
- `GET /api/version`
- `GET /api/session-cwd`
- `GET /api/projects`（含循环内调用）
- `POST /api/projects`
- `POST /api/projects/:name/channels`
- `POST /api/projects/:name/activate`
- `POST /api/projects/:name/rename`
- `DELETE /api/projects/:name`
- `DELETE /api/sessions/:id`
- `POST /api/sessions/:id/attach`
- `POST /api/tasks`
- `POST /api/files/upload`
- `GET /api/files`
- `DELETE /api/files/all`
- `ensureWindowPty`
- Telegram webhook 内部调用

### 2. PTY 客户端迁移（ensureWindowPty onExit）

ptyProc.onExit 处理逻辑改为：
1. 退出前保存 entry.clients 和 entry.clientSizes
2. 删除旧 entry
3. 重建 PTY 后将旧客户端迁移到新 entry
4. 立即推送 lastOutput 避免黑屏

### 3. WebSocket 心跳（wss.on('connection')）

每个连接启动 30 秒间隔的 ping：
- 收到 pong 标记存活
- 超时未 pong 调用 ws.terminate() 断连
- 浏览器检测到断连后自动触发重连逻辑（前端已有指数退避重连）
- 连接关闭时清理 heartbeat interval

## 改动统计

```
server.js | 149 insertions(+), 119 deletions(-)
```

完整改动见本分支 commit 历史（a2c256b / 4a87f78 / 9143b3d 等）。

## 服务管理

### 进程管理：macOS launchd

服务通过 macOS 原生 launchd 管理，plist 位于：

```
~/Library/LaunchAgents/com.nexus4cc.plist
```

特性：
- **开机自启**：`RunAtLoad`
- **崩溃自动重启**：`KeepAlive`，进程退出后 launchd 秒级拉起
- **日志**：stdout → `logs/nexus-stdout.log`，stderr → `logs/nexus-stderr.log`

### 日常操作

| 场景 | 命令 |
|---|---|
| 重启服务（更新代码后） | `kill $(launchctl list \| grep com.nexus4cc \| awk '{print $1}')` |
| 查看状态 | `launchctl list \| grep nexus` |
| 看实时日志 | `tail -f logs/nexus-stdout.log` |
| 看错误日志 | `tail -f logs/nexus-stderr.log` |

**重启不需要 `launchctl unload/load`**。`KeepAlive` 保证杀进程后 launchd 自动拉起新进程，新代码即刻生效。

`launchctl unload/load` 仅用于：
- 永久停止服务
- 修改 plist 配置后重新加载

### 龙虾（远程管理）操作备忘

更新代码后重启：
```bash
cd /Users/zhiweiyin/repo_ds1600/nexus4cc
git pull  # 或其他更新方式
kill $(launchctl list | grep com.nexus4cc | awk '{print $1}')
# 无需其他操作，launchd 会自动拉起
```

## 注意事项

- 未改动前端代码
- 未改动 .env 配置
- 未改动端口绑定（仍然是 `0.0.0.0:59000`）
- 已弃用 PM2，改用 launchd 直接管理
