---
description: 项目开发过程中的经验总结、问题排查与最佳实践
---

# 经验教训与最佳实践总结

## 遇到的问题

### 1. git commit 命令长时间处于 RUNNING 状态

**现象**：

- 执行 `git commit -m "..."` 后，命令状态一直显示 `RUNNING`
- 等待 10-30 秒后仍未完成
- 有时候实际上已经完成，但状态查询未能正确返回

**可能原因**：

- Windows 系统上 Git 处理大文件（如 index.html 约 100KB）需要更长时间
- Git 可能配置了 GPG 签名，等待签名输入
- 文件编码转换（CRLF/LF）耗时
- 网络或磁盘 I/O 延迟

**解决方案**：

- 分步执行命令，避免使用 `&&` 连接多个 Git 命令
- 使用 `git status` 或 `git log -1` 验证操作是否成功
- 增加等待时间，但不要依赖 command_status 的返回状态

---

### 2. git push 显示 "Everything up-to-date"

**现象**：

- 执行 `git push` 返回 `Everything up-to-date`
- 但本地确实有新的更改

**可能原因**：

- 之前的 `git commit` 命令没有成功执行
- 更改没有被正确 staged（`git add` 未执行）
- 之前的后台命令仍在运行，文件被锁定

**解决方案**：

- 在 push 之前先执行 `git status` 确认状态
- 确保 `git add` 和 `git commit` 都成功完成后再 push
- 使用 `git log -1 --oneline` 确认最新提交

---

### 3. AI 模型思维死循环 (Thought Loops)

**现象**：

- 模型在思考过程中反复模拟操作指令，如反复生成 `(Action)`, `(view_file)`, `(Execute)` 等文本。
- 只有思考过程（Thinking），迟迟不发出实际的工具调用（Tool Call）。
- 最终因触发 Token 长度限制而报错。

**可能原因**：

- **过度规划/犹豫**：模型在确定行动前试图在思维中“演练”输出格式，导致陷入自我重复。
- **伪日志 (Pseudo-logging)**：在思考中使用了类似 `(Tool: name)` 或 `Action: command` 的格式，这与模型训练中的某些输出模式冲突，导致模型误以为任务已完成或陷入补全循环。

**解决方案**：

- **果断行动**：一旦决定了下一步操作，立即调用工具，**不要**在思考中详细描述工具调用的格式。
- **避免伪格式**：在 `<thought>` 块中，严禁使用 `(Action)`, `(Tool)`, `(Execute)` 等类似代码或日志的格式化文本。直接用自然语言描述意图（例如：“我将读取文件” 而不是 “(Action: read_file)”）。
- **立即执行**：不要试图在一次思考中罗列太多步骤的详细执行日志，一步一步来。

---

### 4. 代码替换失败 (TargetContent Not Found)

**现象**：

- 使用 `replace_file_content` 时频繁报错 `target content not found in file`。
- 即使肉眼看起来一模一样的代码也匹配失败。

**可能原因**：

- **符号差异**：视觉小说这类项目中经常混用中英文引号（如 `"` 与 `“`、`”`），或者包含不可见的空格/制表符。
- **匹配块过大**：试图一次性替换数百行代码，任何一处细微差异（甚至是一个换行符）都会导致整个动作失败。
- **缓存过时**：在频繁修改的过程中，模型记住的“旧代码”与文件中“新代码”有了差异。

**解决方案**：

- **实时对齐**：在执行替换动作**紧前**，务必先用 `view_file` 读取目标行。**严禁**凭记忆或根据早先的读取记录进行大段替换。
- **化整为零**：不要一次替换整个对象（如 `SCENES`），而是分块替换场景 ID 或具体行。
- **精准锚点**：在 `TargetContent` 中保留具有唯一性的行号或特征字符串，减少匹配歧义。

---

### 5. 控制台输出乱码与命令挂起

**现象**：

- 执行 `run_command` (如 `Get-Content` 或 `type`) 时输出包含乱码（如 `蜑ｧ譛ｬ謨...`）。
- 命令状态长时间处于 `RUNNING` 且无任何输出返回。

**可能原因**：

- **编码冲突**：Windows 环境下控制台编码（通常是 GBK 0x936）与 HTML 文件字符集（UTF-8）不一致，导致中文显示异常。
- **管道 I/O 阻塞**：读取过大的文本文件（100KB+）到控制台缓冲区，有时会导致管道阻塞或长时间挂起。

**解决方案**：

- **优先使用工具**：读取文件内容时，**永远优先使用 `view_file`**。该工具具有更稳健的编码处理机制，且不会阻塞。
- **避免直接打印大文件**：如果必须使用命令行检查文件，请使用 `Select-Object -First 10` (PowerShell) 或 `head` (Linux) 限制输出量。

---

## 最佳实践

### 推荐的 Git 操作流程

```bash
# 步骤 1: 添加文件
git add <文件名>

# 步骤 2: 检查状态
git status

# 步骤 3: 提交（使用较长等待时间）
git commit -m "提交信息"

# 步骤 4: 验证提交
git log -1 --oneline

# 步骤 5: 推送
git push
```

