# App 内置浏览器实现方案

在一个已有的 iOS 17+ SwiftUI app 里，用 WKWebView 构建一个功能完整的隐私加固浏览器。浏览器在 "Playground" 板块内，与一组网页作品列表并列。app 使用自定义 tab bar（不是系统 TabView），通过 `.safeAreaInset(edge: .bottom)` 实现。

---

## 1. 震动反馈桥接（JS → 原生）

让网页能通过 JS bridge 触发原生震动反馈。

**Swift 端：**
- 在 `WKWebViewConfiguration` 的 `userContentController` 上注册一个名为 `"haptic"` 的 `WKScriptMessageHandler`。
- handler 的 `didReceive` 读取 `message.body` 为 String（`light` / `medium` / `heavy` / `rigid` / `soft` 之一），触发对应的 `UIImpactFeedbackGenerator(style:).impactOccurred()`。

**网页端（给页面作者用）：**
```js
function haptic(style = 'medium') {
  if (window.webkit?.messageHandlers?.haptic) {
    window.webkit.messageHandlers.haptic.postMessage(style);
  } else if (navigator.vibrate) {
    navigator.vibrate(style === 'light' ? 5 : style === 'heavy' ? 20 : 10);
  }
}
```

---

## 2. 浏览器入口

在 Playground 列表顶部加一行 **Browser**（地球图标 + "Browser" 标题 + "open any url" 副标题）。点击 push 一个 `BrowserView` 进 NavigationStack（不用 fullScreenCover），这样 app 底部 tab bar 保持可见。

---

## 3. 浏览器布局（两行）

**第一行 — 导航栏：**
- 左侧：`‹ PLAYGROUND` 返回按钮（dismiss）
- 右侧：`‹` 后退、`›` 前进、`</>` 脚本按钮、`⚙` 设置按钮
- 所有按钮常驻显示（没加载页面时禁用态 ink4 色，有页面时 ink1 色）
- 每个按钮 36×36pt frame + contentShape 扩大点击区域

**第二行 — URL 栏：**
- `TextField`，mono 字体，URL 键盘，关闭自动大写和自动纠错
- 右侧：刷新 `↻` 按钮（加载中显示 ProgressView spinner）
- 回车提交导航；没写 scheme 自动加 `https://`

**下方：** 细线分隔符，然后是 WKWebView。

---

## 4. WKWebView 封装（BrowserWebView）

一个 `UIViewRepresentable`，绑定：`url: URL`、`loading: Bool`、`bottomInset: CGFloat`、`canGoBack: Bool`、`canGoForward: Bool`、`urlText: String`，加一个 `BrowserState` 对象（持有 WKWebView 的 weak 引用，让外部能调 `goBack/goForward/reload`）。

**配置：**
```swift
let config = WKWebViewConfiguration()
config.allowsInlineMediaPlayback = true
config.websiteDataStore = .default() // 持久存储——登录态保留
config.userContentController.add(coordinator, name: "haptic")
// 反指纹脚本在 .atDocumentStart 注入（见 §8）

let wv = WKWebView(frame: .zero, configuration: config)
wv.customUserAgent = "..." // 见 §8
wv.allowsLinkPreview = false // 阻止 Universal Links
wv.scrollView.contentInsetAdjustmentBehavior = .never
wv.scrollView.keyboardDismissMode = .onDrag
wv.navigationDelegate = coordinator
wv.uiDelegate = coordinator
```

**Coordinator** 遵循 `WKNavigationDelegate`、`WKScriptMessageHandler`、`WKDownloadDelegate`、`WKUIDelegate`。

---

## 5. Tab Bar 底部留白

app 的自定义 tab bar 约 56pt 高，overlay 在内容上方。push 进去的子页面不知道它的存在，所以给 BrowserWebView 加 `.padding(.bottom, 56)`。键盘弹出时改为 0（监听 `UIResponder.keyboardWillShowNotification` / `keyboardWillHideNotification`）。

---

## 6. Navigation Delegate 行为

**Universal Links / URL Scheme 拦截：**
```swift
func webView(_:decidePolicyFor navigationAction:preferences:) async -> (WKNavigationActionPolicy, WKWebpagePreferences) {
    // 取消非 http scheme（bilibili://、twitter:// 等）
    // target=_blank 的链接在当前 webview 内加载，不弹新窗口
}
```

