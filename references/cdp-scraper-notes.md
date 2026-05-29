# Edge CDP 采集方案 — 实施笔记（2026-05-22）

## 验证结论

| 测试项 | 结果 | 备注 |
|--------|------|------|
| 搜索页数据提取 | ✅ 可行 | 20条笔记，含标题/作者/日期/点赞数 |
| 单篇笔记评论抓取 | ❌ 被拦截 | "当前笔记暂时无法浏览" + 广告屏蔽插件提示 |
| API签名调用（XhsClient） | ❌ 签名过期 | code 300011 "账号异常" |
| CDP fetch() 直接调用API | ❌ 签名过期 | 同300011 |

**核心限制**：小红书搜索页可爬，笔记详情页被CDP反爬拦截。

## CDP 连接要点

### Edge 启动参数（必须全量）
```
--remote-debugging-port=9222
--remote-allow-origins=*        # 否则WebSocket返回403
--new-window https://www.xiaohongshu.com
```

### CDP WebSocket 连接（Python）

```python
import http.client, json, websocket

conn = http.client.HTTPConnection("localhost", 9222, timeout=5)
conn.request("GET", "/json")
tabs = json.loads(conn.getresponse().read().decode())

# 找小红书标签页
xhs_tab = next(t for t in tabs if 'xiaohongshu' in t['url'].lower())
ws = websocket.create_connection(
    xhs_tab['webSocketDebuggerUrl'],
    timeout=10,
    origin="http://localhost:9222"  # ⚠️ 必须设置
)
```

### 常用CDP命令

```python
# 导航
{"id": 1, "method": "Page.navigate", "params": {"url": "..."}}

# 执行JS获取返回值
{"id": 2, "method": "Runtime.evaluate",
 "params": {"expression": "document.title", "returnByValue": True}}

# 获取Cookie
{"id": 3, "method": "Network.getAllCookies", "params": {}}

# 滚动页面加载更多
{"id": 4, "method": "Runtime.evaluate",
 "params": {"expression": "window.scrollTo(0, 700)"}}
```

### 数据提取方式（已验证可靠）

搜索页结构已知（class名称可能变，但 `<section>` + `<a[href*="/explore/"]>` 选择器有效）：

```javascript
// 提取笔记链接
document.querySelectorAll('a[href*="/explore/"]')

// 获取所属section内容（含标题/作者/日期/点赞）
a.closest('section') || a.parentElement;
```

数据解析（非结构化文本行）：
- 行1: 标题（最长、最有信息量）
- 行2: 作者（通常2-8字，不匹配日期/数字模式）
- 行3: 日期（YYYY-MM-DD / MM-DD / N天前 / N小时前）
- 行4: 点赞数（纯数字，≤6位）

## Windows 侧操作规范

**核心规则**：所有操作默认走 Windows 侧，用 execute_code + Python subprocess。

### 路径对照
| 场景 | 路径格式 | 示例 |
|------|---------|------|
| Python/open() | `E:\\...` | `r'E:\Landofdream\舆情监控'` |
| subprocess | `E:\\...` | 同上 |
| terminal (bash) | `/mnt/e/...` | 仅用于中文路径文件读取 |

### 安装Python包（Windows侧）
```python
import subprocess, sys
result = subprocess.run(
    [sys.executable, '-m', 'pip', 'install', '包名',
     '-t', r'E:\Landofdream\xhs_libs'],
    capture_output=True, text=True, timeout=120
)
```

### 写文件（Windows侧）
```python
with open(r'E:\Landofdream\舆情监控\file.py', 'w', encoding='utf-8') as f:
    f.write(content)
# write_file 工具因shell路径问题在Windows侧不可用
```

## CDP 浏览器生命周期管理

### 启动
1. taskkill 已有 Edge 进程
2. 以 debugging port 模式启动新 Edge
3. 等待5秒 + 提示用户手动登录小红书
4. 确认登录态（搜 Network.getAllCookies → check web_session exists）

### 关闭
- 用户可手动关闭该调试窗口
- 再次启动时需重复登录步骤
