# AI 派单模板

> 版本：v1.0　日期：2026-09-05
> 用途：向 AI（Claude Code / Trae / Cursor 等）派发开发任务的标准提示词模板
> 配套：《点餐SaaS-详细实施计划.md》《功能清单-进度跟踪.xlsx》《开源授权核验报告.md》

***

## 0. 快速上手（三步）

1. **开新会话** → 贴「§1 会话启动提示词」+ 本会话要做的任务 ID 清单
2. **派单个任务** → 从「§2 任务模板」复制对应类型，填入 ID 和要求
3. **会话收尾** → 让 AI 输出「完成 ID 清单 + commit 记录 + 下一步建议」，与跟踪表对账

***

## 1. 会话启动提示词（每个新会话开头贴一次）

```
先阅读以下文档建立上下文，读完后用 5 句话以内简述你理解的本次任务范围，等我确认再动手：
1. docs/点餐SaaS-详细实施计划.md —— 架构、ADR 决策（D1-D9）、里程碑 M0-M8
2. docs/功能清单-进度跟踪.xlsx「使用说明」+「功能清单」sheet —— 字段定义、状态流转、红线
3. docs/开源授权核验报告.md —— License 红线

【技术底座与实现途径（重要，防止你从零造轮子）】
- 本项目以 ruoyi-vue-pro（master-jdk25 分支：JDK 25 + Spring Boot 4.x，MIT）为底座，
  位于 /workspace/ruoyi-vue-pro（M0 之后）；不是从零开发
- 每个功能动手前，先看功能清单该 ID 的「实现方式」列，按途径执行：
  ① 芋道直供：底座已有（system/infra/pay/member/product/promotion/trade/mp）——直接用或轻扩展，
    禁止重复实现底座已有的能力（如支付、RBAC、多租户、优惠券基础）
  ② yshop 移植：从 /workspace/reference/yshop-drink-full 移植（MIT 已核验）——
    保留原版权声明，每个移植文件在 NOTICE.md 登记出处
  ③ 自研：底座和 yshop 均无（收银台/桌台/协同/预约/骑手/提现等）——
    参考项目仅限设计层面，可拷代码的只有 MIT 项目
- fuint 系（AGPL/无授权）、Jeepay（LGPL）、Floreant（MRPL）是只读红线，禁止拷贝任何代码
- 不确定的三方 API（微信/云打印/配送/定位）必须先查官方文档或本地 SDK 源码，
  禁止凭记忆编造参数——查不到就停下来问我

【全局工程约束】
- 新功能只写在自建模块（yudao-module-store / pos / delivery / finance / saas），
  不修改芋道上游文件；确需改上游必须走扩展点（Spring 事件/策略替换/@ConditionalOnProperty），
  并在 UPSTREAM_PATCHES.md 登记
- 数据库：新表放 sql/custom/ 按编号脚本；金额一律 int 分；业务表带 tenant_id
- 测试纪律（及时测试新功能）：每个功能 ID 开发完成【立即测试】——单元测试 + 接口/页面冒烟自检；
  测试通过才允许 commit、跟踪表才能改「已完成」；禁止"先写完回头再测"、禁止积压到里程碑末尾
- 核心域（支付幂等/优惠管道/协同域/提现域）必须带单元测试，核心逻辑覆盖率 ≥70%；
  支付/云打印/配送/定位等三方对接，上线前必须沙箱/测试环境真实链路验证，不允许只靠 mock
- 完成每个任务后：更新功能清单-进度跟踪.xlsx 对应 ID 状态（未开始→进行中→已完成）

【Git 提交与推送纪律（强制）】
- 远程仓库：origin = GitHub 私有仓库；upstream = ruoyi-vue-pro 官方（只 fetch/merge，永不 push）
- 分支模型：main（稳定）+ feature/<ID 或里程碑>-<简述>；一个任务单一个分支，完成后合回 main
- 提交时机：每完成一个功能 ID 立即 commit；半天至少一次 commit——
  绝不允许攒一大堆改动不提交就结束任务或会话
- 提交规范（Conventional Commits + 功能 ID）：
  feat(store): A-05 桌号管理-批量导入与二维码导出
  fix(pos): A-03 收银台挂单后余额支付金额翻倍
  type ∈ feat/fix/refactor/docs/test/chore
- 推送时机：每次 commit 后立即 push 到 origin；push 后确认 CI 通过（红灯当天修复）；
  会话/工作日收尾前必须 push，工作区保持干净（git status 无未跟踪的代码文件）
- 禁止提交：application-local.yaml、*.env、密钥证书、node_modules/、dist/、
  reference/（已在 .gitignore）；commit 前先 git status 复查
- 不要动 git config；不要 force push；main 上的合并用普通 merge
```

***

## 2. 任务派单模板（按类型复制）

### ① 底座/环境类（M0 专用）

