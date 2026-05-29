---
name: xhs-sentiment-monitoring
description: 理想之地小红书舆情监控系统。小红书数据采集+情感分析+日报生成全流程。触发词：「小红书舆情」「xhs」「舆情监控」「小红书监控」。
version: 6
date: 2026-05-23
changelog: |
  v7 (2026-05-23): 新增Tab清理流程(CDP HTTP API关闭标签页)；新增"页面爆炸"已知坑；
                   定时任务升级为双关键词(理想之地+恒江雅筑)；多关键词监控文档化。
  v6 (2026-05-22): bb-browser 全链路稳定落地；report_generator.py 废弃，改用 inline 流程；
                   pip install snownlp/jieba 正确 timeout=300s；cronjob 已升级。
  v5: bb-browser 替换 Playwright CDP，搜索/详情/评论成功率大幅提升。
  早期版本: xhs Python API（已废弃，sign 过期）/ xhs-cli（废弃，与 bb-browser 重复）。
---

# 小红书舆情监控系统
> ⚠️ 更新说明（v6 — 2026-05-22）：bb-browser 全链路方案稳定，snownlp/jieba 已装，日报流程跑通。report_generator.py 仍在用废弃 xhs CLI，需改造，参考下方 inline 流程。

## 系统架构（四层）

```
Edge CDP 浏览器（bb-browser 调用，port 9223，Default profile）
       ↓
bb_xhs_collector.py（subprocess 调 bb-browser，搜索+详情+评论）
       ↓
sentiment 分析（正文 + 评论全文，SnowNLP）
       ↓
日报（reports/*.txt）
```

### 数据采集路径现状

| 路径 | 状态 | 说明 |
|------|------|------|
| **bb-browser + xhs adapter** | ✅ 首选 | 搜索+详情+评论全链路，官方维护 |
| **Exa MCP** | ✅ 即用 | 语义搜索，无需登录 |
| ~~xhs Python API~~ | ❌ 废弃 | sign 算法过期，code 300011 |
| ~~xhs-cli~~ | ❌ 废弃 | 底层 Playwright，与 bb-browser 重复 |

## bb-browser 接入方案

### 组件位置

| 组件 | 路径 |
|------|------|
| Node | `C:\Program Files\nodejs\node.exe` |
| bb-browser CLI | `C:\Users\Administrator\AppData\Roaming\npm\node_modules\bb-browser\dist\cli.js` |
| bb-sites adapter dir | `C:\Users\Administrator\.bb-browser\bb-sites\` |
| 采集脚本 | `E:\\Landofdream\\舆情监控\\bb_xhs_collector.py` ✅ 已创建（v3 — 多关键词支持） |

### 调用封装

```python
import subprocess, json

NODE = r'C:\Program Files\nodejs\node.exe'
BB_CLI = r'C:\Users\Administrator\AppData\Roaming\npm\node_modules\bb-browser\dist\cli.js'

def bb_call(args, timeout=90):
    r = subprocess.run(
        [NODE, BB_CLI, '--port', '9223', '--json'] + args,
        capture_output=True, text=True, timeout=timeout)
    return json.loads(r.stdout)
```

### 正确流程（全量评论采集）

```
1. subprocess.run([NODE, BB_CLI, '--port', '9223', 'open', search_url])
2. time.sleep(10)                                   # 等渲染
3. data = bb_call(['site', 'xiaohongshu/search', keyword])
4. 取 notes[n]['url'] + notes[n]['note_id']
5. subprocess.run([NODE, BB_CLI, '--port', '9223', 'open', note_url])
6. time.sleep(10)
7. detail = bb_call(['site', 'xiaohongshu/note', note_id])
8. comments = bb_call(['site', 'xiaohongshu/comments', note_id])
```

### 数据格式（⚠️ 字段类型注意）

**搜索结果 notes**：`likes` 是**字符串**（不是 int）！
```json
{"note_id": "...", "title": "...", "author": "...",
 "likes": "6",        ← 字符串！必须 int(v or 0)
 "url": "https://..."}
