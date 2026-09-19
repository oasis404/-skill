2026-09-19 工作日志
EAM 资产责任人变更模块 · 表结构设计评审（只评审，未改文件）
输入：需求脑图（资产变更：现状问题 + 未来四类变更闭环）+ 设计表 EAM_Asset_Owner_Change / EAM_Asset（SQL Server 风格类型）。

需求闭环：领用 / 调拨 / 借出 / 归还 —— OA 提交申请 → 资产管理员审核 → 系统派单通知财务 → 系统月末回写资产清单（责任人及对应部门）；借出需置资产状态为"已借出"且可查借出明细，归还恢复为"正常"。

P0 结论

变更表 Update_time 与 update_time、Update_by 与 update_by 同表并存 → SQL Server 默认 CI 排序规则下为同一列名，建表报错 2705。建议改名 change_by / change_time。
变更表只有 owner_old_account / owner_new_account，缺 new_user_id / new_user_name / new_dept / new_org 快照；而 EAM_Asset 回写目标是 resp_user_id/account/name + resp_dept + resp_org 五列 → 月末回写断链。
需求"派单通知财务"全表无落点，缺 finance_notify_status/time、finance_ticket_no、fail_reason；另缺 oa_instance_id。
P1 结论
4. 借出语义冲突：借出期间责任人应仍是原责任人，实际使用人是借用人。建议 EAM_Asset 增 current_holder_id/account/name，与 resp_user_* 分离；借出改 holder，归还清 holder，责任人只在领用/调拨时变更。
5. "查看借出明细"无处安放 → 建议独立表 EAM_Asset_Borrow。
6. 归还的"归还对象"缺 return_target；变更原因缺 change_reason。
7. 月末批处理无幂等控制 → 补 effect_status / effect_time / sync_batch_no。
8. 审批只有 Approve_By 姓名，缺 approver_id；所有 *_by 应 id + name 并存。
9. 同一资产并发变更无防重（effect_status=0 时应只允许一条）。
10. EAM_Asset.asset_no 缺唯一约束；is_lent 与 status 双写不一致风险。

P2 规范：Change_no 大小写不一致；SQL Server 无 Boolean（应用 bit）；is_delete 缺默认值；datetime 建议 datetime2(0)；Approve_Status 与 Approve_Status_Name 冗余需单一维护；变更表 id 未标 IDENTITY。

待用户确认三问
A. 一张 OA 单能否含多台资产（决定是否需主表 + 明细表拆分）。
B. 审批单级还是两级（部门负责人 → 资产管理员）。
C. 月末批量回写 vs 审批通过即生效（月结会造成最多 30 天账实差异）。

交付形式：inline SVG 覆盖度矩阵 + 字段来源缺口图 + 建议 SQL（EAM_Asset 增量 ALTER、变更表重建、EAM_Asset_Borrow 新建）。用户偏好"只教不改"，未写任何项目文件。

设计修正（同日，用户质疑后）
用户提问：「借出明细 不是每次更改的 id 和单号都不同 可以一直留档吗」——质疑为何要独立建 EAM_Asset_Borrow 表。

修正结论：用户判断成立，撤回"独立借出明细表"的建议。 变更单本身就是流水表，一条变更一行，id / change_no 天然唯一，留档能力已经具备，靠新建表来做"留档"是多余设计。

真正的缺口只有一处：借出单与归还单的配对关系。 同一资产多次借出归还时（如借出 3 次归还 2 次），归还单无法自证"还的是哪一次借出"。

最终方案（最小改动，不新增表）

变更表加一列 parent_change_id BIGINT NULL：归还单指向借出单的 id，其他类型为 NULL。
配对查询从"按时间推导紧邻前一条借出"（脆弱、每次重算）变成 LEFT JOIN ... ON r.parent_change_id = b.id（一次 JOIN）。
"当前是否在借"不查变更单，直接查 EAM_Asset WHERE is_lent = 1，配合已建议的 current_holder_* 字段。
归还单配对可自动化：因已约定"同一 asset_id 在 effect_status=0 时只允许一条有效单据"，系统可自动找出该资产当前唯一在借的借出单写入 parent_change_id，OA 侧无需手填。
仅在以下情况才需要独立明细表：一次借出多台资产分批归还 / 支持部分归还 / 借出物不是整台资产（耗材） / 需要独立借出单号体系。本场景为单台资产整借整还，不适用。

方法论沉淀：评审时区分"留档"与"状态"两件事——流水表天然满足留档，但事件的生命周期（在借/已归还）、配对关系往往需要额外字段，而不必新建表。用户对"为留档而建表"很敏感，后续评审优先考虑自关联字段方案。

约束确认与定稿交付（同日）
用户确认：「都是一单限制传一台」——一张 OA 单只对应一台资产。此前待确认问题 A 关闭，不需要主表加明细表拆分，单表结构（一单一资产）成立。

该约束的连带影响（已写入交付脚本）

