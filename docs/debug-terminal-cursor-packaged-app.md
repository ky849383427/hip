# 终端光标问题排查记录

> 日期：2025-01  
> 问题：打包后的 hip 应用中，vim 无法正常操作光标（开发模式正常）

## 问题现象

- `yarn tauri dev` 开发模式：vim 正常工作
- `yarn tauri build` 打包后：vim 光标不可见或无法操作
- 普通 shell 命令输入正常，仅 vim 等全屏 TUI 应用异常

## 排查方向与尝试

### 1. WebGL 渲染问题 ❌（非根因）

**假设**：WebGL addon 在打包后的 Tauri WebView2 中渲染异常

**尝试**：
- 将 `TERMINAL_WEBGL_DEFAULT` 从 `true` 改为 `false`
- 用户手动在 Settings 中开关 WebGL 测试

**结果**：问题依旧，排除 WebGL

**经验**：WebGL 问题通常表现为画面撕裂、残影，而非完全无法操作光标

### 2. xterm.js 已知 issue 调研

搜索了多个 xterm.js 相关 issue：

| Issue | 内容 | 相关性 |
|-------|------|--------|
| [#5847](https://github.com/xtermjs/xterm.js/issues/5847) | WebGL 在 Tauri/WKWebView 中的残影问题 | 低（已排除 WebGL） |
| [#5241](https://github.com/xtermjs/xterm.js/issues/5241) | WebGL cursor alpha 被忽略 | 低 |
| [#2581](https://github.com/xtermjs/xterm.js/issues/2581) | DOM renderer 下 vim 光标消失（2019 年已修复） | 低 |

### 3. Tauri 焦点处理问题 ✅（关键方向）

**假设**：打包后的 WebView2 焦点处理与开发模式不同

**关键发现**：

从 [paneflow 项目](https://github.com/ArthurDEV44/paneflow/commit/926387b) 找到类似修复：
> xterm.js 需要内部 textarea 获取焦点才能捕获键盘输入。在 Tauri WebView 中，初始焦点可能不会保持。

**Tauri 相关 issue**：
- [#15624](https://github.com/tauri-apps/tauri/issues/15624)：Windows 上 Alt+Tab 后 WebView2 键盘焦点丢失
- [#5464](https://github.com/tauri-apps/tauri/issues/5464)：WebView2 焦点问题

**根因分析**：
1. xterm.js 依赖内部隐藏的 `<textarea>` 捕获键盘输入
2. Tauri 打包后的 WebView2 焦点机制与开发模式不同
3. `term.focus()` 调用时 WebView 可能尚未准备好

### 4. 透明窗口配置

`tauri.conf.json` 中的配置可能影响焦点行为：
```json
{
  "transparent": true,
  "titleBarStyle": "Overlay",
  "hiddenTitle": true,
  "windowEffects": {
    "effects": ["sidebar", "acrylic", "mica"],
    "state": "followsWindowActiveState"
  }
}
```

## 最终解决方案

### 文件：`src/components/artifact/XtermSurface.tsx`

#### 改动 1：延迟重试焦点

```typescript
// 原代码
term.focus()

// 新代码：在 Tauri 打包的 WebView2 中，初始焦点可能不会保持
const xtermForFocus = term!
xtermForFocus.focus()
setTimeout(() => {
  if (!disposed) xtermForFocus.focus()
}, 100)
setTimeout(() => {
  if (!disposed) xtermForFocus.focus()
}, 500)
```

#### 改动 2：mousedown 使用 queueMicrotask

```typescript
// 原代码
onMouseDown={() => {
  termRef.current?.focus()
}}

// 新代码：使用 microtask 确保 mousedown 传播完成后再获取焦点
onMouseDown={() => {
  queueMicrotask(() => {
    termRef.current?.focus()
  })
  termRef.current?.focus()  // 直接调用作为兜底
}}
```

#### 改动 3：容器添加 tabIndex

```typescript
// 添加 tabIndex={-1} 使容器可获取焦点
<div
  ref={containerRef}
  tabIndex={-1}
  // ...其他属性
/>
```

## 调试经验总结

### 开发模式 vs 打包模式的差异

| 方面 | 开发模式 | 打包模式 |
|------|---------|---------|
| 前端加载 | Vite dev server (localhost) | 本地文件 |
| WebView 初始化 | 较快 | 可能较慢 |
| 焦点处理 | 宽松 | 严格 |
| GPU 渲染 | 浏览器行为 | 系统 WebView 行为 |

### 排查思路

1. **先确认问题范围**：是渲染问题还是输入问题？
   - 渲染问题 → WebGL / Canvas / 主题
   - 输入问题 → 焦点 / 键盘事件 / 事件冒泡

2. **对比开发与打包环境差异**：
   - WebView 版本/行为差异
   - 资源加载时序差异
   - 系统集成差异（窗口管理、焦点）

3. **搜索已知 issue 时的关键词组合**：
   - `[技术栈] + [问题现象] + [环境]`
   - 例如：`xterm.js vim cursor Tauri WebView2 packaged`

4. **关注时序问题**：
   - 异步操作完成后的状态是否正确
   - 延迟重试是否能解决 → 说明是时序问题

### WebGL 离线渲染问题（附带发现）

调研过程中发现 xterm.js WebGL addon 存在已知的纹理图集问题：
- Issue [#5847](https://github.com/xtermjs/xterm.js/issues/5847)：大量不同 `(glyph, fg, bg)` 组合 + 滚动会导致渲染错误
- 该问题非 Tauri 特有，Chromium/Electron 也会触发
- 临时方案：禁用 WebGL 或使用 Canvas 渲染器

## 相关代码位置

| 文件 | 作用 |
|------|------|
| `src/components/artifact/XtermSurface.tsx` | xterm.js 终端组件主体 |
| `src/components/artifact/terminalEnhancements.ts` | WebGL/Unicode11 addon 加载 |
| `src/components/artifact/terminalTheme.ts` | 终端主题配置 |
| `src-tauri/tauri.conf.json` | Tauri 窗口配置 |

## 最新修复（2025-02）

### 1. 启用 Devtools 功能

**文件**：`src-tauri/Cargo.toml`

```toml
tauri = { version = "2", features = ["macos-private-api", "tray-icon", "image-png", "devtools"] }
```

打包后可使用 `Ctrl+Shift+I` 打开开发者工具。

### 2. 添加调试覆盖层

**文件**：`src/components/artifact/XtermSurface.tsx` + `packages/protocol/src/hip-config.ts`

新增 `[terminal].debug` 配置项，启用后显示：
- Focus: textarea 是否获得焦点
- Textarea: 是否是当前焦点元素
- Cursor: 光标是否可见
- Key: 最后按下的键

**配置方法**：
```toml
# ~/.hip/config/hip.toml
[terminal]
debug = true
```

### 3. 详细的调试指南

参见：`docs/vim-cursor-fix-guide.md`

包含：
- 完整的调试步骤
- 常见问题诊断
- 高级调试技巧
- 已知的 xterm.js 限制

## 待进一步验证

- [x] 透明窗口 (`transparent: true`) 是否是焦点问题的诱因 → 已禁用
- [x] `windowEffects` 配置是否影响键盘输入 → effects 数组已清空
- [ ] macOS 上是否存在相同问题
- [x] 用户反馈的 vim 具体症状 → 需要用户运行调试模式确认

## 下一步计划

1. **用户测试**：
   - 启用 `[terminal].debug = true`
   - 运行打包后的应用
   - 打开 vim 并记录调试信息
   - 截图并反馈

2. **如果问题 2. 如果问题仍然存在**：
   - 检查开发者工具 Console 日志
   - 捕获终端转义序列
   - 对比开发模式和打包模式的差异

3. **备选方案**：
   - 尝试 neovim（更好的终端支持）
   - 禁用 vim 的 guicursor 设置
   - 使用 Canvas 渲染器代替 WebGL