### 命令执行建议

1. **分步执行**：不要用 `&&` 连接多个 Git 命令
   - ❌ `git add . && git commit -m "msg" && git push`
   - ✅ 分三步执行

2. **等待时间设置**：
   - `git add`: 2000-3000ms
   - `git commit`: 5000-10000ms（大文件可能需要更长）
   - `git push`: 10000ms

3. **状态验证**：
   - commit 后用 `git log -1` 验证
   - push 后检查返回信息是否包含 `->` 符号

4. **处理超时**：
   - 如果 command_status 返回 RUNNING，不要急于重试
   - 先用 `git status` 检查当前状态
   - 确认状态后再决定下一步操作

5. **避免死循环**：
   - 思考时使用自然语言，避免模拟工具调用的格式。
   - 不要犹豫，决定了就 Call Tool。

6. **代码替换技巧**：
   - 替换前先 `view_file` 确认。
   - 优先使用 `multi_replace_file_content` 配合多个 `ReplacementChunks`，针对性更强。
   - 避免使用包含大量中文字符的长字符串作为匹配目标，除非刚读取过。

7. **大文件处理**：
   - 减少对大文件的全文读取和覆盖。
   - 修改大文件（如 100KB+ 的 `index.html`）逻辑时，尽量分场景、分函数块进行渐进式修改。
   - 触及 Token 限制报错时，主动减少输出中的冗余代码。

---

## 常见问题排查

| 问题 | 检查命令 | 解决方法 |
|------|----------|----------|
| 不确定是否有未提交的更改 | `git status` | 查看 "Changes not staged" |
| 不确定最新提交是什么 | `git log -1 --oneline` | 确认提交信息 |
| 不确定本地是否领先远程 | `git status` | 查看 "ahead of origin" |
| push 失败 | `git remote -v` | 确认远程仓库配置 |

---

### 6. 多游戏存档冲突（file:// 协议下 localStorage 共享问题）

**现象**：

- 游戏 A 的存档出现在游戏 B 的存档列表中
- 删除存档后刷新页面，存档又“复活”了
- 不同游戏的进度、设置、已读文本等数据相互覆盖

**根本原因**：

1. **`file://` 协议限制**：浏览器将所有通过 `file://` 打开的本地 HTML 文件视为来自同一个“源”（origin），因此它们共享同一个 `localStorage` 存储空间。

2. **键名冲突**：多个游戏使用相同的 `localStorage` 键名（如 `save_slot_1`、`auto_save`），导致数据相互覆盖。

3. **自动迁移功能的副作用**：之前代码中有一个“好心办坏事”的迁移功能，会自动将旧前缀的数据**复制**到新前缀，导致删除存档后又被自动恢复。

**解决方案**：

1. **为每个游戏使用唯一的 localStorage 前缀**：

```javascript
// 在 CONSTANTS.STORAGE_KEYS 中配置独特前缀
STORAGE_KEYS: {
    AUTO_SAVE: 'xiaoshenjiu_auto_save',        // 游戏A
    SAVE_SLOT_PREFIX: 'xiaoshenjiu_save_slot_',
    // ...
}

// 另一个游戏使用不同前缀
STORAGE_KEYS: {
    AUTO_SAVE: 'lunhui_auto_save',             // 游戏B
    SAVE_SLOT_PREFIX: 'lunhui_save_slot_',
    // ...
}
```

1. **启动时删除旧数据，而非迁移**：

```javascript
// ✅ 正确做法：直接删除旧前缀数据
useEffect(() => {
    const OLD_PREFIXES = ['lunhui_', 'qijiu_', 'qiujiu_'];
    Object.keys(localStorage).forEach(key => {
        if (OLD_PREFIXES.some(prefix => key.startsWith(prefix))) {
            localStorage.removeItem(key);  // 删除，不复制
        }
    });
}, []);

// ❌ 错误做法：复制旧数据（会导致存档“复活”）
// localStorage.setItem(newKey, oldValue);
```

1. **前缀命名规范**：

| 游戏名称 | 推荐前缀 |
|---------|---------|
| 掌门他养了只小沈九 | `xiaoshenjiu_` |
| 轮回相关游戏 | `lunhui_` |

**注意事项**：

- 部署到不同域名后会自动隔离，无需担心冲突
- 同时打开多个标签页玩不同游戏时，只要前缀不同就不会干扰
- 复制引擎代码创建新游戏时，务必修改 `CONSTANTS.STORAGE_KEYS` 中的所有前缀

---

### 6. 多游戏存档冲突（file:// 协议下 localStorage 共享问题）

**现象**：

- 游戏 A 的存档出现在游戏 B 的存档列表中
- 删除存档后刷新页面，存档又"复活"了
- 不同游戏的进度、设置、已读文本等数据相互覆盖

**根本原因**：

1. **\ile://\ 协议限制**：浏览器将所有通过 \ile://\ 打开的本地 HTML 文件视为来自同一个"源"（origin），因此它们共享同一个 \localStorage\ 存储空间。

2. **键名冲突**：多个游戏使用相同的 \localStorage\ 键名（如 \save_slot_1\、\uto_save\），导致数据相互覆盖。

