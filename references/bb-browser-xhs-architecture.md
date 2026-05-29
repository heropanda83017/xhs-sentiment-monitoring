# bb-browser + xhs Adapter 技术细节（2026-05-23 实测）

## 安装状态

| 组件 | 路径 |
|------|------|
| bb-browser v0.11.6 | `C:\Users\Administrator\AppData\Roaming\npm\node_modules\bb-browser\` |
| bb-sites adapters | `C:\Users\Administrator\.bb-browser\bb-sites\` |
| xhs adapters | `bb-sites\xiaohongshu\{search,note,comments,feed,me,user_posts}.js` |

## 调用链路（实测可行路径）

### 完整四步流程（适用于 cronjob 改造）

```
bb open search_url
    ↓ (等待10s)
bb site xiaohongshu/search 关键词
    ↓ (取 notes[].url / notes[].note_id)
bb open <完整note_url（含xsec_token）>
    ↓ (等待10s)
bb site xiaohongshu/note <note_id>
bb site xiaohongshu/comments <note_id>
```

### Python subprocess 调用模板

```python
import subprocess, json, time

NODE = r'C:\Program Files\nodejs\node.exe'
BB   = r'C:\Users\Administrator\AppData\Roaming\npm\node_modules\bb-browser\dist\cli.js'
PORT = '9223'

def bb_raw(args):
    r = subprocess.run(
        [NODE, BB, '--port', PORT, '--json'] + args,
        capture_output=True, text=True, timeout=90
    )
    try:
        return json.loads(r.stdout)
    except:
        return {'_raw': r.stdout, '_stderr': r.stderr[:200]}

# 1. 搜索
r = bb_raw(['site', 'xiaohongshu/search', '理想之地 武汉'])
notes = r.get('notes', [])
print(f"搜索到 {len(notes)} 条，当前页关键词: {r.get('keyword')}")

# 2. 取高赞笔记
target = sorted(notes, key=lambda x: x.get('likes', 0), reverse=True)[0]

# 3. 打开笔记（必须用完整URL，不能只传note_id）
subprocess.run([NODE, BB, '--port', PORT, 'open', target['url']], timeout=60)
time.sleep(10)

# 4. 详情 + 评论
detail = bb_raw(['site', 'xiaohongshu/note', target['note_id']])
comments = bb_raw(['site', 'xiaohongshu/comments', target['note_id']])
```

## 搜索 adapter 实测输出

```json
{
  "keyword": "理想之地 武汉",
  "sort": "general",
  "sort_label": "综合",
  "count": 20,
  "has_more": true,
  "notes": [
    {
      "note_id": "69fc0e3c00000000230157a3",
      "xsec_token": "ABYP7zUkKBnvywDLtp...",
      "title": "汉口二环四代住宅‼️理想之地降价啦！捡漏",
      "type": "normal",
      "url": "https://www.xiaohongshu.com/explore/69fc0e3c...?xsec_token=...&xsec_source=",
      "author": "武汉二七滨江豪宅主理人",
      "author_id": "..."
    }
  ]
}
```

**注意**：`title` 字段有时为 `null`，正文在 `desc` 字段。需要fallback：
```python
title = n.get('title') or n.get('desc', '') or '(无标题)'
```

## 评论 adapter 输出格式

```json
{
  "count": 5,
  "has_more": false,
  "cursor": null,
  "comments": [
    {
      "id": "...",
      "user": {
        "nickname": "momo",
        "red_id": "xxx",
        "userid": "...",
        "url": "https://www.xiaohongshu.com/user/profile/..."
      },
      "content": "降价后性价比怎么样？",
      "like_count": 2,
      "sub_comments": [
        {
          "user": {"nickname": "博主", ...},
          "content": "回复内容",
          "like_count": 0
        }
      ]
    }
  ]
}
```

## Edge CDP 配置：Default profile ✅ vs DebugProfile ❌（2026-05-23 实测关键坑）

| profile | 小红书登录态 | bb-browser 能否用 | 推荐 |
|---------|------------|----------------|------|
| `--profile-directory=Default` | ✅ 有（"远光里走来的晴朗"） | ✅ 直接可用 | **✅ 必须用** |
| `--profile-directory=DebugProfile` | ❌ 无（空白会话） | ❌ 需手动注入 cookie | ❌ |

**结论**：用 Default profile（已有登录态），不需要 `--profile-directory` 参数，bb-browser 默认就是 Default。

**启动命令（已验证可行）**：
```
taskkill /f /im msedge.exe 2>nul & timeout /t 2
start "" "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
  --remote-debugging-port=9223
  --remote-allow-origins=*
  --new-window
  https://www.xiaohongshu.com
