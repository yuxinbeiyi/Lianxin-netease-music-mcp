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
