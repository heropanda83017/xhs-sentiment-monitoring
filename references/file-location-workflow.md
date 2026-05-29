# 文件定位与Web提取工作流

## 核心原则（2026-05-22）

> **用户问"我的文件在哪"，先查本地，再查网页。**

用户分享一个外部链接（如Kimi/Miro/飞书文档），并不意味着文件不在本地。他们很可能先收到了外部链接、下载到本地、然后才问你。查本地是零成本的第一步，比尝试web提取快得多。

---

## 本地文件快速定位路径（Windows）

按优先级顺序搜索：

```
1. 用户桌面
   C:\Users\Administrator\Desktop\

2. 用户文档（含微信文件缓存）
   C:\Users\Administrator\Documents\
   （含 xwechat_files/ 子目录——微信传输文件默认缓存位置）

3. WPS/企业微信/钉钉云盘缓存
   C:\Users\Administrator\WPS Cloud Files\
   C:\Users\Administrator\.wps-cloud-files\

4. 用户下载目录
   C:\Users\Administrator\Downloads\

5. 项目输出目录
   E:\Landofdream\输出\
   E:\Landofdream\源文件\
```

搜索关键词：`2026` + `方案`/`铺排`/`时尚`/`时装` 组合。

## 外部链接 → 内容提取决策树

```
用户分享URL
    │
    ├─ PDF/Word 链接 → 直接 urllib 下载（二进制模式）
    │
    ├─ Notion/飞书/腾讯文档 → 浏览器工具 open + 截取/导出
    │
    ├─ Kimi/Miro 等 SPA 页面（JS渲染）→ 见下方
    │
    └─ 普通网页 → web_extract 工具
```

## SPA页面（JS渲染型）内容提取

**典型特征**：页面HTML只有 `<div id="root">` + JS bundle引用，无预渲染内容。

**适用场景**：Kimi.link、Miro、Canva 等在线文档工具分享链接。

**提取方法**：

```python
import urllib.request, ssl, re

ctx = ssl._create_unverified_context()
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0 ...'})
resp = urllib.request.urlopen(req, context=ctx, timeout=15)
html = resp.read().decode('utf-8', errors='replace')

# 找JS文件引用
js_files = re.findall(r'src="([^"]+\.js)"', html)

# 逐个加载JS，提取字符串内容
for js in js_files:
    js_url = base_url + "/" + js if not js.startswith("http") else js
    js_req = urllib.request.Request(js_url, headers={'User-Agent': '...'})
    js_resp = urllib.request.urlopen(js_req, context=ctx, timeout=15)
    js_content = js_resp.read().decode('utf-8', errors='replace')
    
    # 提取有意义的中文文本块
    text_blocks = re.findall(r'["\']([^"\']{20,})["\']', js_content)
    meaningful = [t for t in text_blocks 
                  if any('\u4e00' <= c <= '\u9fff' for c in t) and len(t) > 30]
    # 同时检查模板字符串
    templates = re.findall(r'`([^`]{30,})`', js_content)
    meaningful += [t for t in templates 
                   if any('\u4e00' <= c <= '\u9fff' for c in t) and len(t) > 40]
```

**已知SPA来源**：
- Kimi分享链接（`.kimi.link`）→ JS bundle内含完整文档内容
- 腾讯文档导出页 → 可能是PDF直链，优先尝试下载

## 已知文件路径规律（理想之地项目）

用户处理过的文件有以下命名规律：

| 文件类型 | 典型命名 | 典型位置 |
|---------|---------|---------|
| 时尚季方案 | `2026美育江汉·理想之地时尚季全域策划方案V*.docx` | 微信文件缓存 |
| 时尚季铺排 | `2026武汉时尚季铺排v2.pdf` | Desktop / 微信缓存 |
| 竞品分析 | `E:\Landofdream\源文件\03-竞品分析\` | 固定目录 |
| 活动方案 | `E:\Landofdream\源文件\06营销\营销活动\` | 按年份分类 |

---

## 反面教材（本案）

用户问："我之前的时尚季文件在哪个路径下"

我的错误路径：
1. ❌ 先用 `web_extract` → 失败（后端未配置）
2. ❌ 再用 `terminal` + `curl` → 失败（WSL bash路径问题）
3. ❌ 最后才用 `execute_code` + `urllib` → 成功获取到内容

正确路径：
1. ✅ 先搜本地 Desktop + Documents（含微信缓存）→ **直接找到全部文件**
2. 再尝试网页内容提取

---

## 本次发现（2026-05-22）

时尚季文件实际在：
- Desktop: `2026武汉时尚季铺排v2.pdf`
- 微信缓存: `C:\Users\Administrator\Documents\xwechat_files\...\msg\file\2026-05\`
  - V1.0.docx（42KB）
  - V2.0_好友运营升级版.docx（47KB）
  - V3.0_KOL融合版.docx（48KB）

Kimi链接里的方案（"2026江汉之驿全域时尚季策划案"）是独立版本，比本地V1-V3更完整。
