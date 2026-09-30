# FSC 官网（中国大学生方程式系列赛事）抓取备忘

来源：http://www.formulastudent.com.cn （中国汽车工程学会 SAE China 主办）
实测：2026-09-19 跑通全站资料查询 + 31 份官方文件批量下载

---

## 一、站点性质（先判断，省事）

- **静态 HTML、无需登录、无反爬** —— 和 RoboMaster 规则中心（Vue 空壳 + POST API）
  完全相反。这边 `urllib` 直取即可，不需要浏览器、不需要 API 逆向。
- 主办方是**学会**不是高校，但抓法同 `china-university-web-research` 的高校站。

## 二、URL 地图

| 栏目 | URL | 说明 |
|---|---|---|
| 首页 | `/` | 有实时报名院校数、赛事公告 |
| **资料查询** | `/documentation.html` | **官方文件总入口** |
| ├ 通用文档 | `?doc=1`（或裸 URL） | 规则/竞赛手册/技术检查表/案例分析 |
| ├ 油车 | `?doc=2` | FSCC 专项 |
| ├ 电车 | `?doc=3` | FSEC 专项 |
| ├ 无人车 | `?doc=4` | FSAC 专项 |
| └ 巴哈 | `?doc=5` | BSC 专项 |
| 列表页(no JS) | `/documentation.html?doc=N&page=M` | 分页 |
| **文件详情页** | `/news/<id>.html` | **附件链接在正文里** |
| 报名系统 | `ims.formulastudent.com.cn` | 注册才用得到 |
| 英文站 | `formulastudent.sae-china.org` | EN |
| 老站 | `old.formulastudent.com.cn` | 历史归档 |

**文件 CDN 规律**：`http://img.sae-china.org/web/<年>/<月>/<原文件名>.pdf`
→ 直接从详情页正则抓 `href="(https?://[^"]+\.(pdf|zip|docx?|xlsx?|rar|7z))"`

## 三、两阶段流程（重要 —— 别写成单体脚本）

**单体脚本 = 超时**。本次第一版把「爬清单 + 下载」写在一个 `--dl` 分支里，
重跑时又要重爬全站 44 条详情页，600s 直接超时，只下到 2 个文件。

```
阶段1  fsc_scrape.py  →  爬 5 个分类的所有分页 → 逐条进详情页取 CDN 链接
                       →  落盘 _fsc_docs.json（{cat,title,page,file}）
阶段2  fsc_dl.py      →  只读 JSON + ThreadPoolExecutor(5) 并行下
                       →  下载前先看本地是否已存在同尺寸文件（增量）
```

好处：清单可反复复用、断点续传、下载可单独重跑。

## 四、⚠️ 三个必踩的坑（都跟 Windows 编码有关）

### 坑1 · urllib 不会自动 percent-encode 中文/空格 URL

报错形如：
```
UnicodeEncodeError: 'ascii' codec can't encode characters in position 21-27
InvalidURL: URL can't contain control characters (found at least ' ')
```

**修复**（分段 quote，保留 path 的 `/` 和 query 的 `=`）：
```python
def enc(u):
    p = urllib.parse.urlsplit(u)
    return urllib.parse.urlunsplit((
        p.scheme, p.netloc,
        urllib.parse.quote(p.path,  safe="/%"),
        urllib.parse.quote(p.query, safe="=&%"), ""))
```
⚠️ 注意 `safe` 里**不能**留空格 —— 一开始写了 `safe="... "` 结果空格被当安全字符跳过，
仍然 `found at least ' '`。要么 `quote` 后 `.replace(" ", "%20")`，要么 safe 严格。

### 坑2 · Windows 终端 GBK，`print()` 中文可能崩

下载日志里带中文文件名时 `print` 抛 `UnicodeEncodeError`。
**修复**：`print(line.encode("ascii","replace").decode())` —— 日志可读性够用，
真实文件名以落盘为准，别为终端显示折腾。

### 坑3 · PowerShell 重定向 `>` 把 UTF-8 转 UTF-16

`python x.py > out.txt` 之后读回来是 `\u0000` 间隔的豆腐块，中文全毁。
**修复**：让 Python 自己写文件 `open(p,"w",encoding="utf-8").write(...)`，
**不要**经过 PS 重定向。

## 五、可复用骨架

```python
# 阶段1：爬清单
def doc_links(cat_id):          # 遍历分页直到无新条目
    seen, out, page = set(), [], 1
    while page <= 8:
        h = get(f"{BASE}/documentation.html?doc={cat_id}&page={page}")
        found = re.findall(r'href="(/news/\d+\.html)"[^>]*>(.*?)</a>', h, re.S)
        new = ...               # 去重计数，new==0 就 break
        page += 1; time.sleep(0.3)

def pdf_of(news_url):           # 详情页取 CDN 直链
    return re.findall(r'href="(https?://[^"]+\.(?:pdf|zip|xlsx?))"', get(news_url), re.I)[0]
```

```python
# 阶段2：并行增量下载
with ThreadPoolExecutor(5) as ex:
    for line in ex.map(fetch, todo): print(line.encode("ascii","replace").decode())
```

## 六、验证（别只看下载成功）

下载完必须**验魔数**，因为错误页也会返回 200：
```python
assert open(p,"rb").read(4) == b"%PDF"   # PDF
assert open(p,"rb").read(2) == b"PK"     # ZIP/docx/xlsx
```
本次 31 份全部通过，0 坏文件。

抽取正文本用 `pypdf`（`pip install pypdf`）；
⚠️ 封面页因字体嵌入会抽出乱码，**正文页正常**，别误判为文件损坏。

## 七、站点情报（2026 赛季，供引用）

- 报名院校数：**FSCC 139 / FSEC 102 / FSAC 39 / BSC 79**
- 联系方式：010-50911000；zhenghao@sae-china.org；
  北京市大兴区亦庄经济开发区融兴北三街 39 号行知楼
- 规则版本节奏：2026 规则"最终版"于 **2026-04-10** 发布（12月出答题版/答辩纲要）