变更表加 parent_change_id 自关联即可完成借出归还配对，无需独立借出表。
同资产并发风险上升，把"同一资产在途单据互斥"从建议升级为数据库约束：过滤唯一索引 UX_AOC_pending_asset（WHERE is_delete=0 AND effect_status=0 AND approve_status IN ('1','2')）。
change_no 唯一约束是必须项，用于兜底 OA 接口重试导致的重复推送（重推若不拦截会插入重复单据，月末回写执行两次）。
需求文档措辞应从"资产编号（可能复数）"统一改为"资产编号（单台）"，避免开发误以为需要支持数组。
待约定：OA 一次选多台时由 OA 侧拆单，还是推给 EAM 侧拆——接口约定需写明。
交付物：C:\Users\Chg10\Desktop\EAM_资产变更表结构_定稿.sql（354 行，UTF-8 with BOM，18.1 KB）。含四段：主表增量补丁（幂等，COL_LENGTH 保护）、变更表重建（有数据时跳过不删）、变更表索引（含过滤唯一索引）、建表后自检查询。附录含值域约定、月末回写口径、借出明细查询 SQL。

技术要点沉淀（可复用）

SQL Server 中仅大小写不同的列名（Update_time / update_time）在默认 CI 排序规则下属同一列名，建表报错 2705。审计表设计时优先检查这类重名列，自检 SQL 用 GROUP BY LOWER(name) HAVING COUNT(*)>1。
幂等 DDL 模板：加列用 IF COL_LENGTH(...) IS NULL，加默认约束用 sys.default_constraints 判存，加索引用 sys.indexes 判存，唯一索引创建用 TRY/CATCH 兜重复数据。
给 SSMS / Navicat 的中文注释 .sql 文件必须写 UTF-8 with BOM，否则按系统 ANSI(GBK) 解析会中文乱码。
本机 PowerShell 工具不返回 stdout，需要把诊断结果写入文件再用 Read 读取；Bash 工具环境损坏（dirname/ls 缺失），路径含中文时优先走 Python + PowerShell。
Glob 工具不扫描工作区之外的路径（如桌面），验证外部文件需用 PowerShell。
配对字段方案修正 v1.1（同日）
用户提问：「不能通过别的方式将归还单指向借出单吗，这个有 4 个类型，不是每种都需要指向的」——指出 parent_change_id 对 3/4 类型恒为 NULL，属于稀疏字段。

修正结论：改用 borrow_change_no NVARCHAR(64)，存借出单的 change_no 而非自增 id。

三种方案对比（已向用户呈现）：

A 自关联 id（parent_change_id）：整型关联快，但字段名不达意，且把数据库自增 id 暴露到业务单据、跨系统对账困难。
B 存业务单号（borrow_change_no，采纳）：语义直白、与 OA 单号体系天然对齐、人工与财务可直接阅读。
C 独立关联表：无空列最"干净"，但只有一种关系却建表属过度设计。
采纳 B 的决定性理由：脑图明确"变更单号 OA 生成"，即所有单号 OA 自己都有。归还申请时 OA 直接带出借出单号传字符串即可，无需先反查 EAM 拿自增 id。原则：数据库自增 id 不应跨系统暴露，跨单据引用一律用业务单号。

同时新增两个约束（写入脚本）

UX_AOC_borrow_no 过滤唯一索引：同一借出单只能被归还一次（驳回单 approve_status<>'4' 不占坑）。
CK_AOC_borrow_no CHECK：borrow_change_no IS NULL OR order_type = 4，防止误填到非归还类型。
重要认知（回应"字段稀疏"质疑）：变更表本身就是 4 类变更共用的宽表，expect_return_date（仅借出）、actual_return_date / return_target（仅归还）、new_*（仅前 3 类）都是同样的稀疏字段。留空是该设计的固有特征，不是缺陷。已在脚本附录 A 补"字段适用类型对照表"（字段组 × 4 类型矩阵），开发按表校验即可。

交付物更新：C:\Users\Chg10\Desktop\EAM_资产变更表结构_定稿.sql（v1.1，423 行，22.9 KB，UTF-8 with BOM）。新增附录 D 说明借出单号的两个写入时机（OA 带入 / 系统自动配对），附录 E 为借出明细查询。

上下文补充：公司 OA 为蓝凌 OA（同日）
用户查询「蓝凌oa系统蓝小佳手册」，据此确认公司 OA 平台为蓝凌（Landray）OA，即 EAM 资产变更单的产出方与推送方。这对后续集成方案有直接影响。

查证结论（蓝凌三个"蓝小X"服务品牌，勿混淆）

