# Phase 0 调研手册 —— research-notes.md 模板与采集配方

交付物是一份 `research-notes.md`。模板如下，逐节填写；每一行结论都必须
能指出证据来源（URL / 存档快照 / CDP 会话）。

```markdown
## <站点> 调研笔记（日期）
### 1 镜像与入口
| 域名 | 状态 | 证据 |
|---|---|---|
| a.com | ✓ 主站 | curl 200 + 卡片解析 |
| b.net | ✗ 停放域，302 → 广告 | curl -w %{redirect_url} |
### 2 列表文法
首页=/ 分类=/category/<slug> 搜索=/?s= 分页=/page/N
卡片：div.post > a[href=*-{id}.html] title属性=标题
      img data-original(带 !320x216 后缀需剥离) div.meta=日期
### 3 详情语法
标题=.archive-title h1 封面=.post-meta img.img 演员=a[href*=/cast/]
标签=.tags a[rel=tag] 播放源=a.btn-server[data-link] (名称→倒序token)
### 4 播放链路
token → https://resolver/c=<token[::-1]> → 打包脚本/明文m3u8/streamtape嵌入
各源网络绑定实测：[表]
### 5 多语言
/zh 前缀全路由可用；方括号一并翻译 → 词表双语
### 6 网络绑定四格实测
| 出口 | master | segment |
| 本机(签发方) | 200 | 200 |
| jina | ? | ? |
| allorigins | ? | ? |
| codetabs | ? | ? |
```

## 采集配方

**参考实现检索词**：`<site> downloader github`、`<site> scraper python`、
`<site> m3u8 extract`。重点看：镜像常量、解析函数、他们注释里写的坑。

**Wayback**：
```
http://archive.org/wayback/available?url=<site>/<path>&timestamp=2026
http://web.archive.org/cdx/search/cdx?url=<site>*&output=json&filter=statuscode:200&collapse=urlkey&limit=40
```
存档 HTML 直接当离线 fixture 与选择器验证材料。

**CDP 活体解剖**（CF 站点唯一途径；浏览器容器需已通过该站验证或可自动点击）：
```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    b = p.chromium.connect_over_cdp("http://127.0.0.1:9222", timeout=8000)
    pg = next((x for x in b.contexts[0].pages if "<site>" in x.url), None) \
         or b.contexts[0].new_page()
    pg.set_default_timeout(20000)
    pg.goto("https://<site>/<path>", wait_until="domcontentloaded")
    pg.wait_for_timeout(1000)
    html = pg.content()          # 结构分析 / fixture 采样
```

**原图尺寸采样**（决定 style.ratio）：取列表页 3-5 个不同缩略图 URL，
`curl -s | python -c "import sys,PIL.Image as I;print(I.open(sys.stdin.buffer).size)"`，
取众数宽高比保留两位小数。

**网络绑定四格实测**（决定播放模式建议——上线前必做一次）：

```bash
URL=<新解析出的播放地址>; ENC=$(python -c "import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1],safe=''))" "$URL")
curl -sS -o /dev/null -w "本机:   %{http_code}\n" "$URL"
curl -sS -m 45 -o /dev/null -w "jina:   %{http_code}\n" "https://r.jina.ai/$URL"
curl -sS -m 45 -o /dev/null -w "origins: %{http_code}\n" "https://api.allorigins.win/raw?url=$ENC"
curl -sS -m 45 -o /dev/null -w "codetabs: %{http_code}\n" "https://api.codetabs.com/v1/proxy?quest=$ENC"
```

判读：第三方全部非 2xx 且本机 200 → **绑定解析方网络**，该站跨网络必须 relay
（README 与 .env 注释都要写明）；第三方也 200 → direct 可用。
注意三个服务自身偶发 522/超时，需重测确认而非单次定论。

**镜像健康检查**：每个候选镜像走一次完整列表请求，
`curl -sS -m 20 -L -w "%{http_code} %{url_effective}\n"`——
最终 URL 跳到无关域（购物/短链）即停放域，剔除且不要进浏览器可达列表。
