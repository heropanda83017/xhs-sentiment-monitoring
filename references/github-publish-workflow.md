# GitHub 发布 workflow

本文档记录将 Hermes skill 发布到 GitHub 的完整流程。

## 发布内容清单

| 类型 | 是否发布 |
|------|---------|
| SKILL.md | ✅ |
| references/*.md | ✅ |
| 核心脚本（如 bb_xhs_collector.py） | ✅（路径脱敏后） |
| 数据文件、凭据 | ❌ |
| sentiment_analyzer.py、report_generator.py（有缺陷） | ❌ |

## 路径脱敏规则

上传前将以下路径替换为占位符：

| 原始路径 | 替换为 |
|---------|--------|
| `E:\Landofdream\舆情监控` | `/path/to/monitoring` |
| `E:\Landofdream\wiki-land-of-dream-planning` | `/path/to/wiki` |
| `C:\Users\Administrator\...` | `/path/to/...` |
| `C:\Program Files\nodejs\node.exe` | `/path/to/node.exe` |

## Python urllib 方式（推荐，gh CLI 不可用时）

```python
import urllib.request, urllib.error, json, base64

TOKEN = "ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
headers = {
    "Authorization": "Bearer " + TOKEN,
    "Accept": "application/vnd.github.v3+json",
    "X-GitHub-Api-Version": "2022-11-28",
    "Content-Type": "application/json"
}

# 1. 验证 token
req = urllib.request.Request("https://api.github.com/user", headers=headers)
with urllib.request.urlopen(req) as resp:
    user = json.loads(resp.read())
username = user["login"]
print(f"OK: {username}")

# 2. 创建 repo（private=False → 公开）
data = json.dumps({
    "name": "repo-name",
    "description": "description",
    "private": False,
    "auto_init": True
}).encode()
req2 = urllib.request.Request(
    "https://api.github.com/user/repos",
    data=data, headers=headers, method="POST"
)
try:
    with urllib.request.urlopen(req2) as resp:
        result = json.loads(resp.read())
    print(result["html_url"])
except urllib.error.HTTPError as e:
    body = json.loads(e.read())
    if "already exists" in body.get("message", ""):
        print("REPO_EXISTS")

# 3. 获取 main 分支 SHA
main_sha = None
for branch in ["main", "master"]:
    req3 = urllib.request.Request(
        f"https://api.github.com/repos/{username}/repo-name/git/ref/heads/{branch}",
        headers=headers)
    try:
        with urllib.request.urlopen(req3) as resp:
            main_sha = json.loads(resp.read())["object"]["sha"]
        break
    except urllib.error.HTTPError:
        continue

# 4. 创建 blob
def create_blob(content):
    enc = base64.b64encode(content.encode("utf-8")).decode()
    bd = json.dumps({"content": enc, "encoding": "base64"}).encode()
    req = urllib.request.Request(
        f"https://api.github.com/repos/{username}/repo-name/git/blobs",
        data=bd, headers=headers, method="POST")
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())["sha"]

# 5. 构建 tree → commit → update ref
tree_items = [{"path": "SKILL.md", "mode": "100644", "type": "blob",
               "sha": create_blob(skmd_content)}]
tree_data = json.dumps({"tree": tree_items, "base_tree": main_sha}).encode()
req_tree = urllib.request.Request(
    f"https://api.github.com/repos/{username}/repo-name/git/trees",
    data=tree_data, headers=headers, method="POST")
tree_sha = json.loads(urllib.request.urlopen(req_tree).read())["sha"]

commit_data = json.dumps({
    "message": "feat: initial upload",
    "tree": tree_sha,
    "parents": [main_sha] if main_sha else []
}).encode()
commit_sha = json.loads(urllib.request.urlopen(
    urllib.request.Request(
        f"https://api.github.com/repos/{username}/repo-name/git/commits",
        data=commit_data, headers=headers, method="POST"
    )).read())["sha"]

# 更新分支
upd = json.dumps({"ref": "refs/heads/main", "sha": commit_sha}).encode()
req_up = urllib.request.Request(
    f"https://api.github.com/repos/{username}/repo-name/git/refs",
    data=upd, headers=headers, method="POST")
with urllib.request.urlopen(req_up) as resp:
    json.loads(resp.read())

print(f"https://github.com/{username}/repo-name")
```

## 已发布的仓库

| 技能 | 仓库 | 可见性 |
|------|------|--------|
| xhs-sentiment-monitoring | github.com/heropanda83017/xhs-sentiment-monitoring | Public |
| karpathy-llm-wiki | github.com/heropanda83017/karpathy-llm-wiki | Public |

## Token 注意事项

- Token 格式：`ghp_` + 36位字符
- 复制时容易漏字符，务必验证（第一次 401 → 检查字符是否完全一致）
- 需要 `repo` scope 才能创建仓库
- Token 有效期：建议 30-90 天

## 改公开（private → public）

```python
url = f"https://api.github.com/repos/{username}/{repo}"
data = json.dumps({"private": False}).encode()
req = urllib.request.Request(url, data=data, headers=headers, method="PATCH")
with urllib.request.urlopen(req) as resp:
    result = json.loads(resp.read())
# result["private"] == False 即成功
```

## gh CLI 为什么不适用

`gh` CLI 在 Windows subprocess 中无法通过参数传入 token 认证（`gh auth status` 总报 "not logged in"）。Python urllib 直接调 API 是更可靠的方案。

## 安装已发布 skill 到新环境

```bash
git clone https://github.com/heropanda83017/xhs-sentiment-monitoring.git \
  ~/.hermes/skills/xhs-sentiment-monitoring
```
