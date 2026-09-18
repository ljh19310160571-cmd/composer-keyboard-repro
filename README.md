# 输入框 / 软键盘 视口 bug 复现

一个真实项目（聊天类 PWA「蛋窝」）里抽出来的最小复现，不含业务逻辑、不含任何密钥/接口/数据。

**这份 demo 里的布局 CSS 和视口 JS 是从真实项目源码里原样摘出来的**（方案 A = 线上实际在跑的代码，只删掉了背景图和无关装饰），不是凭空模拟的效果图——可以放心照着这份代码分析/改。

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

## 真实代码在哪（这个仓库只是摘录，不是完整项目）

`index.html` 里 `<style>` 和 `<script>` 各有一段用注释框起来的「摘录」，对应原项目：

- **CSS 结构**（原样摘自 `chat-interior.css` / `preview-migration.css`）：
  `html,body,#root{height:100%;overflow:hidden}` → `.app-shell{height:var(--app-h,100dvh)}` → `.chat-view{display:flex;flex-direction:column;height:100%}` → `.chat-stream{flex:1;overflow:auto}` + `.composer{position:relative;flex:0 0 auto}`（注意：**现在是 `relative` 不是 `fixed`**，composer 能停在底部完全靠 flex 列的 `flex:0 0 auto`）
- **JS 视口逻辑**（原样摘自 `App.tsx` 顶层 `useEffect`）：监听 `window.visualViewport` 的 `resize`/`scroll`，把 `vv.height` 写进 CSS 变量 `--app-h`，整个 `.app-shell` 的高度绑定这个变量——键盘弹起时理论上应该让 app-shell 跟着变矮，composer 被"挤"到键盘上方。

## 想请教的问题

1. 方案 C（fixed 定位）在 standalone + 软键盘场景下，是否真的比 visualViewport 更稳？有没有已知的坑（比如某些安卓机型 fixed 元素在键盘弹起时也会被顶飞/错位）？
2. 有没有更可靠的、专门针对 **standalone PWA + 安卓软键盘** 这个组合的最佳实践？
3. 是否需要结合 `ResizeObserver` 或别的兜底，还是纯 CSS `fixed` + `env(safe-area-inset-bottom)` 就够？

万分感谢！🙏
