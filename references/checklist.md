# 验收清单与命令模板

## 提交前（每次）

```bash
set -o pipefail
python3 -m unittest discover -s tests        # 全绿
python3 -m compileall -q main.py config.py core service sites tests
git diff --check
for s in deploy/fnos/*.sh; do sh -n "$s"; done
```

## .env.example 覆盖校验

```bash
python3 - <<'PY'
import re
cfg = set(re.findall(r'"(TVBOX_[A-Z0-9_]+)"', open("config.py").read()))
ex  = set(re.findall(r'^(TVBOX_[A-Z0-9_]+)=', open("deploy/fnos/.env.example").read(), re.M))
print("missing:", sorted(cfg-ex) or "none")
print("unknown:", sorted(ex-cfg) or "none")
PY
```

## e2e 接入

`tests/live/e2e_test.py`：search_keywords 字典加站点词条、
run_site_tests 序列追加、health/config 断言的站点数同步、
海报断言分支按站点形态选择（proxy/raw/signed）。

## 性能计时模板

```bash
B=https://<入口>
curl -sS -m 90 "$B/api/<site>?ac=detail&ids=<id>"   # detail ≤3s(直链) / ≤6s(relay)
# master → child → segment 各段：
for u in <master> <child> <seg>; do
  curl -sS -m 40 -o /dev/null -w "%{http_code} ttfb=%{time_starttransfer} total=%{time_total} %{size_download}B\n" "$u"
done
# 重复请求第二遍应命中 15s 微缓存（≤0.5s）
```

## 播放模式判定（上线前必做一次）

解析出播放地址后立即用三个第三方出口并行拉取：

```bash
for p in "https://r.jina.ai/URL" \
         "https://api.allorigins.win/raw?url=ENC" \
         "https://api.codetabs.com/v1/proxy?quest=ENC"; do
  curl -sS -m 45 -o /dev/null -w "%{http_code}\n" "$p"
done
```

- 第三方 200 → 直链可行，默认 direct。
- 第三方 4xx/超时 → 网络绑定，跨网络部署必须 relay；文档标注清楚。

## 发布顺序

1. 单元全绿 + compileall + git diff --check
2. push GitHub
3. NAS：git pull → stop.sh → start.sh
4. 公网冒烟：/config.json 样式、各站首页卡片、一条详情+分片 200
5. README/.env.example/tvbox_config.json 已同步（含 style.ratio）
6. 海报跟随 play_mode 的两条断言在用例集中