```

**笔记详情**：
```json
{"title": "...", "desc": "...", "author": {"nickname":"..."},
 "likes": 6, "comments_count": 3, "collects": 2, "tags": [...]}
```

**评论**：
```json
{"count": 5, "has_more": true, "cursor": "...",
 "comments": [{"user": {"nickname":"momo"}, "content": "...", "like_count": 2}]}
```

### 已知坑

| 问题 | 解法 |
|------|------|
| 搜索返回 `{"error": "Not logged in"}` | Edge CDP 窗口未登录，手动扫码/登录 |
| `note` 返回 `"Missing xsec token"` | 必须先 `open` 完整 URL（含 token），等10s再调 note |
| `note` 返回 `"Note detail not loaded"` | 笔记已删/私密，或 xsec_token 过期，换一条 |
| 搜索返回 0 条 | 先 `open` 搜索 URL，再等10s |
| `ValueError: Unknown format code 'd'` | `likes` 是字符串，`int(likes or 0)` 而非直接 `int(likes)` |
| **页面爆炸**：`bb-browser open` 每次调用都会打开**新标签页**，不重用现有标签页 | 每次采集后必须清理标签页，否则 Edge 内会堆积数十个 xiaohongshu 页面（实测可达124个） |
| `open about:blank` 不会关闭当前标签页 | 这只是打开空白页，原标签页依然存在。关闭需用 CDP HTTP API |

## 情感分析 + 日报生成（已验证可行流程）

> ⚠️ report_generator.py 废弃（调已废弃的 xhs CLI），直接用 SKILL.md 里的 inline 流程。
> pip install snownlp jieba 超时规律：60s/180s 均失败，**300s 才成功**。

```python
import json, os, glob, datetime
from snownlp import SnowNLP

monitor_dir = r'E:\Landofdream\舆情监控'
data_dir = os.path.join(monitor_dir, 'data')
reports_dir = os.path.join(monitor_dir, 'reports')
os.makedirs(reports_dir, exist_ok=True)

# 1. 读最新 xhs_full JSON
files = sorted(glob.glob(os.path.join(data_dir, 'xhs_full_*.json')),
               key=os.path.getmtime, reverse=True)
with open(files[0], encoding='utf-8') as f:
    notes = json.load(f)

# 2. 情感分析（正文 + 评论全文）
results = []
for note in notes:
    text = (note.get('content') or '')[:500]
    for c in note.get('comments', []):
        text += ' ' + (c.get('content') or '')
    if not text.strip():
        score, sentiment = 0.5, 'neutral'
    else:
        try:
            score = SnowNLP(text).sentiments
        except:
            score = 0.5
        sentiment = 'positive' if score >= 0.6 else 'negative' if score <= 0.4 else 'neutral'
    results.append({
        'note_id': note['note_id'],
        'title': note.get('title', ''),
        'likes': int(note.get('likes', 0) or 0),  # ⚠️ likes 是字符串
        'sentiment': sentiment,
        'score': round(score, 3),
        'content': (note.get('content') or '')[:80]
    })

# 3. 保存 sentiment JSON
ts = files[0].split('_')[-1].replace('.json', '')
with open(os.path.join(data_dir, f'sentiment_{ts}.json'), 'w', encoding='utf-8') as f:
    json.dump(results, f, ensure_ascii=False, indent=2)

# 4. 统计 + 生成报告
neg = [r for r in results if r['sentiment'] == 'negative']
pos = [r for r in results if r['sentiment'] == 'positive']
neu = [r for r in results if r['sentiment'] == 'neutral']
wsum = sum(r['score'] * r['likes'] for r in results)
wcount = sum(r['likes'] for r in results)
widx = wsum / wcount if wcount > 0 else 0.5

