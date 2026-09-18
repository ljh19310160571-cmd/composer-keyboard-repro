# 输入框 / 软键盘 视口 bug 复现

一个真实项目（聊天类 PWA）里抽出来的最小复现，跟业务代码无关，不含任何密钥/接口/数据。

## 现象

安卓（vivo）上，把这个网页**添加到主屏幕**（standalone / PWA 模式）后，从桌面图标打开：

- 点输入框弹出软键盘时，输入框有时不跟着键盘一起弹上来，或者要卡好几下、点几次屏幕才归位；
- 同一个页面用**浏览器 tab** 打开完全正常，只有 standalone 模式才复现；
- 电脑 Chrome 的移动端模拟器也复现不出来（模拟不了真实软键盘），只能真机测。

## 怎么测

1. 手机浏览器（vivo 自带浏览器 / Chrome 都行）打开这个页面；
2. 菜单里选「添加到主屏幕」；
3. **关掉浏览器，从桌面图标重新打开**（一定要是这个入口，直接开 tab 复现不出来）；
4. 滚动聊天到底部，点输入框，看键盘弹出时输入框跟不跟得上、卡不卡。

页面顶部有个按钮条，可以切换我们试过的几种方案，方便对比：

- **A. visualViewport JS 撑高度**（当前项目线上在用的办法）：监听 `window.visualViewport` 的 `resize`/`scroll`，把它的 `height` 写成一个 CSS 变量 `--app-h`，容器高度绑定这个变量。问题：standalone 下键盘收起时这个事件有时不触发/延迟触发，容器高度卡在旧值，输入框被裁掉，要再点一下屏幕才刷新。
- **B. `interactive-widget=resizes-content`**：viewport meta 标签里加这个值，理论上是现代浏览器处理键盘遮挡的标准方案，键盘弹起时布局视口自动缩小。实测在这台 vivo 的 standalone 模式下**键盘弹出但输入框完全不跟随**，比 A 更糟，已回退。
- **C. composer 用 `position: fixed` 固定在页面底部**：不依赖 JS 算高度，直接把输入框钉在视口底部。这是还没试过、想请教的方案——朋友的建议。
- **D. 对照组**：什么都不做，只用 `100dvh`，用来看没有任何 hack 时是什么表现。

## 相关代码位置（供参考，这个仓库里没有完整项目）

原项目里：
- `App.tsx`：`sendMessage` 里发送后的重绘兜底逻辑；顶层 `useEffect` 里的 `visualViewport` 监听（方案 A）
- `index.html`：viewport meta 标签（方案 B 的开关在这里）
- `chat-interior.css` / `preview-migration.css`：`.composer` / `.composer-row` / `.app-shell` 的定位和高度

## 想请教的问题

1. 方案 C（fixed 定位）在 standalone + 软键盘场景下，是否真的比 visualViewport 更稳？有没有已知的坑（比如某些安卓机型 fixed 元素在键盘弹起时也会被顶飞/错位）？
2. 有没有更可靠的、专门针对 **standalone PWA + 安卓软键盘** 这个组合的最佳实践？
3. 是否需要结合 `ResizeObserver` 或别的兜底，还是纯 CSS `fixed` + `env(safe-area-inset-bottom)` 就够？

万分感谢！🙏
