---
name: jabletv-proxy-new-site
description: 在 jabletv-proxy 工程中开发新站点适配的标准流程。当需要为该工程新增上游站点（Spider / Adapter / 注册装配 / 测试 / 端到端验证）时使用，覆盖上游调研方法、设计决策表、已知陷阱与验收清单。
---

# jabletv-proxy 新站点开发

为 `jabletv-proxy`（FongMi TVBox 多站点 CMS 代理）新增一个上游站点适配。最新范例是
`sites/supjav/`——遇到取舍问题时优先参照它。深入材料见 [references](references/)：
[踩坑清单](references/pitfalls.md) · [验收清单与命令](references/checklist.md)。

## 项目架构速览

```
main.py                 /config.json /api/{site} /api/{site}/play /api/{site}/poster /health
service/registry.py     SiteRegistry + SiteSpec 装配；每站 HlsRelay 中转策略
core/models.py          VideoCard / VideoDetail / VideoPage（中立记录）
core/adapter.py         BaseTVBoxAdapter：direct|redirect|relay；_media_play_url/_poster_url 跟随 play_mode
core/upstream.py        SiteSession：语义 HTTP → 挑战检测 → CDP 升级 → 身份回灌 → HTTP 重放
core/browser.py         共享浏览器：Turnstile 有界自动点击（武装延迟+≤3次）；陈旧同址页强制重载
core/media.py           HlsRelay（清单重写 + 签名继承 + 15s 微缓存）；PosterService 纯图片代理
sites/<site>/           spider.py · adapter.py · access.py · categories.py · labels.py(可选)
```

依赖方向 `main → service → sites/core`：core 不 import 站点包，站点互不导入；
Spider 返回中立记录，Adapter 做 CMS 映射，HTTP 层不碰上游 HTML。

## 开发流程总览

| 阶段 | 产出 | 验收 |
| --- | --- | --- |
| 0 上游调研 | URL 文法、卡片/详情结构、播放解析链、变体词表、镜像健康度 | 每项有真实抓取证据 |
| 1 站点包骨架 | sites/&lt;site&gt;/ 六件套 | 可独立导入 |
| 2 设计决策 | 决策表逐条落地 | 全部勾选 |
| 3 注册与配置 | SiteSpec + config 变量 + .env.example 同步 | 覆盖校验脚本通过 |
| 4 测试 | 内嵌 fixture 的离线用例 | 全绿且命名合规 |
| 5 端到端与性能 | e2e 接入 + 计时报告 | 见 checklist |
| 6 文档同步 | README 最小更新 + AGENTS 事实记录 | push |

## Phase 0 上游调研（先证据后代码）

1. 参考实现：GitHub 搜 `<site> scraper/downloader`（UAV-Downloader、FetchJAV 等），
   直接拿镜像列表、解析链路与坑位。
2. Wayback 存档：`archive.org/wayback/available` 与 CDX API 取真实 HTML 当离线 fixture。
3. 活体解剖：CF 保护站点用工程远程浏览器走 CDP：

```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    b = p.chromium.connect_over_cdp("http://127.0.0.1:9222", timeout=8000)
    pg = b.contexts[0].pages[0]
    html = pg.content()          # 分析卡片/详情/播放器结构
```

4. 五个必答问题：
   - 列表文法：分类路径？搜索参数？分页是 `/page/N` 还是 `?page=N`？
   - 卡片语法：选择器？懒加载属性？备注里有什么？
   - 详情语法：标题/封面/演员/标签各在哪个节点？
   - 播放链路：按钮 token → 解析接口 → 打包脚本还是明文？**签名是否绑定网络身份？**
   - 多语言前缀：有没有 `/zh/` 一类路径？前缀会不会连变体方括号一起翻译？

## Phase 1 — 站点包骨架

```
sites/<site>/
├── __init__.py     # 一行模块文档："XX TVBox source."
├── categories.py   # 分组表 + 路由映射（分类页与合法性校验共用）
├── labels.py       # 可选：标题前缀 → 客户端标签（参考 missav/supjav 的 labels.py）
├── spider.py       # 上游客户端 + HTML 解析 → 中立记录
├── access.py       # BrowserPolicy 静态白名单 + ready_probe
└── adapter.py      # BaseTVBoxAdapter 子类，只实现差异
```

**categories.py 约定**（参照 `sites/supjav/categories.py`）：分组四元组
`(type_id, 中文名, path, query)`，`query` 为空串表示无参数；派生 `ROUTES` 映射，
未知 `tid` 回落到默认分组。

**access.py 必备**：

```python
def _is_ready(html: str, url: str) -> bool: ...

def access_policy(mode: str) -> BrowserPolicy:
    return BrowserPolicy(
        key="<site>", mode=mode,
        reusable_hosts=frozenset(MIRRORS),
        allowed_hosts=frozenset({*MIRRORS,
            "cloudflare.com", "challenges.cloudflare.com"}),
        ready_probe=_is_ready,
    )
```

`_is_ready` 必须能三分：挑战页（交给共享检测处理）/ 合法空结果 / 就绪文档。

**Spider 必备 API**（签名固定，Adapter 与离线测试都依赖）：