today = datetime.date.today().strftime('%Y-%m-%d')
ts2 = datetime.datetime.now().strftime('%H%M%S')
report = f"""小红书舆情日报 | 理想之地·武汉 | {today}
采集笔记: {len(results)}条 | 总点赞: {wcount}
正面:{len(pos)} 负面:{len(neg)} 中性:{len(neu)}
加权舆情指数: {widx:.3f}（{'偏正面 ↑' if widx>0.55 else '偏负面 ↓' if widx<0.45 else '中性 →'}）

■ 负面舆情
"""
for r in sorted(neg, key=lambda x: x['score']):
    report += f"  [{r['score']:.2f}] likes={r['likes']:4d} | {r['title'] or '(无标题)'}\n"
    report += f"       └ {r['content'][:75]}\n"

report += f"""
■ 正面舆情（高互动TOP3）
"""
for r in sorted(pos, key=lambda x: -x['likes'])[:3]:
    report += f"  [{r['score']:.2f}] likes={r['likes']:4d} | {r['title'] or '(无标题)'}\n"

out = os.path.join(reports_dir, f'舆情日报_{today.replace("-","")}_{ts2}.txt')
with open(out, 'w', encoding='utf-8') as f:
    f.write(report)
print(f"✅ {out}")
```

## Tab 清理流程（采集后必做）

⚠️ 多次采集后 Edge 内会堆积大量小红书标签页，**必须在每次采集完成后清理**。

### 方法1：CDP HTTP API（推荐，无需另外安装工具）

```python
import urllib.request, json

resp = urllib.request.urlopen("http://localhost:9223/json", timeout=5)
tabs = json.loads(resp.read())
closed = 0
for t in tabs:
    url = (t.get("url", "") or "")
    if "xiaohongshu.com" in url:
        tid = t.get("id", "")
        req = urllib.request.Request(f"http://localhost:9223/json/close/{tid}", method="GET")
        urllib.request.urlopen(req, timeout=3)
        closed += 1
print(f"已关闭 {closed} 个小程序标签页")
```

### 方法2：重启 Edge（粗暴但干净）

```python
import subprocess
subprocess.run(['taskkill', '/f', '/im', 'msedge.exe'], capture_output=True)
# 重新启动
edge_path = r'C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe'
subprocess.Popen([edge_path, '--remote-debugging-port=9223', '--remote-allow-origins=*',
                  '--new-window', 'https://www.xiaohongshu.com'])
```

### 优化建议：单标签页复用

对需要多次 open 的场景（如采集多条笔记详情+评论），**不要每条笔记都 open 一个新标签页**——改用 `cdp json/activate` 激活已有标签页后导航。但 bb-browser 当前不支持 tab 激活后的 URL 变更，建议的变通方案：

1. 先 open 搜索页 → 执行 site search 拿到笔记列表
2. 对每条笔记：用 bb-browser `open` 打开详情 → 采集 → **立即记下 tab_id**
3. 全部采集完成后，统一用 CDP HTTP API 关闭所有 xiaohongshu 标签页

bb_xhs_collector.py v3 支持通过 CLI 参数传入多个搜索关键词：

```bash
python bb_xhs_collector.py 恒江雅筑 金融街恒江雅筑 金融街 学区 一初学苑 维权
```

**改造要点：**
- 原 `SEARCH_KEYWORD = "理想之地"` 改为 `SEARCH_KEYWORDS` 列表
- 无参数时默认 `["理想之地"]`（向后兼容）
- 每个关键词依次执行 open search tab → 搜索 → 详情 → 评论 全流程
- 输出文件名含关键词标识（`xhs_full_{keyword}_{timestamp}.json`）

**典型应用场景：** 竞品/关联项目舆情监控。当同集团其他项目（如恒江雅筑）爆发危机时，通过关键词监控防止舆情泛化至本盘。

## 技术参考文档（references/）

| 文件 | 内容 |
|------|------|
| `bb-snownlp-pipeline.md` | bb-browser + SnowNLP 全链路验证记录，含 pip 超时规律、组件路径、已知坑 |
| `bb-browser-xhs-architecture.md` | bb-browser 架构设计（早期） |
| `cdp-scraper-notes.md` | CDP 爬虫踩坑记录 |
| `windows-pip-install.md` | pip 安装超时问题 |
| 其他 *.md | 各专项踩坑记录 |

## 目录结构

```
E:\Landofdream\舆情监控\
├── bb_xhs_collector.py        # ✅ 主力采集脚本（bb-browser subprocess）
├── sentiment_analyzer.py      # SnowNLP 规则引擎（供参考，独立运行可用）
├── report_generator.py         # ⚠️ 有缺陷，需用上方 inline 方式代替
├── data/
│   ├── xhs_full_YYYYMMDD_HHMMSS.json   # 原始数据
│   └── sentiment_YYMMDD_HHMMSS.json    # 情感分析结果
└── reports/
    └── 舆情日报_YYYYMMDD_HHMMSS.txt    # 日报
