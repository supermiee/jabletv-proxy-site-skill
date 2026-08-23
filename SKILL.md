---
name: jabletv-proxy-new-site
description: 在 jabletv-proxy 工程中开发新站点适配的标准流程。当需要为该工程新增上游站点（Spider / Adapter / 注册装配 / 测试 / 端到端验证）时使用。按本流程执行，即使面对结构完全未知的站点也能完成从调研到上线的全部工作。
---

# jabletv-proxy 新站点开发

为 `jabletv-proxy`（FongMi TVBox 多站点 CMS 代理）新增上游站点适配。
范例实现：`sites/supjav/`（最新最全，含解析链/标签/多模式）；其余 `sites/missav|jable|hanime1`。

深入材料：

| 文件 | 用途 |
| --- | --- |
| [references/research-playbook.md](references/research-playbook.md) | Phase 0 调研交付物模板 + 证据采集配方 + 网络绑定判定 |
| [references/testing-harness.md](references/testing-harness.md) | 假会话测试脚手架代码模板 + 每站必测用例清单 + 错误升级矩阵 |
| [references/pitfalls.md](references/pitfalls.md) | 全程实证踩坑清单（先读一遍再动手） |
| [references/client-contract.md](references/client-contract.md) | 客户端菜单(class)/筛选(filters)契约与规划准则 |
| [references/checklist.md](references/checklist.md) | 验收命令、性能计时模板、发布顺序 |

## 项目架构速览

```
main.py                 /config.json /api/{site}[/play|/poster] /health
service/registry.py     SiteRegistry + SiteSpec；每站 HlsRelay(UrlPolicy) 中转策略
core/models.py          VideoCard(id,name,pic,remark) VideoDetail(...play_url) VideoPage(items,page,pagecount)
core/adapter.py         BaseTVBoxAdapter：direct|redirect|relay；媒体与海报路由均跟随 play_mode
core/upstream.py        SiteSession：语义 HTTP → 挑战检测 → CDP 升级 → 身份回灌 → HTTP 重放
core/browser.py         共享浏览器：Turnstile 有界自动点击；陈旧同址页强制重载
core/media.py           HlsRelay（重写+签名继承+15s 清单微缓存）；PosterService 图片代理
sites/<site>/           spider.py · adapter.py · access.py · categories.py · labels.py?
```

依赖方向 `main → service → sites/core`；core 不 import 站点包，站点互不导入；
Spider 出中立记录，Adapter 做 CMS 映射，HTTP 层不碰上游 HTML。

## 流程总览

| 阶段 | 产出 | 验收 |
| --- | --- | --- |
| P0 上游调研 | research-notes（模板见 playbook） | 五问全部有抓取证据 |
| P1 站点包骨架 | 六件套 | 可独立导入，签名符合约定 |
| P2 设计决策 | 决策表逐条落地 | 全部勾选 |
| P3 注册与配置 | SiteSpec + config 变量 + .env.example 同步 | 覆盖校验通过 |
| P4 测试 | 假会话离线用例集 | 必测清单全覆盖 |
| P5 e2e 与性能 | live 接入 + 四段计时报告 | 见 checklist |
| P6 文档同步 | README/AGENTS/.env.example/tvbox_config | push |

## P0 上游调研

**交付物是 `research-notes.md`，不是代码。** 严格按
[research-playbook.md](references/research-playbook.md) 的模板填写：
五问证据（列表文法/卡片语法/详情语法/播放链路/多语言）、镜像健康表、
网络绑定四格实测、原图尺寸采样。三个信息源按序使用：
参考爬虫仓库 → Wayback 存档 fixture → 远程浏览器 CDP 活体解剖。

## P1 站点包骨架

文件职责、Spider 七个必备方法签名、access.py 三态 ready_probe、
Adapter 最小差异面 —— 见 [testing-harness.md](references/testing-harness.md) 附录，
代码范本直接对照 `sites/supjav/spider.py` 与 `adapter.py`。

## P2 设计决策表（逐条落地）

**接入层**
- [ ] access.py 只放真实文档主机 + cloudflare.com/challenges.cloudflare.com。
- [ ] ready_probe 三分语义：挑战页（交共享检测）/ 合法空结果（如搜索无结果的
      posts 容器）/ 就绪文档；镜像校验器语义与之对齐。
- [ ] 多镜像逐一健康验证——停放域（302 到广告/购物站）必须剔除。

**数据层**
- [ ] vod_id 白名单式严格归一（如纯数字 `\d{1,12}`），只信任自家列表解析产物。
- [ ] 列表备注 = 变体标签（若有）＋可选日期；标签词表双语收录且长词优先；
      只取标题开头连续方括号块。
- [ ] 有多语言前缀时默认中文路由；前缀会连方括号一起翻译。

**播放层**
- [ ] 解析顺序数据驱动（按钮 token 规则化提取后倒序 → resolver 端点），
      尝试上限 3 次；打包脚本禁用字面量 m3u8 预过滤。
- [ ] 质量预选只在 direct 模式做；relay/redirect 返回母清单交给播放器自适应。
- [ ] **P0 已判定的网络绑定结论决定该站默认播放模式的建议**（绑定 → 文档标注
      「跨网络必须 relay」）。

**展示层**
- [ ] 海报直出原图；style.ratio = P0 实测的原图宽高比；代理与否跟随 play_mode。
- [ ] 菜单与筛选项规划遵循客户端契约——先分清「条目变体 vs 可查询维度」，
      逐 key 白名单校验（详见 [references/client-contract.md](references/client-contract.md)）。
- [ ] 搜索输入归一化按站点习惯（范本 `sites/jable/spider.py` 番号连字符补全）。

## P3 注册与配置

```python
<site>_MEDIA_POLICY = UrlPolicy(allowed_hosts=frozenset({…}))    # 该站媒体域名白名单
<site>_relay = HlsRelay(http_session, <site>_MEDIA_POLICY,
                        public_base_url=PUBLIC_BASE_URL, site_key="<site>")
SiteSpec(key="<site>", name="…", adapter=XxxAdapter(spider,
            cache_ttl=CACHE_TTL, page_size=PAGE_SIZE, stale_ttl=CACHE_STALE_TTL,
            site_key="<site>", relay_base_url=PUBLIC_BASE_URL,
            play_mode=<SITE>_PLAY_MODE),
         relay=<site>_relay, client_headers=…, style=dict(_LANDSCAPE_STYLE))
```

config.py 增加 `TVBOX_<SITE>_ACCESS_MODE` / `TVBOX_<SITE>_PLAY_MODE`；
`.env.example` 同步并跑覆盖校验脚本（checklist）。client_headers 只放
「播放器直连时确实需要」的头（实测过 Referer/UA 影响后再加）。

## P4 测试

按 [testing-harness.md](references/testing-harness.md)：假会话脚手架代码 +
每站必测用例清单（契约/分页文法/空结果/非法 id/挑战分类/变体标签/解析链/
镜像回退/海报跟随模式/relay 白名单）。fixture 必须来自 P0 真实抓取；
时序常量一律 mock.patch 固定；管道前 set -o pipefail。

## P5 e2e 与性能

站点加入 `tests/live/e2e_test.py`；性能验收跑 checklist 的四段计时模板
（detail/master/child/segment），阈值：detail ≤3s(direct)/≤6s(relay 首次)、
重复清单 ≤0.5s（微缓存命中）、分片吞吐记录基线。

## P6 文档同步

README「站点能力」表一行 + 新增环境变量行；AGENTS.md 记录事实性结论
（签名绑定、镜像状态、特殊解析链）；`.env.example` 与 `tvbox_config.json`
对齐注册表生成结果。
