# Vim 光标问题修复指南

> 问题：打包后的 hip 应用中，vim 无法正常操作光标（开发模式正常）

## 📋 修复清单

### ✅ 已完成的修复

1. **焦点延迟重试** (XtermSurface.tsx)
   - 初始 focus() 后，100ms 和 500ms 后重试
   - 确保 WebView2 中的 textarea 获得焦点

2. **mousedown 优化** (XtermSurface.tsx)
   - 使用 queueMicrotask() 确保事件传播完成
   - 添加直接调用作为兜底

3. **容器可聚焦** (XtermSurface.tsx)
   - 添加 tabIndex={-1} 使容器可获取焦点

4. **强制刷新** (XtermSurface.tsx)
   - 600ms 后调用 term.refresh() 刷新光标

5. **Devtools 已启用** (Cargo.toml)
   - 添加 `devtools` 功能，打包后可用 Ctrl+Shift+I 调试

6. **调试覆盖层** (XtermSurface.tsx + hip-config.ts)
   - 新增 `[terminal].debug` 配置项
   - 显示焦点、光标、键盘状态

## 🔧 调试步骤

### 步骤 1：启用调试模式

在 `~/.hip/config/hip.toml` 中添加：

```toml
[terminal]
debug = true
```

### 步骤 2：打包并运行

```bash
# 打包应用
yarn tauri build

# 运行打包后的应用
# Windows: src-tauri/target/release/hip.exe
```

### 步骤 3：测试 vim

1. 打开终端
2. 运行 `vim test.txt`
3. 观察：
   - 光标是否可见？
   - 按 `i` 进入插入模式，光标变化？
   - 按 `Esc` 退回普通模式？

### 步骤 4：查看调试信息

调试覆盖层会显示：

```
Focus: ✅/❌      # xterm textarea 是否获得焦点
Textarea: ✅/❌   # textarea 是否是当前焦点元素
Cursor: 👁️/🚫    # 光标是否可见
Key: a            # 最后按下的键
```

### 步骤 5：打开开发者工具

按 `Ctrl+Shift+I` 打开 WebView2 开发者工具：

1. **Console 标签**：查看错误和日志
2. **Elements 标签**：检查 DOM 结构
3. **检查 textarea**：
   - 找到 `.xterm-helper-textarea`
   - 检查是否有 `focus` 状态
   - 检查样式（display, visibility）

## 🐛 常见问题诊断

### 问题 1：光标完全不可见

**症状**：vim 打开后看不到光标

**可能原因**：
- DECTCEM 转义序列未正确处理
- xterm.js 的 cursorBlink 设置问题

**诊断方法**：
```bash
# 在 vim 中运行
:set cursorline?
:set guicursor?
```

**解决方案**：
在 vim 配置中添加：
```vim
" ~/.vimrc
set guicursor=a:blinkon0
set nocursorline
```

### 问题 2：光标位置错误

**症状**：光标显示但位置不对

**可能原因**：
- 终端尺寸计算错误
- 字体度量问题

**诊断方法**：
```bash
# 检查终端尺寸
echo $COLUMNS x $LINES
tput cols
tput lines
```

**解决方案**：
确保字体在 Terminal open() 前加载完成（已修复）

### 问题 3：键盘输入无响应

**症状**：光标可见但按键无效

**可能原因**：
- textarea 未获得焦点
- 键盘事件被拦截

**诊断方法**：
- 调试覆盖层显示 `Focus: ❌`
- 检查 textarea 是否是 activeElement

**解决方案**：
点击终端区域或使用快捷键重新聚焦

### 问题 4：插入模式光标不变化

**症状**：始终显示块状光标

**可能原因**：
- guicursor 设置未生效
- xterm.js 未正确处理光标形状序列

**诊断方法**：
```bash
# 在 vim 中检查
:echo &guicursor
```

**解决方案**：
在 vim 配置中明确设置：
```vim
set guicursor=n-v-c:block,i-ci-ve:ver25,r-cr:hor20,o:hor50
```

## 🔬 高级调试

### 捕获终端转义序列

在 xterm.js 中添加转义序列日志：

```typescript
// XtermSurface.tsx 中添加
term.onData((data) => {
  if (data.includes('\x1b[')) {
    console.log('Escape sequence:', JSON.stringify(data))
  }
})
```

### 对比开发模式和打包模式

1. **开发模式**：
   ```bash
   yarn tauri dev
   # 检查 vim 行为
   ```

2. **打包模式**：
   ```bash
   yarn tauri build
   # 运行打包后的应用，检查 vim 行为
   ```

3. **记录差异**：
   - 焦点时机
   - 键盘事件
   - 光标转义序列

### 检查 TERM 环境变量

```bash
# 在终端中运行
echo $TERM
# 应该显示：xterm-256color
```

如果显示其他值，检查 `src-tauri/src/pty.rs` 中的配置。

## 📊 已知的 xterm.js 限制

1. **WebGL addon 纹理图集问题** (Issue #5847)
   - 大量 glyph 组合 + 滚动会导致渲染错误
   - 临时方案：禁用 WebGL

2. **Tauri WebView2 焦点问题** (Issue #15624)
   - Windows 上 Alt+Tab 后焦点丢失
   - 已通过延迟重试修复

3. **透明窗口副作用**
   - `transparent: true` 可能影响焦点
   - 当前已禁用透明窗口

## 🎯 下一步行动

如果问题仍然存在：

1. **收集更多信息**：
   - 调试覆盖层截图
   - 开发者工具 Console 日志
   - vim 版本和配置

2. **尝试替代方案**：
   - 使用 neovim（更好的终端支持）
   - 禁用 vim 的 guicursor 设置

3. **提交 issue**：
   - 包含调试信息
   - 包含复现步骤
   - 包含环境信息

## 📚 相关资源

- [xterm.js 文档](https://xtermjs.org/)
- [Tauri WebView2 调试](https://v2.tauri.app/develop/debug/)
- [vim 光标设置](https://vimhelp.org/options.txt.html#%27guicursor%27)
- [DECTCEM 转义序列](https://vt100.net/docs/vt510-rm/DECTCEM.html)