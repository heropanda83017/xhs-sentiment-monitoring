# Cloudflare 浏览器渲染 — CDP 舆情采集备选方案

## 背景

当 Edge CDP 方案不可用时（如浏览器登录态失效、CDP 端口被占用），可用 Cloudflare Browser Rendering API 作为备选 JS 渲染方案。

## 核心工具（hermes-cloudflare 插件）

插件已安装至：`~/.hermes/profiles/land-of-dream-planning/plugins/hermes-cloudflare/`

**注册工具（2026-05-22 验证）：**
- `cf_markdown` — URL → Markdown（最适合舆情采集）
- `cf_screenshot` — URL → 截图
- `cf_json_extract` — 结构化数据提取
- `cf_crawl` / `cf_scrape` — 爬取
- `cf_links` — 提取页面链接
- `cf_content` — 原始内容

## 前置要求

1. Cloudflare API Token（`cfut_...`）已写入 `~/.hermes/.env`：`CLOUDFLARE_API_TOKEN`
2. Cloudflare Account ID 已写入：`CLOUDFLARE_ACCOUNT_ID`
3. `plugins/hermes-cloudflare` 已复制到 `~/.hermes/profiles/land-of-dream-planning/plugins/`
4. `config.yaml` 中 `plugins.enabled` 包含 `hermes-cloudflare`

## 典型调用

```
cf_markdown https://www.xiaohongshu.com/explore/xxx
```

返回 Markdown 格式页面内容，适合后续文本分析。

## 限制

- 免费版每天 5000 次请求
- 部分反爬站点仍会拦截
- 不保留登录态（匿名渲染），不适合需要登录才能访问的内容

## 调试插件加载问题

如插件未生效，开启调试：

```bash
# 设置环境变量后重启 gateway
HERMES_PLUGINS_DEBUG=1 hermes gateway start

# 查看 stderr 输出
hermes -z "回复OK" 2>&1 | grep -i "cloudflare\|plugin\|Loading"
```

常见跳过原因：`not in plugins.enabled` → 需在 `config.yaml` 添加 `plugins.enabled: [hermes-cloudflare]`
