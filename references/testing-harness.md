# 测试脚手架与必测清单

## 假会话脚手架（照抄改路由即可）

```python
class XxxSession:
    """get_html 按文档类型返回 fixture；get 返回解析器/清单响应。"""
    def __init__(self):
        self.html_calls = 0
        self.urls, self.get_calls = [], []
        self.blocked_hosts, self.challenge_hosts = set(), set()

    def get_html(self, url):
        self.last_url = url; self.urls.append(url); self.html_calls += 1
        if any(f"https://{h}" in url for h in self.challenge_hosts):
            raise BrowserChallengeError("waiting")
        if any(f"https://{h}" in url for h in self.blocked_hosts):
            raise RuntimeError("challenge")
        return DETAIL_FIXTURE if re.search(r"/\d+\.html$", url) else LISTING_FIXTURE

    def get(self, url, **kw):
        self.get_calls.append(url)
        if "resolver" in url:                       # 解析端点按 token 路由
            return SimpleNamespace(text=self.embed_bodies.get(url, ""))
        return SimpleNamespace(text=self.playlist_bodies.get(url, ""))  # 清单/分片
```

要点：`challenge_hosts` 抛 BrowserChallengeError（模拟人工接管语义）、
`blocked_hosts` 抛普通异常（模拟传输层失败）；两者行为差异正是升级矩阵要测的。

## 错误分类与升级矩阵（Spider/网关行为契约）

| 上游现象 | Spider/网关应抛 | 客户端看到 | 后续行为 |
| --- | --- | --- | --- |
| DNS/连接失败 | UpstreamTransportError | 502 retry_later | 不启 CDP |
| CF 挑战页(标记或403) | ChallengeDetectedError→浏览器 | — | 自动点击/人工接管 |
| 非挑战但结构不符 | UpstreamDocumentInvalidError | 502 inspect_parser | 不重试不冷却误报 |
| 浏览器内挑战超时 | BrowserChallengeError | 503 waiting_manual | 页面保留可续接 |
| 总预算耗尽 | UpstreamDeadlineError | 504 | — |

## 每站必测用例清单

1. **契约**：AdapterContractMixin 四方法 + home filters 结构。
2. **列表文法**：fixture → VideoCard 全字段逐项断言（含懒加载属性剥离、
   相对/绝对 URL、备注组成规则）。
3. **分页文法**：每个分类形态一断言；未知 tid 回落默认分组。
4. **搜索**：编码、归一化规则（如有）、空结果返回空列表而非回退。
5. **非法 id**：路径穿越/非空/带参数 各抛 ValueError。
6. **详情元数据**：单次 html_calls==1 断言 + 字段映射。
7. **播放解析链**：源优先级顺序、尝试上限、打包脚本解出、ST 兜底、
   全部失败 play_url 为空。
8. **镜像回退**：blocked_hosts 全镜像 → UpstreamTransportError；
   challenge_hosts 单镜像 → BrowserChallengeError 且 URL 不再前移。
9. **海报跟随模式**：direct 直出 / relay 包装为 /api/<site>/poster。
10. **relay 白名单**：已知域 ✓ / 陌生域 ✗。

时序类测试固定常量：

```python
pacing = mock.patch.multiple("core.browser",
    _CHALLENGE_AUTOCLICK_MAX_ATTEMPTS=2,
    _CHALLENGE_AUTOCLICK_FIRST_DELAY_SECONDS=0,
    _CHALLENGE_AUTOCLICK_RETRY_DELAY_MS=50)
with pacing:
    with self.assertRaises(BrowserChallengeError):
        runtime.fetch(policy, url)
```

## 附录：Phase 1 骨架签名速查

Spider 七方法：get_categories/home/category/search/video(vod_id,*,select_quality)/
detail/player。镜像回退统一 `_get_mirror_html(path, validator)`：挑战上抛、
校验失败转 DocumentInvalid、全败 TransportError。分页页码取显式链接最大值，
整页视为仍有下一页。

Adapter 构造：`(spider, cache_ttl, page_size, stale_ttl, site_key,
relay_base_url, play_mode)`；四个 CMS 方法中 filter_data 必须过白名单
（范本 hanime1/_selected_filters），home_content 输出 filters 结构。
