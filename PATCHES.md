# 莲心本地补丁记录（PATCHES）

> 本文件记录针对 neteasecli / mpv 的本地化补丁，用于团队与后续升级时快速找回。
> ⚠️ `npm update neteasecli` 会覆盖全局包改动，升级后需重新打补丁。

## P2-2：neteasecli mpv 实例复用补丁（2026-09-24）

- **文件**：`%APPDATA%\\npm\\node_modules\\neteasecli\\dist\\player\\mpv.js`
  （本机实际路径：`C:\\Users\\guoyuxin\\AppData\\Roaming\\npm\\node_modules\\neteasecli\\dist\\player\\mpv.js`）
- **改动 1**：`spawn('mpv', ...)` 参数新增 `--keep-open=yes`，歌曲播完 mpv 不退出，供实例复用。
- **改动 2**：新增 `async load(url, title)` 方法（位于 `play()` 与 `pause()` 之间）：
  - mpv 运行中 → `setProperty('media-title', ...)` + `sendCommand(['loadfile', url])` 无缝换流，返回 `{ reused: true }`；
  - 未运行或 IPC 失败 → 回退 `this.play(url, title)`，返回 `{ reused: false }`。

### 补丁片段（供重打）
```js
// spawn 参数（改动 1）
'--keep-open=yes',

// 新增 load 方法（改动 2），插入在 play() 与 pause() 之间
async load(url, title) {
    // P2-2: reuse the resident mpv instance via IPC loadfile for
    // gapless track switches; fall back to a fresh spawn when idle.
    if (!(await this.isRunning())) {
        await this.play(url, title);
        return { reused: false };
    }
    try {
        await this.setProperty('media-title', title || 'neteasecli');
        await this.sendCommand(['loadfile', url]);
        return { reused: true };
    }
    catch (error) {
        await this.play(url, title);
        return { reused: false };
    }
}
```

## server.js 本地优化（批1 + 批2，2026-09-24）

- 批1（提交 `0756cb6`，莲心主仓库）：播放延迟优化与 mpv 退出清理。
- 批2（本仓库）：neteasecli 模块 API（懒加载缓存 `neteaseModule()`）+ mpv 实例复用（`mpvPlayer.load()`），详见 `docs/网易云合一入口实现方案.md` 第 11 节。


## P3：残留 mpv 清理（2026-09-29）

修复「歌实际播放了但工具报错 + 同歌多通道同播」问题。根因：旧会话 detached 启动的
mpv 不会随退出结束，持续独占单实例命名管道 `\\.\pipe\neteasecli-mpv`，导致新会话
spawn 的 mpv 无法绑定同名管道（IPC 检测失败、报 "mpv started but IPC socket not
available"），但 mpv 仍在出声，且每次失败重试都再 spawn 一个新 mpv 造成多通道叠加。

- **neteasecli `dist/player/mpv.js`**：`play()` 在 `stop()` 后新增
  `await this.forceKillStaleMpv()` 兜底——IPC quit 连不上时，按可执行路径
  （netease-music-mcp-main）或命令行特征（neteasecli-mpv）强杀残留 mpv，确保新实例
  能绑定管道。新增 `async forceKillStaleMpv()` 方法。
- **`src/server.js`**：新增 `cleanupStaleMpvAtStartup()`，`main()` 在 stdio 托管 8765
  前调用，启动时清理跨会话残留 mpv（同样按路径/管道特征精确匹配，不误杀用户自开 mpv）。

### 补丁片段（供重打）
```js
// mpv.js play() 内，stop() 之后：
await this.forceKillStaleMpv().catch(() => { });
// mpv.js 新增方法：
async forceKillStaleMpv() {
    return new Promise((resolve) => {
        if (process.platform !== 'win32') {
            try {
                const proc = spawn('pkill', ['-f', 'neteasecli-mpv']);
                proc.on('close', () => resolve());
                proc.on('error', () => resolve());
            } catch { resolve(); }
            return;
        }
        try {
            const ps = [
                "$t = Get-CimInstance Win32_Process -Filter \"Name='mpv.exe' or Name='mpv.com'\" | Where-Object { ($_.ExecutablePath -like '*netease-music-mcp-main*') -or ($_.CommandLine -like '*neteasecli-mpv*') }",
                "$t | ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }",
            ].join('; ');
            const proc = spawn('powershell.exe', ['-NoProfile', '-NonInteractive', '-Command', ps], { windowsHide: true });
            proc.on('close', () => resolve());
            proc.on('error', () => resolve());
        } catch { resolve(); }
    });
}
```