```python
class XxxSpider:
    def __init__(self, session, page_size=24): ...
    def get_categories(self) -> list[dict]                    # [{"type_id","type_name"}]
    def home(self) -> list[VideoCard]
    def category(self, tid, page=1) -> VideoPage
    def search(self, keyword, page=1) -> VideoPage
    def video(self, vod_id, *, select_quality=True) -> VideoDetail
    def detail(self, vod_id) -> VideoDetail                   # 兼容包装 → video()
    def player(self, vod_id) -> dict                          # 兼容包装 → {"url","parse","header"}
```

内部约定：镜像回退统一走 `_get_mirror_html(path, validator)`——挑战异常原样上抛
（保住人工接管页面）、校验失败转 `UpstreamDocumentInvalidError`、全部失败
`UpstreamTransportError`；分页页码取「显式链接最大值」且整页结果视为仍有下一页。

**adapter.py 最小差异**（其余全部继承基类）：

```python
class XxxAdapter(BaseTVBoxAdapter):
    def __init__(self, spider, cache_ttl=CACHE_TTL, page_size=PAGE_SIZE,
                 stale_ttl=0, relay_base_url="", play_mode="direct"):
        super().__init__(cache_ttl, page_size, stale_ttl=stale_ttl,
                         site_key="<site>", relay_base_url=relay_base_url,
                         play_mode=play_mode)
        self.spider = spider

    def home_content(self): ...        # class + filters + list[:page_size]
    def category_content(self, tid, pg="1", filter_data=""): ...   # filter 值必须过白名单
    def search_content(self, keyword, pg="1"): ...
    def cms_detail_content(self, vod_id): ...                      # 媒体/海报路由由基类按 play_mode 处理
```

## Phase 2 设计决策表

- [ ] access.py 只放真实文档主机 + cloudflare.com/challenges.cloudflare.com；
      ready_probe 必须能区分「挑战页 / 合法空结果 / 就绪文档」三种状态。
- [ ] vod_id 白名单式严格归一（如纯数字），拒绝路径穿越。
- [ ] 有多语言前缀时默认中文路由；变体方括号会被一并翻译 → 标签表需双语。
- [ ] 变体标签只取标题开头连续方括号块；已知词表长词优先防子串误吞。
- [ ] 解析顺序数据驱动（按钮 token 倒序 → resolver），尝试上限 3 次；
      打包脚本不要用字面量 m3u8 预过滤（扩展名可能是编码 token）。
- [ ] **质量预选只在 direct 模式做**；relay/redirect 把母清单交给播放器自适应，
      省一次上游往返。
- [ ] 海报直出原图、卡片 style.ratio = 原图宽高比；是否经代理跟随该站 play_mode。
- [ ] **尽早测签名的网络绑定**：用 r.jina.ai / api.allorigins.win / api.codetabs.com
      三个第三方出口拉同一签名地址，对照本机结果——绑定则跨网络必须 relay。

## Phase 3 注册与配置

```python
# service/registry.py
<site>_MEDIA_POLICY = UrlPolicy(allowed_hosts=frozenset({…}))       # 静态媒体域白名单
<site>_relay = HlsRelay(http_session, <site>_MEDIA_POLICY,
                        public_base_url=PUBLIC_BASE_URL, site_key="<site>")
SiteSpec(key="<site>", name="...", adapter=XxxAdapter(
            spider, cache_ttl=CACHE_TTL, page_size=PAGE_SIZE,
            stale_ttl=CACHE_STALE_TTL, site_key="<site>",
            relay_base_url=PUBLIC_BASE_URL, play_mode=<SITE>_PLAY_MODE),
         relay=<site>_relay, client_headers=…, style=dict(_LANDSCAPE_STYLE))
```

config.py 增加 `TVBOX_<SITE>_ACCESS_MODE` 与 `TVBOX_<SITE>_PLAY_MODE`；
`.env.example` 必须同步（跑 checklist 里的覆盖校验脚本）。

## Phase 4 测试规范

- fixture HTML 内嵌在测试文件里，绝不访问真实上游。
- 列表/详情/搜索/分页/海报形态断言 + `AdapterContractMixin.assert_contract()`。
- 命名 `test_<expected_behavior>`；空结果、非法 id、挑战分类都要有用例。
- 时序类测试用 `mock.patch("core.browser._CHALLENGE_AUTOCLICK_*", …)` 固定节奏。
- 运行务必 `set -o pipefail` 再接管道，防止 tail 吞掉退出码。

## Phase 5 端到端与性能

把站点加进 `tests/live/e2e_test.py` 的 run_site_tests 序列；
性能验收用 references/checklist.md 的分段计时模板（detail/master/child/segment 四段），
播放起步预算参考：detail ≤3s，重复清单 ≤0.5s（微缓存命中）。

## Phase 6 文档同步

- `README.md`：只更新「站点能力」表与新增环境变量行，保持快速开始导向；
  行为性结论一句话带过即可。
- `AGENTS.md`：记录与其他站点不同的事实性结论（如某 CDN 签名绑定网络、
  某镜像已停放成广告域），供后续会话直接引用。
- `deploy/fnos/.env.example`：新增变量必须同步，并跑覆盖校验脚本
  （见 references/checklist.md）。
- `tvbox_config.json`：静态示例的 style.ratio 与站点清单对齐注册表生成结果。

完成后按 references/checklist.md 的「发布顺序」执行验证与推送。