```
> 不加 `--profile-directory`，默认就是 Default profile，已有登录态。

## bb-browser xhs adapter 输出格式（实测，2026-05-23）

bb-browser 返回格式：`{"id": "xxx", "success": true/false, "data": {...}}`
**真实数据在 `data` 字段**，不是直接在根级别。常见错误：直接访问 `result['notes']` 而不是 `result['data']['notes']`。

| adapter | data 字段路径 | 特殊处理 |
|---------|-------------|---------|
| `xiaohongshu/search` | `data['notes']` | — |
| `xiaohongshu/note` | `data['title']/data['desc']/data['likes']` | `data['author']` 是字符串（nickname），非对象 |
| `xiaohongshu/comments` | `data['comments']` | — |

**author 字段类型转换**：
```python
# search adapter: author 是字符串
author_name = n['author']  # "👑墨墨幼乖👑" — 直接字符串

# note adapter: author 是嵌套对象
author_name = d['data']['author']['nickname']  # {'nickname': '...', ...}

# 兼容写法
def get_author(item, source='note'):
    if source == 'search':
        return item.get('author', '')
    else:  # note
        return item.get('author', {}).get('nickname', '') if isinstance(item.get('author'), dict) else str(item.get('author', ''))
```

**likes 字段类型**：`data['likes']` 是字符串（如 `"20"`），需要 `int()` 转换。`data['comments_count']` 可能是字符串或数字，防御性处理：
```python
likes = int(n.get('likes', 0) or 0)
comments_count = int(d['data'].get('comments_count', 0) or 0)
```

## 踩坑记录

### 1. Not logged in（最常见）
- **表现**：`bb site xiaohongshu/search` 返回 `{"error": "Not logged in", "hint": "Please log in..."}`
- **原因**：Edge CDP 9223 实例未登录小红书
- **解法**：在 Edge 窗口手动扫码/手机号登录
- **注**：这是无法自动化的硬限制，只能提醒用户每日维护

### 2. 搜索返回0条（不是 Not logged in）
- **表现**：exit=0 但 `notes: []`
- **原因**：当前页面不是小红书搜索结果页（可能还在首页/其他页面）
- **解法**：显式执行 `bb open https://www.xiaohongshu.com/search-result?keyword=...&source=web_search_notes` 再等待 10s

### 3. Note detail not loaded
- **表现**：`bb site xiaohongshu/note xxx` 返回 `"Note detail not loaded"`
- **原因**：笔记被删/设私密/xsec_token 过期/笔记不存在
- **解法**：换一条笔记；xsec_token 从搜索结果取，缓存 30 分钟内有效
- **注**：有时笔记真实存在但被反爬，刷新重试可能成功

### 4. Missing xsec token for note
- **表现**：`bb site xiaohongshu/note xxx` 返回 `"Missing xsec token for note"`
- **原因**：直接用 note_id 打开页面，xsec_token 未在内存中缓存
- **解法**：`open <完整URL（含xsec_token参数）>` → 等待10s → 再调 note
- **注**：search adapter 返回的 `url` 字段已含 `xsec_token`，必须用完整 URL

### 5. 页面 title 是"你的生活指南"
- **表现**：`bb get title` 返回 `小红书 - 你的生活指南`（不是搜索页）
- **原因**：导航命令发出后页面未完全加载就执行了下一条命令
- **解法**：`open` 后等待 8-12 秒再调用 adapter

## cronjob 当前架构（已实现，2026-05-22）

```
cronjob 8f6dd14afafa（每日09:00）
  ① bb_xhs_collector.py    # subprocess 调 bb-browser（已创建 v2）
     ├─ 搜索 → 取 top10 高赞笔记
     ├─ 逐条 open + note + comments
     └─ 输出 data/notes_with_comments_<date>.json
  ② sentiment_analyzer.py  # 分析标题+正文+评论全量文本
  ③ report_generator.py    # 生成报告
```

**⚠️ 前提**：Edge 9223 已启动 + 已手动登录小红书（Default profile 有登录态）。