## P3-1：8765 Web 播放器"MCP 间歇性失效"修复（2026-10-15）

背景：`logs/debug.log` 中出现过"8765 不在线 → 自动拉起 → 拉起超时"连续 7 次（每次约 9s，
跨 3.5 分钟）的事故。根因之一在本文件负责的 Node 侧：Web 播放器有两个主人、`shutdown` 会把
8765 弄下线、启动期的 PowerShell 抢占既慢又多余。详细诊断见
`docs/网易云MCP失效诊断报告_20261015.md`。

- **文件**：`src/server.js`
- **改动 1（重要）**：`shutdown` 工具不再调用 `closeWebPlayerBestEffort()`。
  它现在只停播放 + 清听歌状态，**保留 8765**。原因：桥接（`api_server.py`）用 HTTP 探测 8765
  判活，关掉之后莲心的所有音乐操作都会失败，而工具返回值仍是 `success: true`，莲心无从察觉。
- **改动 2（重要）**：`main()` stdio 分支去掉 `closeStandaloneWebPlayerBestEffort(8765)`，
  改成"先到先得"：端口已在监听就复用，不杀对方。
  原来两条拉起路径（stdio 托管 / `--web-player` 独立）会互相 Stop-Process，导致
  "新起的 node 立刻 EADDRINUSE 静默退出"，日志里只能看到"拉起超时"。
- **改动 3**：新增 `GET /api/ping` 轻量探针（只回进程存活，不触碰 mpv IPC）。
  `/api/status` 要问 mpv，最坏 ~3s，外部用 0.8s 超时判活必然误报"离线"。
- **改动 4**：新增 `isPortListening(port)` 端口级判活工具函数（`net.createConnection` + 500ms 超时）。
- **改动 5**：新增 `import net from "node:net"`。
- **改动 6**：`probeTrackPlayable()` 增加负面结果缓存 `PROBE_NEG_CACHE_MS = 60s`。
  原来只缓存"可播"结果，歌单播完触发整队列回绕探测时会对同一批 VIP/无版权曲目反复打网易云。

### 补丁片段（供重打）
```js
// 1) 顶部新增
import net from "node:net";

// 2) 常量区
const PROBE_NEG_CACHE_MS = 60 * 1000;  // 不可播/VIP 结果缓存 60s

// 3) handleWebApi 内，/api/status 之前插入
if (req.method === "GET" && url.pathname === "/api/ping") {
  jsonResponse(res, 200, { success: true, ok: true, uptime: Number(process.uptime().toFixed(3)), webPlayerPort, stateVersion: serverStateVersion });
  return;
}

// 4) isPortListening（放在 closeWebPlayerBestEffort 之前）
async function isPortListening(port, timeoutMs = 500) { /* net.createConnection + once(connect/timeout/error) */ }

// 5) shutdown 工具体
const playback = await stopPlaybackBestEffort();
return textResult({ success: true, message: "Stopped playback and cleared the listening state. The local web player service stays available for future calls.", playback, webPlayer: { success: true, closed: false, kept: true } });

// 6) main() stdio 分支
await cleanupStaleMpvAtStartup();
if (await isPortListening(8765)) {
  console.error("netease-music-mcp web player: 8765 已在监听，复用现有播放器");
} else {
  const web = await startWebPlayer(8765);
  console.error("netease-music-mcp web player: " + web.url);
}

// 7) probeTrackPlayable 的缓存命中改成 ok/negative 双 TTL，失败时也写缓存
```