```
执行计划 §5.1 M0 任务「jdk25 决策门」：
1. clone 芋道 ruoyi-vue-pro master-jdk25 分支到 /workspace/ruoyi-vue-pro
   （若已 clone 则 fetch 最新）
2. git remote 设置：origin 指向我的私有仓库 <你的GitHub地址>，upstream 指向官方
3. 执行 mvn clean package -DskipTests，报告构建结果（成功/失败 + 关键报错）
4. 扫描官方仓库 issue：pay/mall/mp 模块 SB4 相关报错是否堆积、有无修复
验收：结论（绿灯/红灯）+ 证据写入 docs/ADR/jdk25-decision.md；
首个 commit 推送到 origin main。
```

### ② 单功能开发（最常用）

```
实现功能清单 A-05「桌号管理」：
- 途径：自研（若清单该 ID 为「yshop移植/芋道」，先去对应底座或
  reference/yshop-drink-full 找现成实现，能复用不重写）
- 范围：REST API + 管理端页面（门店端角色菜单一并配好，参考 S-04）
- 前置阅读：计划 §5.2 M1 任务 2、§7.2 store 表设计；先读已完成的 A-07 门店管理代码对齐风格
- 要求：桌码防伪签名（tableId+HMAC+过期时间）；CSV 批量导入；
  二维码批量导出 PDF（每页 N 张，含桌号/门店名/落地页 URL）
- 完成动作：测试通过（单测 + 按验收标准逐条冒烟自检）→ commit（feat(store): A-05 ...）→ push →
  跟踪表 A-05 状态改「已完成」+ 备注
验收（你自评时逐条对照）：一门店建 10 桌；批量导入 20 桌成功、
重复桌号导入报错清晰；导出 PDF 打开正常；伪造桌码被拒绝。
```

### ③ 批量移植类（yshop → 主仓库）

```
按计划 §7.3 移植 yshop 规格三表（功能 A-14）：
- 源：/workspace/reference/yshop-drink-full/yshop-drink-boot3（MIT 已核验）
- 目标：对齐芋道 product 域命名规范，放 yudao-module-store-biz
- 纪律：保留每个移植文件的原版权声明；NOTICE.md 逐文件登记
- 餐饮化扩展：估清/时段价/做法类非库存规格（计划 §5.2 任务 3，BiteGo priceItemKey 思路）
- 完成动作：集成测试（多规格 SKU 笛卡尔积生成用例）通过 →
  commit → push → 跟踪表更新
验收：测试绿 + NOTICE.md 更新 + 移植清单（源文件→目标文件对照表）写进 commit message。
```

### ④ Bug 修复类

```
缺陷：[现象 + 复现步骤 + 期望 vs 实际]
先定位根因（给出证据：文件/行号/日志），修复遵守全局约束（不改上游）。
完成动作：防回归测试 → commit（fix(scope): ID 描述）→ push →
跟踪表对应 ID 备注「YYYY-MM-DD 返工：原因」。
不要顺手重构无关代码。
```

### ⑤ 里程碑验收类

```
对照计划 §5.3 M2 验收标准，逐项自检并输出验收报告：
每项给：通过/不通过 + 证据（测试名/截图路径/日志片段）。
不通过的列出修复建议和受影响的功能 ID。
不要降低标准自行判过；报告存 docs/验收/M2-验收报告.md 并 commit + push。
```

***

## 3. 敏感域人工红线（这些不要完全交给 AI）

| 域               | 原因             | 做法                                   |
| --------------- | -------------- | ------------------------------------ |
| 支付回调/退款         | 资金安全，AI 幻觉代价极高 | AI 写完后你逐行审 + 微信沙箱全流程测                |
| 提现/资金（X-08）     | 涉及合规           | 架构 AI 出，打款逻辑人工复核                     |
| 数据库迁移脚本         | 破坏性操作          | AI 生成，你审查后手动执行                       |
| 租户越权测试（M8）      | 安全是攻防思维        | AI 生成用例，验收自己过                        |
| git push 到 main | 保护稳定线          | AI 在 feature 分支工作，合 main 由你操作（或明确授权） |

***

## 4. 会话节奏

- **一会话一里程碑**：M0 一个会话，M1 拆 3-4 个会话；上下文快满果断开新会话（上下文都在文档里，损失很小）

- **会话收尾固定动作**：让 AI 输出三件套——①完成的 ID 清单（含测试结果）②commit+push 记录（git log 摘要 + CI 全绿确认）③下一步建议；你与跟踪表对账，不符即纠正

- **周例会**：过一遍跟踪表「进行中」项 + git log 周报（关注是否有超 1 天未推送的改动）

- **里程碑结束**：跑一次「⑤ 验收模板」+ 批量对账跟踪表

***

## 附：派单前自查清单（派出去之前 30 秒过一遍）

- [ ] 任务有明确的功能 ID（对应跟踪表一行）

- [ ] 说清了途径（芋道复用 / yshop 移植 / 自研）

- [ ] 有可判定的验收标准（能打勾的，不是"做好"）

- [ ] 说了测试要求（完成即测试，测试不过不提交）

- [ ] 说了 commit + push 要求

- [ ] 敏感域任务已标注"完成后等我复核"

