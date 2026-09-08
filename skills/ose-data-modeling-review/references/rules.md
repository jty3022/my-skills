# OSE 数据规则

基于用户提供的 Data Team Rulebook，按已确认边界调整，保留原编号方便评审。以下调整优先于原文；工具路线不是当前强制架构。

## R1–R3：命名、注释和管理字段

- 新表和字段用小写业务英文、下划线，表名单数；禁止拼音缩写、保留字和 field1 等无意义字段。
- 业务表默认自增 bigint id；外键使用“被引用表名_id”；布尔加 is_；时间点用 _at，日期用 _date。
- 每张新表、每个新增或修改含义的字段必须有注释；枚举列全取值含义；表注释写业务用途和权威来源；关联字段写引用对象。含义改变同步更新注释。
- 业务表默认带 id、created_at、updated_at、is_deleted。业务数据软删除；日志、流水表允许物理删除但须注释声明，执行删除仍遵循项目权限规则。
- 修改存量表只检查本次影响部分，不因规范差异强制更换已有主键、重命名无关字段或整表整改。

## R4：索引

外键须有合适索引；已有以它为最左列的可用联合索引，不重复加单列索引。业务唯一性用数据库唯一约束兜底，不只应用层先查。

高频过滤和排序按查询设计索引，注意最左前缀。使用 MySQL EXPLAIN 检查访问方式、估计行数和索引；大表全扫描需分析原因，小表扫描不自动判错。索引名用 idx_表名_字段名，联合索引依次列字段。超过五个索引说明收益与写入成本，不直接禁止。

明确软删除后业务编号能否复用；给唯一键追加 is_deleted 并不自动支持任意多次删除重建。

## R5：建模

建表前明确实体、关系、枚举、权威来源及哪些字段会变。多对多建中间表并注明关联；有组合唯一语义时用约束表达。

业务信息默认单处维护。宽表、冗余、JSON（JavaScript Object Notation，JavaScript 对象表示法）结合用途评审，不一律禁止。冗余说明来源、更新责任和同步方式；固定且需筛选、关联、约束的结构优先明确列。字段数量不是单独拆表依据。

## R6：事务与并发

“读取—计算—写回”必须考虑并发。默认 version 乐观锁：更新递增版本并匹配旧版本，影响零行时重新读取并重试或提示冲突。已有正确的原子更新或其他保障时，不机械叠加锁；检查业务条件也受保护。

高冲突且重试代价大时考虑事务内 SELECT ... FOR UPDATE；不在长事务或外部调用期间持锁。事务只包必要原子操作，唯一性依赖数据库约束。

## R7：权威来源

核心数据只有一个 SSOT（Single Source of Truth，单一真相源），其他系统为引用或受控副本。

原文约定员工来自飞书/HRIS（Human Resource Information System，人力资源信息系统）、项目编号来自 ERP（Enterprise Resource Planning，企业资源计划系统）、设备位号来自 E3D 导出。具体 OSE 实体先核对实际来源，不能无证据地替换来源或接入外部系统。来源不明时澄清，确认后写注释。

## R8–R11：未来数仓，按需适用

当前不要求创建这些层级，也不把 MySQL 数据库与 PostgreSQL schema 用法混用。

| 层 | 全称 | 职责 |
| --- | --- | --- |
| ODS | Operational Data Store，操作数据存储层 | 原始追加落地，保留源字段，加 _source_system、_extracted_at、_batch_id，记录入库结果；不强加自增和软删除 |
| DWD | Data Warehouse Detail，数据仓库明细层 | 清洗、统一编号与字段，建立映射 |
| DWS | Data Warehouse Summary，数据仓库汇总层 | 按主题和粒度汇总，冗余写来源与同步方式 |
| ADS | Application Data Store，应用数据层 | 应用可直接使用的最终结果 |

明细、汇总、应用层按数据粒度设计唯一键和管理字段。例如项目月汇总可用 project_id 与 month 唯一标识。

采用数仓后原始落地不手改，加工与发布结果可重建，加工逻辑进入 Git。若使用 dbt（data build tool，数据构建工具），每个模型至少一至两条有意义的测试，失败拦截发布；当前不强制引入。

实际对接 Ontology 时，ODS 对应 Bronze，DWD/DWS 对应 Silver，ADS 对应 Gold，权威来源保持一致。

## R12：共库过渡规则

当前业务与报表共用 MySQL 服务，允许经现有后端受控查询，不能套用原文的一律禁止。

新增或修改的分析查询限制合理业务范围（例如时间、项目）；明细列表分页或限制返回数量；汇总也需控制扫描范围，仅限制返回行数不足以控制扫描成本。结合执行计划检查关联、扫描及排序成本。

未来有只读副本或独立分析实例后逐步迁移，不自动部署或改连接。前端和调用方经后端接口访问，不获取数据库凭据。

## R10.5、R13–R15：指标、接口和职责

业务指标后端统一定义与计算，各报表复用同一口径；前端可做日期和千分位展示，不重复实现成本、业务金额、完成率等算法。新指标先查已有定义，新增查询按现有后端分层实现。

新增来源先确认数据契约和同步方式；仅变展示由前端处理。对外提供 API（Application Programming Interface，应用程序编程接口），不给数据库凭据。沿用项目鉴权与权限机制，不绕过权限，不强制替换为 PostgREST、行级策略或特定网关。

AI（Artificial Intelligence，人工智能）应用默认只读，写入走授权的正式业务端点。数仓发布结果通过修正来源或加工逻辑后重建；普通业务记录修改仍走业务端点，不能要求所有业务修改都重跑加工流程。

## R16：评审分级

本次范围内缺少注释、枚举不全、缺业务唯一约束、并发无保护、重复指标、暴露数据库凭据、适用外键缺索引属于必须修复。受控共库查询本身不是红线。

宽表、超过五个索引、冗余和 JSON 要求说明设计理由，结合事实判断。数仓及 dbt 要求只在实际采用后适用。原文技术选型、建设时间表和湖仓阈值仅作参考，不自动触发迁移。

## MySQL 示例

独立示意，不直接作为生产迁移；实际字段、枚举、来源与版本策略按业务确定。示例中 project_no 删除后不可复用。

```sql
CREATE TABLE project (
    id BIGINT NOT NULL AUTO_INCREMENT COMMENT '项目记录主键',
    project_no VARCHAR(64) NOT NULL COMMENT '项目编号，删除后不可复用',
    name VARCHAR(100) NOT NULL COMMENT '项目名称',
    status TINYINT NOT NULL COMMENT '项目状态: 1=未开始 2=进行中 3=已完成',
    version INT NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '记录创建时间',
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
        ON UPDATE CURRENT_TIMESTAMP COMMENT '记录最后修改时间',
    is_deleted TINYINT NOT NULL DEFAULT 0 COMMENT '删除标记: 0=正常 1=已删除',
    PRIMARY KEY (id),
    UNIQUE KEY idx_project_project_no (project_no)
) ENGINE=InnoDB COMMENT='项目主表；示例假定编号以 ERP 为权威来源，名称与状态由 OSE 维护';

UPDATE project
SET status = 2, version = version + 1
WHERE id = 100 AND version = 3 AND is_deleted = 0;
-- 必须检查影响行数；零行时重读，区分不存在、已删除和版本冲突。
```