**下载处理：**
```swift
func webView(_:decidePolicyFor navigationResponse:) async -> WKNavigationResponsePolicy {
    // canShowMIMEType 为 false 或 Content-Disposition: attachment 时返回 .download
}
func webView(_:navigationResponse:didBecome download:) { download.delegate = self }
func webView(_:navigationAction:didBecome download:) { download.delegate = self }

// WKDownloadDelegate
func download(_:decideDestinationUsing:suggestedFilename:) async -> URL? {
    // 保存到用户配置的下载目录（见 §11）
    // 同名文件自动加序号避免覆盖
}
func downloadDidFinish(_:) { // 震动反馈 }
```

**状态更新：**
- `didCommit`：设 loading = true，更新 canGoBack/canGoForward/urlText
- `didFinish`：设 loading = false，写入浏览历史（跳过 about:blank），更新状态
- `didFail` / `didFailProvisionalNavigation`：设 loading = false

---

## 7. WKUIDelegate（JS 弹窗）

Coordinator 需要额外遵循 `WKUIDelegate`，并在创建 WKWebView 时设置 `wv.uiDelegate = context.coordinator`。

三个方法都用 `UIAlertController`，present 在 `webView.window?.rootViewController` 上：

```swift
// alert()
func webView(_ webView: WKWebView, runJavaScriptAlertPanelWithMessage message: String, initiatedByFrame frame: WKFrameInfo) async {
    await withCheckedContinuation { (cont: CheckedContinuation<Void, Never>) in
        let ac = UIAlertController(title: nil, message: message, preferredStyle: .alert)
        ac.addAction(UIAlertAction(title: "OK", style: .default) { _ in cont.resume() })
        webView.window?.rootViewController?.present(ac, animated: true)
    }
}

// confirm()
func webView(_ webView: WKWebView, runJavaScriptConfirmPanelWithMessage message: String, initiatedByFrame frame: WKFrameInfo) async -> Bool {
    await withCheckedContinuation { cont in
        let ac = UIAlertController(title: nil, message: message, preferredStyle: .alert)
        ac.addAction(UIAlertAction(title: "Cancel", style: .cancel) { _ in cont.resume(returning: false) })
        ac.addAction(UIAlertAction(title: "OK", style: .default) { _ in cont.resume(returning: true) })
        webView.window?.rootViewController?.present(ac, animated: true)
    }
}

// prompt()
func webView(_ webView: WKWebView, runJavaScriptTextInputPanelWithPrompt prompt: String, defaultText: String?, initiatedByFrame frame: WKFrameInfo) async -> String? {
    await withCheckedContinuation { cont in
        let ac = UIAlertController(title: nil, message: prompt, preferredStyle: .alert)
        ac.addTextField { $0.text = defaultText }
        ac.addAction(UIAlertAction(title: "Cancel", style: .cancel) { _ in cont.resume(returning: nil) })
        ac.addAction(UIAlertAction(title: "OK", style: .default) { _ in cont.resume(returning: ac.textFields?.first?.text) })
        webView.window?.rootViewController?.present(ac, animated: true)
    }
}
```

---

## 8. 反指纹加固

**User-Agent 伪装：**
- 移动端模式（默认）：iPhone Safari 18 UA
- 桌面模式：Mac Safari 18 UA
- 在设置里切换（见 §10），切换后自动 reload

**注入的 JS（WKUserScript，在 .atDocumentStart，forMainFrameOnly: false）：**
```js
// WebRTC — 替换 RTCPeerConnection 构造函数，调用即抛异常
// Canvas — hook toDataURL，读取前给像素数据加 1-bit 确定性噪声
// WebGL — hook getParameter，查询 UNMASKED_VENDOR/RENDERER 时返回 "Generic GPU"
// 硬件 — 伪装 hardwareConcurrency(6)、deviceMemory(4)、platform("iPhone")、vendor("Apple Computer, Inc.")
// 语言 — navigator.language="en-US"、navigator.languages=["en-US","en"]
// 屏幕 — width=393、height=852、colorDepth=32、devicePixelRatio=3（iPhone 15 尺寸）
// AudioContext — hook OfflineAudioContext.startRendering，在输出音频采样里加微小扰动
```

---

## 9. 脚本面板（Bookmarklets）

从 `</>` 按钮弹出的下拉浮层，右上角对齐，220pt 宽，带阴影。

**内容（从上到下）：**
1. **View Source** 按钮 — 执行 `document.documentElement.outerHTML`（加 iframe 内容），在全屏 sheet 中用 mono 字体深色底显示，有复制按钮
2. **SCRIPTS** 标签 + `+` 新建按钮
3. 已保存脚本列表，每行：名称（display 字体）+ ✏️ 编辑 + ▶ 运行
4. 运行：执行 `wv.evaluateJavaScript(script.code)`，触发 `.sensoryFeedback(.success)`
5. 编辑：弹 sheet，内含名称 TextField + 代码 TextEditor + 删除按钮

