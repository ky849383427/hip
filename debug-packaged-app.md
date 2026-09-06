# 打包应用调试指南

## 问题现状

之前的修复已应用：
1. ✅ 延迟重试焦点 (100ms, 500ms)
2. ✅ mousedown 使用 queueMicrotask
3. ✅ 容器添加 tabIndex={-1}
4. ✅ 强制刷新 (600ms)

但问题可能仍然存在，需要进一步诊断。

## 下一步调试计划

### 1. 启用 Devtools（生产模式）

在 `src-tauri/Cargo.toml` 中添加 devtools 功能：

```toml
tauri = { version = "2", features = ["macos-private-api", "tray-icon", "image-png", "devtools"] }
```

然后重新打包，使用 `Ctrl+Shift+I` 打开开发者工具。

### 2. 添加终端调试日志

在 `XtermSurface.tsx` 中添加调试代码，记录：
- 焦点状态
- 键盘事件
- vim 光标转义序列

### 3. 检查 vim 光标转义序列

vim 使用以下转义序列控制光标：
- `\x1b[?25h` - 显示光标 (DECTCEM)
- `\x1b[?25l` - 隐藏光标 (DECTCEM)

如果 xterm.js 没有正确处理这些序列，光标会不可见。

### 4. 测试步骤

1. **打包应用**：`yarn tauri build`
2. **运行打包后的应用**
3. **打开 vim**：`vim test.txt`
4. **观察**：
   - 光标是否可见？
   - 按 `i` 进入插入模式时光标变化？
   - 按 `Esc` 退回普通模式？
5. **打开 Devtools**：`Ctrl+Shift+I`
6. **检查控制台**：是否有错误？
7. **检查 Elements**：xterm 的 textarea 是否获得焦点？

### 5. 可能的根因

1. **WebView2 焦点丢失**：已修复但可能不够
2. **xterm.js textarea 未激活**：可能需要在特定时机调用 focus()
3. **vim termcap 配置问题**：TERM 变量正确但终端能力不匹配
4. **Tauri 透明窗口副作用**：虽然已禁用但可能有残留影响

### 6. 对比测试

创建一个最小测试：
- 在普通浏览器中打开 xterm.js
- 在 Tauri dev 模式下
- 在 Tauri 打包模式下

对比三者的行为差异。