# 实施规格

SPEC（Implementation Specification，实施规格）。

文件：SKILL.md 负责触发、流程和边界；references/rules.md 是详细规则与示例的唯一来源；agents/openai.yaml 允许自动选用；PRD.md 保存需求。

仅在 OSE 工作区 .agents/skills/ose-data-modeling-review 下新增文件，不改业务代码、数据库、全局个人技能或三个子仓库。工作区根目录不是 Git 仓库，不创建分支、不提交。

验证：运行 skill-creator 的 quick_validate.py；检查文件非空、引用存在、无未完成模板、配置与确认决策一致。

场景走查：受控后端报表允许；前端重复成本算法需纠正；历史无关字段不整改；六个索引要求理由而非直接禁止；未来汇总表不强制自增主键；非 OSE 任务不适用。

不连接数据库、不运行业务测试。静态检查不证明示例已在目标 MySQL 运行，也不证明当前会话动态加载了技能；实际自动选用需在能发现新技能的后续会话验证。
