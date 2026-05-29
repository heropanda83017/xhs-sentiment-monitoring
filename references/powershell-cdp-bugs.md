# Edge CDP Cookie提取方案对比

## 方案A：get_xhs_cookies.ps1（PowerShell） — ❌ 弃用

**状态**：不修复。Python CDP方案已验证可行，PS版本不再维护。

### 已知 Bug
1. **Bug 1**：WS连接目标错误 — 用 `$tabs[0]` 而非 `$xhsTab`
2. **Bug 2**：硬编码命令字符串，JSON响应解析不稳定
3. **Bug 3**：CDP端口发现不稳定，仅扫描5个端口

## 方案B：Python CDP直接提取 — ✅ 已验证可行（2026-05-22）

**推荐路径**，通过 `execute_code` 工具直接在 Windows Python 中运行。

### 关键步骤

#### 1. 重启Edge并开启调试端口
```python
subprocess.run(['taskkill', '/f', '/im', 'msedge.exe'], capture_output=True)
time.sleep(2)
subprocess.Popen([
    r'C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe',
    '--remote-debugging-port=9222',
    '--remote-allow-origins=*',  # ⚠️ 必须加，否则WS返回403
    '--new-window', 'https://www.xiaohongshu.com'
])
time.sleep(5)
```

> **注意**：用户需在此窗口手动登录小红书（CDP会话是独立的，不继承已有登录态）

#### 2. 通过CDP WebSocket提取Cookie
```python
import json, http.client, websocket, ssl

conn = http.client.HTTPConnection("localhost", 9222, timeout=5)
conn.request("GET", "/json")
tabs = json.loads(conn.getresponse().read().decode())

xhs_tab = next(t for t in tabs if 'xiaohongshu' in t['url'].lower())
ws = websocket.create_connection(xhs_tab['webSocketDebuggerUrl'], timeout=10, origin="http://localhost:9222")
ws.send(json.dumps({"id": 1, "method": "Network.getAllCookies", "params": {}}))
resp = json.loads(ws.recv())
ws.close()

# 过滤小红书域名
xhs_cookies = {}
for c in resp['result']['cookies']:
    if 'xiaohongshu' in c['domain'] or 'xhs' in c['domain']:
        xhs_cookies[c['name']] = c['value']
```

#### 3. 验证登录态
- ✅ 必须包含：`a1` + `web_session` + `id_token`
- ❌ 无 `web_session` → 用户未登录

#### 4. 安装依赖
```python
# websocket-client 需单独安装
subprocess.run([sys.executable, '-m', 'pip', 'install', 'websocket-client', '-t', r'E:\Landofdream\xhs_libs'])
```

## Cookie保存格式

```json
{
  "cookies_dict": {"a1": "...", "web_session": "...", ...},
  "cookie_string": "a1=...; web_session=...; ...",
  "source": "Edge CDP",
  "captured_at": "2026-05-22"
}
```

保存路径：`E:\Landofdream\xhs_libs\xhs_cookies.json`
纯字符串备份：`E:\Landofdream\舆情监控\cookie.txt`

## 安全提醒
Cookie信息为 🔴机密级，禁止写入memory或输出文件。用完即清理，执行 `/new` 隔离对话。
