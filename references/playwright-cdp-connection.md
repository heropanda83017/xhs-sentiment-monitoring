# Playwright Python CDP 连接 Edge — 验证可行（2026-05-22）

## 核心方法

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://localhost:9223")
    ctx = browser.contexts[0]
    page = ctx.pages[0]  # 已有标签页

    # 或创建新页
    page = browser.new_page()
    page.goto(url, wait_until="networkidle", timeout=30000)

    # 提取 Vue 响应式数据（__INITIAL_STATE__ 有循环引用，用 unwrap 脚本）
    unwrap_js = """
    function unwrap(obj, depth) {
        if (depth > 6 || obj === null || obj === undefined) return obj;
        if (typeof obj !== 'object') return obj;
        if ('_value' in obj && 'dep' in obj) return unwrap(obj._value, depth + 1);
        if ('value' in obj && 'dep' in obj) return unwrap(obj.value, depth + 1);
        if (Array.isArray(obj)) return obj.map(item => unwrap(item, depth + 1));
        const result = {};
        for (const key of Object.keys(obj)) {
            if (key === 'dep' || key.startsWith('__')) continue;
            try { result[key] = unwrap(obj[key], depth + 1); } catch(e) {}
        }
        return result;
    }
    """
    result = page.evaluate("""
    try {
        var state = window.__INITIAL_STATE__ || {};
        return JSON.stringify(unwrap(state, 0));
    } catch(e) { return JSON.stringify({"error": e.message}); }
    """, unwrap_js)

    data = json.loads(result)
    browser.close()
```

## 为什么不用 websocket-client

CDP 是**事件驱动协议**——服务器随时推送 `Page.loadEventFired`、`Network.requestWillBeSent` 等事件。websocket-client 的 `ws.recv()` 会收到所有消息，包括不想要的事件推送，导致：

- 发命令 → 等响应 → 收到事件推送 → 继续等 → 超时
- `id` 匹配失效

Playwright 内部维护命令队列，自动过滤事件，只返回目标命令的响应。

## Edge 启动参数

```
--remote-debugging-port=9223
--remote-allow-origins=*
--user-data-dir=C:\Users\Administrator\AppData\Local\Microsoft\Edge\User Data\DebugProfile
--no-first-run
```

**端口选 9223 而非 9222**：避免与已有 Edge 实例冲突（`netstat` 检查端口占用）。

## WebSocket 端口冲突（WinError 10048）

`websocket.create_connection()` 与 Edge CDP HTTP API 共享同一端口时会产生冲突。症状：

```
OSError: [WinError 10048] 通常每个套接字地址只允许使用一次
```

**解决**：换端口（9223+），或完全改用 Playwright Python `connect_over_cdp()`。

## headless Edge 的局限

`--headless=new` 模式：
- ✅ 不显示窗口
- ❌ 创建全新空 profile，**不继承用户登录态**
- ❌ Cookie SQLite DB 文件被 Edge 进程锁住，无法读取

**用途**：仅用于不需要登录的数据采集（如公开页面截图）。

## 验证记录

| 场景 | 结果 | 备注 |
|------|------|------|
| Edge 启动 + playwright 连接 | ✅ | Edge 已有 chromium，playwright 直连 |
| page.goto() 导航 | ✅ | wait_until="networkidle" 有效 |
| __INITIAL_STATE__ 提取 | ✅ | 需 unwrap 处理循环引用 |
| 单篇笔记（404） | ❌ | 笔记本身不存在，非反爬 |
