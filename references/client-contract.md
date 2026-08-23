# 客户端契约 —— 菜单（class）与筛选项（filters）规划指南

FongMi TVBox 如何消费我们输出的菜单与筛选、以及为新站点规划它们的设计准则。

## 1. 数据流

```
home_content() 返回:
{
  "class":   [{"type_id": "latest", "type_name": "最新发布"}, …],   # 顶部分类页签
  "filters": {"<type_id>": [ <筛选组>, … ], …},                     # 按 tid 键控
  "list":    [ VideoCard… ]
}

用户进入某分类并选择筛选项后，客户端请求:
GET /api/{site}?ac=detail&t=<type_id>&pg=<页码>&f=<URL编码JSON>
其中 f = 所有筛选组选中值的合并对象，如 {"sub":"/dm34/luxu"} 或
{"tag":"NTR","sort":"本週排行","date":"…","duration":"…"}
```

`dispatch_cms` 把 `t/pg/f` 原样交给 `adapter.category_content(tid, pg, filter_data)`；
**校验与拼装完全是 Adapter 的责任**。

## 2. 字段参考

| 字段 | 说明 |
| --- | --- |
| `type_id` | 分类页签标识，原样回传于 `t=`；未知值必须回落默认分组 |
| `filters` 的键 | `type_id`；不支持筛选的分类不要出现 |
| 筛选组 `key` | 回传 `f` JSON 里的键名；同站内唯一 |
| 筛选组 `name` | 面板显示名（中文） |
| 筛选组 `init` | 默认选中的 `v`；惯例空串 = 全部 |
| `value[]` | `{"n": 显示名, "v": 回传值}`；首项惯例 `{"n":"全部","v":""}` |

## 3. 规划准则

1. **菜单 6–10 个**，中文短词命名；只放上游稳定存在的路径。
2. **先分清「条目变体」与「可查询维度」**——这是唯一的规划难点：
   - 同一部片的不同版本是独立条目（SupJav 中字/无码破解）→ 平铺为菜单或卡片标签，
     **不要**做成筛选；
   - 同一列表支持服务端参数查询（Hanime1 标签/排序/日期/时长）→ 才做成 filters 组。
3. 每个筛选组 = 一个独立查询维度；`v` 必须是白名单 token（slug/枚举/受控路径），
   绝不透传任意字符串。
4. 首项恒为 `{"n":"全部","v":""}`，让用户能清空该维度。
5. 多组可以同时生效：客户端把所有组的选中值合并进同一个 `f`——
   Adapter 必须逐 key 白名单校验后再拼上游查询。
6. 无筛选的站点输出 `"filters": {}` 即可。

## 4. 四站对照（实现范本）

| 站点 | class 菜单 | filters 设计 | 回传校验函数 |
| --- | --- | --- | --- |
| JableTV | 最新/热门/新片/快速选片 | 仅快速选片：多分组 `tag_N`，**有意单选**（取首个有效值） | `sites/jable/adapter.py:_selected_tag` |
| MissAV | 观看日本AV等四分组 | 每组一个 `sub` → 子路径白名单 | `sites/missav/adapter.py:_selected_path` |
| Hanime1 | 影片类型 | `tag/sort/date/duration` 四维同时生效 | `sites/hanime1/adapter.py:_selected_filters` |
| SupJav | 十个精选分类 | `{}`（上游无可查询参数） | — |

## 5. 新站点落地步骤

1. 从 P0 笔记挑出「可作为查询参数」的维度，每个设计一组（key/name/values）。
2. 在 `categories.py` 定义 GROUPS 与各维度的白名单常量（`VALID_*` 集合）。
3. `home_content` 输出 class 与 `filters = {tid: [groups…]}` 或 `{}`。
4. `category_content` 解析 `filter_data`：逐 key 白名单校验 → 拼入上游查询 → 缓存键包含筛选值。
5. 测试：合法组合、非法组合（全部被丢弃）、空筛选、多组同时生效。