```

## 依赖安装

```python
# 中国网络 pip 超时规律：60s/180s 均失败，300s 成功
subprocess.run([sys.executable, '-m', 'pip', 'install', 'snownlp', 'jieba'],
               capture_output=True, timeout=300)
```

定时任务

**cronjob ID**: `8f6dd14afafa` | 每日 09:00 | 状态：🟢

**监控范围**：两组关键词并行
- 组A：`理想之地 武汉` — 主项目舆情
- 组B：`恒江雅筑` — 关联项目舆情（2026-05-23 新增，用于学区维稳事件监测）

**双写输出**：本地 `reports/` + Notion「Hermes报告中心/📊 小红书舆情日报」（追加模式，不覆盖历史）

**监控范围**：两组关键词并行采集
- 组A：`理想之地 武汉`（原监控，保持）
- 组B：`恒江雅筑`（2026-05-23 新增，应对学区维稳事件）

**前提**：Edge CDP 窗口（port 9223，Default profile）已启动 + 已登录小红书

**Edge 启动命令**（bat → `shell:startup`）：
```
taskkill /f /im msedge.exe 2>nul & timeout /t 2
start "" "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
  --remote-debugging-port=9223 --remote-allow-origins=*
  --new-window https://www.xiaohongshu.com
```

**cronjob 关键配置**：
- 采集后必须清理标签页（见上方 Tab 清理流程）
- 保存格式含关键词标识：`data/xhs_full_{keyword}_{日期}.json`
- 日报含跨组关联分析：恒江雅筑讨论是否关联到理想之地

## Notion 报告分发

每日舆情日报除了存本地 reports/*.txt，还可同步写入 Notion 实现手机端访问。

**实测可用流程** (2026-05-23):
- 目标页面：`📊 小红书舆情日报`（子页面ID: `369d2d6a-b18a-8190-9302-e78ac9134945`）
- 写入前清空所有 blocks → 用 callout/bullet/heading/divider 构建美观格式 → 分批写入（每批≤10个）
- 详细实现见 `school-district-crisis-response` skill 的 `references/notion-report-delivery.md`

**cronjob 集成要点**：
- Step 0: Edge CDP 前置检查
- Step 1: 双关键词采集（理想之地 + 恒江雅筑）
- Step 2: SnowNLP 情感分析
- Step 3: 写入本地 reports/*.txt
- Step 4: 写入 Notion 页面（使用 notion-client 清空+重写）

**注意事项**：
- Notion API 对批量写入有速率限制，每批不超过10个blocks
- 积分响应超时建议设置120s
- 页面ID必须是完整的36位UUID（search返回的完整值，不是截断显示）

## GitHub 发布

详见 `references/github-publish-workflow.md`。

**已发布仓库**（heropanda83017）：

| 技能 | 仓库 |
|------|------|
| xhs-sentiment-monitoring | github.com/heropanda83017/xhs-sentiment-monitoring |
| karpathy-llm-wiki | github.com/heropanda83017/karpathy-llm-wiki |

Token 需有 `repo` scope。建议 30-90 天有效期。发布时必须脱敏路径（详见 references/）。

## 数据分级

| 类别 | 级别 |
|------|------|
| 情感分析结论（正面/负面趋势） | 🟢公开 |
| Cookie/登录态 | 🔴机密 |
| 具体评论原文 | 🟡敏感，仅摘要引用 |