蓝小悦：蓝凌官方在线服务品牌，客服/运维，含 7x24 AI 智能客服，通过"蓝凌客户空间"小程序联系。
蓝小佳：订阅类增值服务产品（资源集市），在集市中订阅现成资源包，无需额外实施、即选即用。标准版首年免费含高频场景精选资源包；尊享版按需购买，含资源组合与个性化专家连线。注意：蓝小佳不是 AI 助手，蓝凌的 AI 助手是 MK Claw / 智能体平台。
蓝小凌：销售咨询（19928791324）。
EAM 与蓝凌 OA 集成的公开技术线索（待用户确认方向后再深化）

蓝凌 OA 产品线为 EKP（如 EKP V16.0），基于 Spring + Struts + Hibernate/MyBatis，早期前端 jQuery/Dojo，后期引入 Vue。
老版本提供 WebService，新版本提供 REST。启动审批流程的接口为
http://[IP]:[PORT]/sys/webservice/kmReviewWebserviceService?wsdl
方法 addReview(KmReviewParamterForm webForm)，关键参数：docSubject（标题）、fdTemplateId（模板id）、formValues（表单数据 JSON，明细表格式为 "明细表id.列id":["值1","值2"]）、docStatus（"10"草稿 / "20"待审，默认20）、docCreator（发起人 JSON）、attachmentForms（附件，base64）。
表单原始数据以 XML 存于 km_review_main 表的 extend_data_xml 字段。
官方集成工具为 TIC 第三方集成中心（服务配置 → 函数配置 → 转换适配 → 流程模板 → 映射关系，支持表单控件/机器人节点/审批节点事件三种映射方式）。
公开可获取的文档：《EKPV16.0用户手册-TIC第三方集成中心》106页、《蓝凌智能OA快速入门手册》53页（均在 book118，付费）；完整开发手册为蓝凌官方交付给客户的内部资料，需向实施顾问或客服索取。
待确认：用户具体需要哪一类手册（EAM 集成开发 / 蓝小佳订阅使用 / 用户操作 / 表单与流程配置），已发起询问。

交付：蓝凌 EKP 新建流程操作手册（同日）
用户确认需求为**「如何创建新的流程」**，即管理员视角在蓝凌 OA 里新建流程。

已交付：C:\Users\Chg10\Desktop\蓝凌OA新建流程操作手册.html（35.7 KB，UTF-8 with BOM，单文件浅色可打印，按用户讲义规范：内联 SVG + VS Code 暗色代码高亮 + 蓝/绿/粉/米四色 + details/summary 答疑）。

手册核心结论（蓝凌 EKP 流程配置的结构，来自官方管理员手册目录）

流程与表单分开配置，最后绑定到"业务类别"，这是新手最易卡住的认知点。类别决定入口位置与发起权限。
六步主线：建分类 → 新建流程模板 → 设计节点与审批人 → 流程检测 → 设计表单模板 → 绑定并发布。
流程模板 vs 通用流程模板：后者可被多个分类共用（判断标准是"会不会有第二个分类也用"）。
两个关键流程选项开关：「驳回的节点通过后直接返回本节点」「重新流转节点时重新计算节点处理人」。
审批人指定方式优先级：公式（动态计算）> 角色 > 部门/岗位 >> 指定具体人员（写死人会导致调岗离职后流程卡死）。
节点类型共十余种，资产变更实际只需 6 种：开始、审批、条件分支、机器人、抄送、结束。
表单数据默认以 XML 存于 km_review_main.extend_data_xml；明细表按列存储。
通用编号规则是变更单号（如 CHG-20260310-001）的生成处。
机器人节点回写 EAM 靠 TIC 第三方集成中心，五步：服务配置 → 函数配置 → 数据查询验证 → 转换适配 → 流程中引用（可挂在表单控件/机器人节点/审批节点事件三处）。
反向推单接口：http://[IP]:[PORT]/sys/webservice/kmReviewWebserviceService?wsdl，方法 addReview(KmReviewParamterForm)；参数 docSubject / fdTemplateId / formValues / docStatus / docCreator / attachmentForms。注意附件走 base64，fdTemplateId 测试与生产不同值不可写死。
表单控件中**「数据填充控件」对本项目最有用**：输入资产编号自动带出资产名称、当前责任人等，正是 EAM 表结构里"快照字段"的数据来源。需配合 TIC 函数配置。
对资产变更四类流程的配置建议（已在手册中给出）

推荐方案一：单流程 + 条件分支。一个流程模板承载四类变更，表单放"变更类型"单选控件，审批后由条件分支节点按 fd_order_type 分四路，各路接不同机器人节点。理由：四类审批人相同、字段重合度高；新增类型只需加分支。
方案二（四套独立流程）字段与编号规则可独立，但重复配置、维护成本翻四倍。
四个分支不可合并：调拨/领用改"责任人"，借出/归还改"当前持有人"，两组字段在 EAM 侧是分开的；合并会导致归还时误清责任人字段。
已提示的三条风险：控件 ID 联调后不可改（否则接口静默收不到数据）；流程模板未绑类别则用户看不到入口；不要直连 OA 库读 extend_data_xml，应走接口。
