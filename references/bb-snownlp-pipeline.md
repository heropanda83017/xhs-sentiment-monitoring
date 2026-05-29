# bb-browser + SnowNLP 全链路验证记录

## 2026-05-22 验证成功

### 组件路径（已安装）
| 组件 | 路径 |
|------|------|
| Node.js | `C:\Program Files\nodejs\node.exe` |
| bb-browser CLI | `C:\Users\Administrator\AppData\Roaming\npm\node_modules\bb-browser\dist\cli.js` |
| bb-sites adapters | `C:\Users\Administrator\.bb-browser\bb-sites\` |
| 采集脚本 | `E:\Landofdream\舆情监控\bb_xhs_collector.py` |
| Python | `C:\Program Files\Python312\python.exe` |

### Edge CDP 要求
- **必须用 Default profile**（`AppData\Local\Microsoft\Edge\User Data\Default`），不是 DebugProfile
- Default profile 已登录小红书（nickname: "远光里走来的晴朗"）
- DebugProfile 是独立会话，需要重新登录
- CDP 端口：`9223`

### pip install snownlp/jieba 超时规律
| timeout | 结果 |
|---------|------|
| 60s | ❌ 超时 |
| 180s | ❌ 超时 |
| **300s** | ✅ 成功（28s 完成） |

中国网络访问 pypi.org 不稳定，**一律设 300s**。

### 情感分析关键代码
```python
from snownlp import SnowNLP

text = (note.get('content') or '')[:500]
for c in note.get('comments', []):
    text += ' ' + (c.get('content') or '')
score = SnowNLP(text).sentiments
sentiment = 'positive' if score >= 0.6 else 'negative' if score <= 0.4 else 'neutral'
```
- `likes` 字段是**字符串**，必须 `int(likes or 0)`，否则 format string 报错
- 加权舆情指数 = Σ(score × likes) / Σ(likes)

### bb-browser xhs 适配器已知坑
| 问题 | 解法 |
|------|------|
| 搜索返回 0 条 | 先 `bb-browser open <search_url>`，等待 10s，再调 search |
| note 返回 "Missing xsec token" | 必须先 open 完整 URL（含 token），等 10s，再调 note |
| note 返回 "Note detail not loaded" | 笔记已删除/私密，或 token 过期，换一条 |
| `ValueError: Unknown format code 'd'` | `int(likes or 0)` 而非直接 `int(likes)` |
| Edge CDP 未登录 | 重启 Edge Default profile，重新扫码登录 |
