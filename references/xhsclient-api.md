# XhsClient API 关键方法（舆情监控适用）

**包版本**：xhs 0.2.13  
**安装路径**：`E:\Landofdream\xhs_libs`
**状态**：❌ API 签名算法已过期（2026-05-22），**推荐改用 Edge CDP 浏览器采集**

## 初始化（含签名封装）

```python
import sys
sys.path.insert(0, r'E:\Landofdream\xhs_libs')

from xhs import XhsClient
from xhs.help import sign

# ⚠️ 必须：sign不接收web_session参数，需要wrapper
def sign_wrapper(uri, data=None, a1="", web_session=""):
    return sign(uri, data, a1=a1)

client = XhsClient(
    cookie=r"完整cookie字符串",
    sign=sign_wrapper,  # ⚠️ 必须传wrapper，否则NoneType报错
    timeout=10,
)
```

## ⚠️ 已知故障：签名算法过期

**症状**：所有API请求返回：
```json
{"code": 300011, "success": false, "msg": "当前账号存在异常，请切换账号后重试"}
```

- `get_self_info()` 返回 `{"code": -1, "success": false}`
- Cookie（a1/web_session/id_token）仍然有效，但 MD5 签名被后端废弃
- 即使传了正确的 sign wrapper，签名校验仍然失败

**影响范围**：全部 API 接口（搜索、笔记、评论、用户信息等）

## 当前有效方案：Edge CDP 浏览器采集

详见 `xhs-data-extraction` skill 的 CDP 章节。核心思路：

```python
# 1. 启动Edge调试模式 → 用户手动登录
# 2. 通过CDP Network.getAllCookies提取Cookie
# 3. 用Runtime.evaluate从页面DOM或__INITIAL_STATE__提取数据
# 4. 无需过签名验证，浏览器原生处理
```

## 未验证的备选：HTML方式

```python
# 可能不需要签名验证的接口
note = client.get_note_by_id_from_html("note_id_str")
# 但2026-05-22测试返回JSON解析错误，不确定是否需要更新
```

## 关键方法速查（供未来签名修复后使用）

### get_note_by_keyword — 关键词搜索笔记
```python
result = client.get_note_by_keyword(
    keyword="理想之地 武汉",
    page=1, page_size=20,
    sort="general",
    note_type=0,  # 0=全部, 1=图文, 2=视频
)
```

### get_note_all_comments — 获取全部评论（含子评论）
```python
comments = client.get_note_all_comments(
    note_id="note_id_str",
    crawl_interval=1,  # 请求间隔（秒）
)
```

### get_note_by_id — 获取单篇笔记详情
```python
note = client.get_note_by_id("note_id_str")
```

### 其他方法

| 方法 | 用途 |
|------|------|
| `get_note_comments(note_id, cursor)` | 分页获取评论（不含子评论） |
| `get_note_sub_comments(note_id, root_comment_id)` | 获取指定根评论的子评论 |
| `get_user_info(user_id)` | 获取用户信息 |
| `get_search_suggestion(keyword)` | 搜索建议/联想 |

## Cookie信息

- **格式**：直接从浏览器DevTools复制或CDP提取的完整cookie字符串
- **传入方式**：`XhsClient(cookie="...")`，无需单独配置文件
- **有效期**：通常数周
- **安全级别**：🔴机密级，禁止写入memory或输出文件
- **安全操作**：提取后立即关闭当前会话执行 `/new`
