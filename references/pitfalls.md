# 踩坑清单（本次 SupJav 开发全程实证）

每条都真实发生过；遇到"看起来应该工作但没反应"时先对照此表。

## 1. CDN 签名绑定解析方网络 → 跨网络直链不可行
- premilkyway/turboviplay：URL 内嵌 `asn=`，第三方出口拉取 403/522。
- Streamtape：`get_video` 带混淆 `ip=` 参数，第三方出口 403 Access Denied。
- **判定方法**：解析后立即用三个第三方代理服务（r.jina.ai、api.allorigins.win/raw、
  api.codetabs.com/v1/proxy）并行拉同一地址，与本机结果对比。
- 结论写进 README；跨网络部署该站必须 relay。redirect 模式仅适合同出口部署。

## 2. Turnstile 自动点击时序
- 勾选框渲染完 ≠ 可交互：需要 ~4s 武装延迟，过早点击会被静默重置并烧掉预算
  （表现为"首次请求失败、第二次才过"——第二次是新预算+控件已成熟）。
- 点击成功后 Cloudflare 立即导航离开挑战页，轮询的 content() 会抛
  "Unable to retrieve content because the page is navigating and changing the content."
  —— 必须按软错误处理继续轮询，否则把成功的通过误杀成 60s 冷却。
- 接受点击后的守卫不要在重新读页之前 break 轮询循环。

## 3. Turnstile widget 的 DOM 可见性
- 勾选框 iframe 在 closed shadow root 里：querySelectorAll 数量为 0，
  frame.url 可能还是空串（导航未提交）。
- 正确姿势：枚举非主 frame → frame_element().bounding_box()（CDP 层可穿透）；
  只在 looks_like_challenge 时执行；未找到不消耗尝试次数。
- 不要注入 attachShadow 重写——会被识别判负。

## 4. 无头浏览器过不了放行
点击全部成功、cf_clearance 也发了，刷新仍循环挑战：UA 是 HeadlessChrome +
SwiftShader WebGL 的组合指纹被拒。浏览器必须有头（Xvfb/桌面）。

## 5. 镜像健康度
- 上线前逐个验证镜像不是停放域：supjav.net/org 会 302 到 Shopee 广告，
  每次回退白烧几十秒且污染请求。
- 空搜索结果是合法文档（posts 容器无卡片），镜像校验器必须接受，
  否则误触发回退。

## 6. 清单签名继承
部分 CDN 母清单带签名、子清单里分片是不带签名的相对路径：
中转重写时要把来源清单的 query 继承到无签名的分片/KEY 行上，
已签名的行保持原样（防止 query 双写导致畸形 URL）。

## 7. 打包脚本解包
Dean-Edwards packer 的扩展名可能是编码 token——不要用字面量 "m3u8"
预过滤脚本；base 可能到 36，注意 count 上限保护。

## 8. 多语言前缀
qTranslate 类站点 /zh/ 前缀会连变体方括号一起翻译（[Reducing Mosaic]→[无码破解]）。
标签词表必须双语收录，长词优先防 "无码" 吞掉 "无码破解"。

## 9. 陈旧同址标签页
重启/复用后标签页 URL 与目标一致但内容是半加载垃圾：被动轮询会烧满整个预算。
同址续接前先做一次快速预检——既非就绪也非挑战就强制 goto 重载。

## 10. 中转端点 400 = 白名单缺域名
SupJav 这类会轮换播放商域名的站点，看到 `/api/<site>/play?u=https%3A%2F%2F<陌生域>`
返回 400 即为媒体白名单缺失：优先 TVBOX_<SITE>_RELAY_HOSTS 追加，稳定后再进代码默认表。
