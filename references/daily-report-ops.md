# 每日科技日报 · 运维要点（校园网直连时代）

> 脚本：`<本地资料库>`（调度）+ `daily_tech_report_v2.py`（生成）
> 调度：**Windows 任务计划程序**（不是 Hermes cron）——`Hermes每日科技日报` + `Hermes每日科技日报推送`，每天 18:30
> 外置知识：记忆外置库 `详细知识\daily_report.md`

## ⚠️ 代理策略（2026-09-13 改，旧版是坑）

**正确策略：有代理就用代理，没代理就直连。绝不硬编码死代理兜底。**

旧版 `daily_pipeline.ps1` 里有这么一段兜底：

```powershell
if (-not $env:HTTP_PROXY) {
    $env:HTTP_PROXY = "http://127.0.0.1:7892"   # ← 硬编码夜煞云端口，已删
    $env:HTTPS_PROXY = "http://127.0.0.1:7892"
}
```

关掉 VPN 后 7892 端口已死，但这段逻辑会把**本来能直连的源（HN/GitHub/翻译）也导向死代理**
→ 日报整体挂掉。已删除，改为尊重"无代理"状态走直连。

## 校园网（无 VPN）下各源实测（2026-09-13）

| 源 | 状态 | 说明 |
|------|------|------|
| Hacker News / HN-Algolia | ✅ 直连 | **日报主力** |
| GitHub API / Trending | ✅ 直连 | 主力 |
| 东方财富（财经） | ✅ 直连 | |
| bing 图片 / pexels（配图） | ✅ 直连 | |
| mymemory（翻译） | ✅ 直连 | |
| ❌ arxiv（学术） | ❌ 超时 | **有本地兜底 `SEMICONDUCTOR_DATA`，不影响出报** |
| ❌ V2EX / 36kr | ❌ 超时 | 非核心板块，跳过 |
| ❌ 新浪行情 | ⚠️ 403 | 需 Referer，部分可用 |

**结论：校园网无 VPN 下日报正常生成、质量无损**（实测对比 18:30 有 VPN 版 vs 23:16 无 VPN 版，
内容甚至更丰富，因为 HN/GitHub 才是主力源）。所有抓取函数都有 `try/except` + 8-25s 超时，
死源只是被跳过、不会中断整个生成。

## ❌ arxiv 别折腾了（试过的路全堵）

`export.arxiv.org` 直连不通；换域名 `arxiv.org/api/query` → **302 跳回 export.arxiv.org**，死循环；
ghproxy 转发 → **403（只放行 github 域名）**；gh-proxy / ghfast.top / jina / x-mol 全不通。
→ arxiv 板块靠脚本内 `SEMICONDUCTOR_DATA` 本地静态数据兜底（来源爱集微等国内站），
渲染成"前沿论文/存储芯片/应用芯片"等板块，**够用，不要再去接镜像**。

## PowerShell 坑（`-replace` 第2参数）

`daily_pipeline.ps1` 往 HTML 里插吉大通知那块曾写错：

```powershell
# ❌ 报 "The -ireplace operator allows only two elements to follow it, not 4"
$h = $h -replace '(<body[^>]*>)', '$1' + "`n" + $jluHtml

# ✅ 用双引号插值拼成一个替换串（$1 要转义成 `$1，否则被 PS 当自己的变量）
$h = $h -replace '(<body[^>]*>)', "`$1`n$jluHtml"
```

## 其他已知坑

- `print(emoji)` 在无控制台环境崩 → 跑前设 `$env:PYTHONUTF8="1"`
- `pipeline_log.txt` 是 **UTF-16**，读取要 `Get-Content -Encoding Unicode`
- 日报开头会前置**吉大每日通知**（`jlu_notices.py`，抓 OA `jldxList`，按学生事务关键词过滤
  + 排除学院党建/研究生噪声；`KWS`/`EXCLUDE` 可调）