**ScriptStore：** `ObservableObject`，`[UserScript]` 存在 UserDefaults key `"browser_scripts"` 里。提供按 id 的 `nameBinding` / `codeBinding`。

---

## 10. 设置面板

从 `⚙` 按钮弹出，样式同脚本面板，240pt 宽。

**菜单项：**
1. **History（历史记录）** — 弹半屏 sheet，按时间列表（标题 + URL + 时间），点击导航，CLEAR 按钮
2. **Desktop/Mobile Mode（桌面/移动模式）** — 切换 UA 并 reload（显示 🖥/📱 图标，桌面模式时右侧 ON 标签）
3. **Cookies** — 弹半屏 sheet，按域名分组，展开可看每条 cookie 的名称/值，可单条删除或按域名删除
4. **Downloads（下载位置）** — 显示当前文件夹名，点击弹 `fileImporter` 选新目录（通过 security-scoped bookmark 持久保存权限）
5. **Clear All Data（清除全部数据）** — 确认弹窗，然后 `WKWebsiteDataStore.default().removeData(ofTypes: allWebsiteDataTypes())`
6. **PROTECTION（防护检测）** — 可折叠，显示 WebRTC/Geo/IP/Canvas 状态，CHECK 按钮运行检测 JS + ipify.org 查 IP
7. **SPOOFED AS（伪装身份）** — 可折叠，静态显示注入的身份信息（UA、Platform、Screen、Lang、CPU、GPU）

**泄漏检测 JS（通过 evaluateJavaScript 运行）：**
```js
// WebRTC：尝试 new RTCPeerConnection — 被拦截 = blocked
// Geo：navigator.permissions.query({name:'geolocation'}) — denied = blocked
// Canvas：画测试图案，toDataURL，用类 FNV 循环 hash，返回 hex
// 同时读回 navigator.userAgent、platform、screen 尺寸、language 确认伪装生效
```

**IP：** `URLSession.shared.data(from: "https://api.ipify.org?format=json")`

---

## 11. 下载位置管理

**DownloadSettings：** `ObservableObject` 单例。在 UserDefaults 里存 security-scoped bookmark。`resolvedURL` 解析 bookmark（失败则回退到 app 的 Documents/Downloads）。`setFolder(_:)` 由 fileImporter 结果调用。`folderName` 在设置里显示。

---

## 12. 浏览历史

**BrowserHistory：** `ObservableObject` 单例。`[HistoryEntry]` 存在 UserDefaults key `"browser_history"`，最多 200 条。`add(url:title:)` 跳过连续重复和 `about:blank`，同时清理已有的 about:blank 条目。

**HistoryEntry：** `Codable, Identifiable`，字段：`id`、`url`、`title`、`date`。

**Sheet：** 条目列表，每条显示标题（无标题则显示 URL）、URL（mono 字体）、日期。点击导航并关闭。CLEAR 按钮 + × 关闭。

---

## 13. Cookie 管理器

**CookieManager：** `ObservableObject` 单例。`load()` 从 `WKWebsiteDataStore.default().httpCookieStore.allCookies()` 读取所有 cookie，按域名分组。`deleteCookie(_:)` 和 `deleteCookies(for domain:)` 支持精细控制。

**Sheet：** 域名列表带数量，展开可看每条 cookie（名称 + 截断值 + × 删除）。展开底部有 "DELETE ALL FOR 域名" 按钮。

---

## 14. 初始状态

浏览器打开时 `currentURL = URL(string: "about:blank")`，WKWebView 立即存在（泄漏检测和脚本执行需要它）。上面叠一个 "地球图标 + enter a url to browse" 提示，用 `.allowsHitTesting(false)`，输入真实 URL 后自动隐藏。

---

## 关键实现注意事项

- 所有下拉浮层（脚本、设置）用 overlay + 透明背景点击关闭层，对齐 `.topTrailing`
- 浏览器通过 `navigationDestination` push（不是 fullScreenCover），保持 tab bar 可见
- `about:blank` 页面让 CHECK 可以不加载真实页面就立即运行
- 桌面模式 UA：`Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.0 Safari/605.1.15`
- 移动模式 UA：`Mozilla/5.0 (iPhone; CPU iPhone OS 18_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.0 Mobile/15E148 Safari/604.1`
- 指纹伪装值对应一台通用 iPhone 15（393×852 @3x，6 核，4GB 内存）
- 反指纹 JS 只注入 BrowserWebView，不注入 playground piece 的 WebView（那些是你自己的页面）
