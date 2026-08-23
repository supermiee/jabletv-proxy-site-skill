# jabletv-proxy 新站点开发技能

为 [jabletv-proxy](https://github.com/supermiee/jabletv-proxy)（FongMi TVBox 多站点 CMS 代理）
沉淀的新站点适配开发技能：把 SupJav 站点从调研、实现、测试到公网上线的完整过程标准化，
包含可直接套用的决策表、分段计时模板与全程实证的踩坑清单。

## 安装

opencode：复制到 `~/.config/opencode/skills/jabletv-proxy-new-site/`
Claude Code：复制到 `.claude/skills/jabletv-proxy-new-site/`

## 触发

当对话中出现「给 jabletv-proxy 加一个新站点 / 新增 XX 源适配」类需求时自动适用；
也可显式要求按该技能流程执行。

## 文件

| 文件 | 内容 |
| --- | --- |
| `SKILL.md` | 六阶段流程 + 接入层/数据层/播放层/展示层设计决策表 |
| `references/research-playbook.md` | Phase 0 交付物模板、证据采集配方、网络绑定四格实测 |
| `references/testing-harness.md` | 假会话脚手架代码、错误升级矩阵、每站必测用例清单 |
| `references/client-contract.md` | FongMi 菜单与筛选项契约、规划准则、四站对照 |
| `references/pitfalls.md` | 实证踩坑清单（CDN 签名绑定、Turnstile 时序、镜像健康度…） |
| `references/checklist.md` | 验收命令、覆盖校验脚本、性能计时模板、发布顺序 |

范例实现：jabletv-proxy 仓库 `sites/supjav/`。

---

最后验证：2026-08-23 · 对应 jabletv-proxy commit `d18dd33` · skill version 1.0.0-supjav
