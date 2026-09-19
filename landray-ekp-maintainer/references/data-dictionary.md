# EKP 数据字典与诊断速查

> 本文件用于核对表名、列名与状态码。**任何库表级结论动手前都应以实际环境的《数据字典》为准。**

## 数据字典的获取方式

系统内置了数据字典查询页，是核对表结构最可靠的来源：

```
入口：http://<域名>:8081/sys
路径：运维管理 → 管理员工具箱 → 常用工具集 → 查看数据字典
用法：选择对应模块后查看表与字段定义
```

比翻官方文档更快，且反映的是**本环境实际的**结构（含二开改动）。

## 核心表清单

| 表名 | 用途 | 关键列 |
|------|------|--------|
| `km_review_main` | 流程实例主表 | `fd_id` 实例 ID、`doc_subject` 标题、`doc_status` 状态码、`doc_create_time` 发起时间、`extend_data_xml` 表单数据（XML） |
| `sys_org_person` | 人员 | `fd_id` 主键、`fd_no` 工号、`fd_login_name` 登录名 |
| `sys_org_element` | 组织（部门/机构） | `fd_id` 主键、`fd_name` 名称 |
| `ekp_<hash>` | 各流程表单的数据表 | 表名由平台按模板生成（如 `ekp_1933defb2ec2b62c1d7f`），含 `fd_id` 与各控件列 |
| `sys_*` | 系统配置类表 | 视版本而异，以数据字典为准 |

**关联规律**：流程主表、表单数据表、组织表之间统一以 `fd_id` 关联。查人时需联 `sys_org_element` 取 `fd_name`。

## 流程状态码（`km_review_main.doc_status`）

| 取值 | 含义 |
|------|------|
| `10` | 草稿 |
| `20` | 待审 |
| `11` | 驳回 |
| `00` | 废弃 |
| `30` | 结束（已审批完成） |

这是排查流程问题时**最先要看的一个字段**——它直接告诉你单子处于生命周期的哪个阶段。

## 表单列的命名规律

表单控件生成的数据列，列名通常是**控件中文名的拼音**。例如"流程编号"对应 `fd_liuChengBianHao`。

这条规律的实用价值：拿到一个陌生环境的库，看到 `fd_shenQingRenYuan` 能立刻反应出是"申请人员"。但**不要据此猜列名去写 SQL**——拼音的连写方式因配置习惯而异，务必以数据字典核对。

## 诊断语句集

### 按标题查流程实例

```sql
-- 用途：定位流程实例并查看其状态
-- 输入：流程标题关键字
-- 输出：实例 ID、标题、状态码、发起人、发起时间
SELECT
    m.fd_id            AS 实例ID,
    m.doc_subject      AS 流程标题,
    m.doc_status       AS 状态码,
    m.doc_create_time  AS 发起时间,
    p.fd_login_name    AS 发起人登录名,
    e.fd_name          AS 发起人姓名
FROM km_review_main m
LEFT JOIN sys_org_person  p ON m.doc_creator_id = p.fd_id
LEFT JOIN sys_org_element e ON m.doc_creator_id = e.fd_id
WHERE m.doc_subject LIKE '%' + @keyword + '%'
ORDER BY m.doc_create_time DESC;
```

### 统计各状态的在途单据量

```sql
-- 用途：评估配置改动的在途影响面
-- 输出：各状态码对应的单据数量
-- 注意：做任何流程模板改动前先跑这条，把在途数量报给业务确认
SELECT
    doc_status            AS 状态码,
    COUNT(*)              AS 单据数,
    MIN(doc_create_time)  AS 最早发起,
    MAX(doc_create_time)  AS 最晚发起
FROM km_review_main
WHERE doc_create_time >= DATEADD(DAY, -90, GETDATE())
GROUP BY doc_status
ORDER BY doc_status;
```

### 按工号查人员及其部门

```sql
-- 用途：核对审批人是否仍在该部门，排查"审批人离职导致流程无人处理"
SELECT
    p.fd_id          AS 人员ID,
    p.fd_no          AS 工号,
    p.fd_login_name  AS 登录名,
    e.fd_name        AS 姓名,
    e.fd_parent_id   AS 上级组织ID
FROM sys_org_person p
LEFT JOIN sys_org_element e ON p.fd_id = e.fd_id
WHERE p.fd_no = @empNo OR p.fd_login_name = @loginName;
```

## 使用这些语句的三条纪律

模板中给出的语句是**只读查询**，可以放心在只读账号下执行。涉及结构变更或批量更新的语句属于另一类操作，必须连同回滚语句一起提交评审，由授权人员在数据库工具中落地。另外，直连库查询仅用于诊断定位，**业务取数应走平台接口**，避免解析 XML 与绕过权限留痕。