3. **自动迁移功能的副作用**：之前代码中有一个"好心办坏事"的迁移功能，会自动将旧前缀的数据**复制**到新前缀，导致删除存档后又被自动恢复。

---
---

### 7. 存档/读档点击无效 (2026-01-22)

**问题描述**：
用户反馈在点击“保存存档”或“读取存档”时，界面无任何反应，点击无效。

**原因分析**：
原代码在 `handleSaveSlot`、`handleLoadSlot` 和 `handleDeleteSlot` 中直接使用了浏览器原生的 synchronous（同步）方法 `window.confirm` 和 `window.alert`。

- 在某些全屏 Web 应用环境、移动端浏览器或受限的 iframe/沙箱环境中，原生的模态对话框可能被浏览器策略拦截，或者因为不属于 DOM 渲染的一部分而无法正确显示。
- 这会导致 JavaScript 执行被挂起等待用户响应，但用户看不到对话框，从而感觉 UI "死机" 或点击无效。

**解决方案**：
将所有的原生弹窗替换为游戏内自定义的异步 UI 组件。

1. **引入 Custom Dialogs**: 利用现有的 `AlertDialog` 和 `ConfirmDialog` 组件。
2. **异步化逻辑**: 将 `window.confirm` 替换为 `await showConfirm(...)`，将 `alert` 替换为 `await showAlert(...)`。
3. **状态管理**: 通过 `dialogState` 状态来控制自定义对话框的显示与隐藏，确保对话框完全在 React/Preact 的渲染循环内，不受浏览器弹窗策略影响。

**代码变更示例**：

```javascript
// Before (Native - Blocking/Unreliable)
if (window.confirm("是否覆盖?")) {
    // ...
    alert("保存成功");
}

// After (Custom - Async/Consistent)
if (await showConfirm("是否覆盖?")) {
    // ...
    await showAlert("保存成功");
}
```

**最佳实践**：

- 在 Web 游戏或沉浸式应用开发中，应尽量避免使用 `window.alert/confirm/prompt`。
- 使用自定义 UI 弹窗不仅能保证交互的可靠性，还能保持游戏美术风格的一致性。

### 8. 背景图片切换错误 (快速点击导致的状态竞争)

**现象**：
用户发现在快速点击（快进或快速阅读）时，背景图片偶尔不会按照剧情设定变换，卡在上一张图或显示错误的过渡状态。

**原因分析**：
这是一个典型的 **Race Condition (竞态条件)** 问题。
在原有的 `BackgroundLayer` 组件中：

1. 背景切换依赖于 `setTimeout` (1秒) 来完成淡入淡出。
2. 当用户点击速度超过动画时间（<1s）时，上一张图的定时器尚未执行完毕，虽然 `useEffect` 触发了清理，但逻辑中没有正确处理“中途被打断”的状态重置。
3. 导致 `nextSrc`（淡入图）和 `currentSrc`（当前底图）的状态更新错乱，新的背景请求可能被旧的定时器回调覆盖。

**解决方案**：
完全重构 `useEffect` 中的过渡逻辑，确保“清理”和“重置”的原子性。

1. **严格的资源清理**：在 `useEffect` 的 cleanup 函数中，必须使用 `cancelAnimationFrame` 和 `clearTimeout` 清除所有挂起的异步任务。
2. **强制状态同步**：每当 `src` 发生变化时，如果检测到上一次过渡还未完成，应立即重置状态，而不是等待旧的动画结束。
3. **逻辑优化**：

   ```javascript
   useEffect(() => {
       if (src !== currentSrc) {
           // 1. 立即设定新目标
           setNextSrc(src);
           setIsTransitioning(false); // 先重置动画状态

           // 2. 下一帧启动新动画
           const raf = requestAnimationFrame(() => setIsTransitioning(true));

           // 3. 设置完成回调
           const timer = setTimeout(() => {
               setCurrentSrc(src);
               // 注意：这里不需要手动 setNextSrc(null)，留给下一次 render 的 else 分支处理更安全
           }, 1000);

           // 4. ✅ 关键：清理函数
           return () => {
               cancelAnimationFrame(raf);
               clearTimeout(timer);
           };
       } else {
           // 5. 稳定状态：清理过渡痕迹
           setNextSrc(null);
           setIsTransitioning(false);
       }
   }, [src, currentSrc]);
   ```

**最佳实践**：

- 在处理任何持续时间较长的 UI 动画或过渡时，必须假设状态随时可能在中途改变。
- 始终实现健壮的 `cleanup` 函数。
- 对于高频触发的输入（如游戏中的点击），UI 响应逻辑必须能够优雅地处理中断。

---

## 更新日志

- 2026-01-11: 初次创建，记录 Git 操作问题和解决方案
- 2026-01-12: 新增“思维死循环”问题的排查与预防方案
- 2026-01-20: 新增“代码替换失败”、“编码乱码”及“大文件操作”经验总结
- 2026-01-22: 新增“多游戏存档冲突”以及“存档点击无效”问题的完整分析与解决方案
- 2026-01-22: 新增“背景图片切换错误”问题的完整分析与解决方案
