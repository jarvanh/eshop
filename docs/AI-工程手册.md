# AI 工程手册（点餐 SaaS）

> 日期：2026-09-05
> 用途：AI 编码代理（Claude Code / Trae / Cursor 等）的**执行规则与技术规范**——每个 AI 会话必读、全程遵守
> 读者：AI（本文档只为 AI 而写）。人如何开启会话、如何派单，见《派单指南.md》
> 项目背景、决策与里程碑：见《点餐SaaS-详细实施计划.md》（本文档不重复项目内容，只定执行规则）

***

## 目录

- [1. 技术底座与实现途径](#1-技术底座与实现途径)

- [2. 工程约束](#2-工程约束)

- [3. 测试纪律](#3-测试纪律)

- [4. Git 仓库与提交纪律](#4-git-仓库与提交纪律)

- [5. 关键技术方案（实现规格）](#5-关键技术方案实现规格)

- [6. 数据库设计](#6-数据库设计)

- [7. CI/CD 与监控](#7-cicd-与监控)

- [8. 行为红线](#8-行为红线)

***

## 1. 技术底座与实现途径

**本项目以 ruoyi-vue-pro（master-jdk25 分支：JDK 25 + Spring Boot 4.x，MIT）为底座，位于 /workspace/ruoyi-vue-pro（M0 之后），不是从零开发。**

**四条铁律**：

1. 动手前必读文档 + 查该功能 ID 的「实现方式」列——**禁止重复实现底座已有能力，禁止从零造轮子**
2. 不确定的三方 API（微信/云打印/配送/定位）必须查官方文档或本地 SDK 源码，**禁止凭记忆编造参数**——查不到就停下来问人
3. 每个功能开发完成**立即测试**（单测 + 冒烟自检），测试通过才算完成、才能提交——**禁止先写完回头再测，禁止积压测试**
4. 测试通过后**立即 commit 并立即 push**——**禁止攒代码不提交、提交了不推送**

**实现途径（按功能清单「实现方式」列执行）**：

| 途径         | 适用                                                       | 规则                                                                        |
| ---------- | -------------------------------------------------------- | ------------------------------------------------------------------------- |
| ① 芋道直供     | 底座已有（system/infra/pay/member/product/promotion/trade/mp） | 直接用或轻扩展，**禁止重复实现**（支付、RBAC、多租户、优惠券基础等）                                    |
| ② yshop 移植 | 底座无、yshop 有（MIT 已核验）                                     | 从 /workspace/reference/yshop-drink-full 移植；保留原版权声明，每个移植文件在 NOTICE.md 登记出处 |
| ③ 自研       | 底座和 yshop 均无（收银台/桌台/协同/预约/骑手/提现等）                        | 参考项目仅限设计层面；可拷代码的只有 MIT 项目                                                 |
| ④ 三方 SDK   | 支付/打印/配送/定位/地图                                           | 必须查官方文档或本地 SDK 源码后使用                                                      |

**License 只读红线**：fuint 系（AGPL/无授权）、Jeepay（LGPL）、Floreant（MRPL）**禁止拷贝任何代码**，只可看设计。依赖注意：mysql-connector-j 为 GPLv2+FOSS Exception（SaaS 服务端可用；若做私有化交付需换 MariaDB 驱动）。

***

## 2. 工程约束

### 2.1 模块与代码规范

- 分层跟随芋道（controller/service/dal/convert），自建模块同构；MapStruct 转换

- 新功能只写在自建模块（yudao-module-store / pos / delivery / finance / saas），**不修改芋道上游文件**；确需改上游必须走扩展点（Spring 事件/策略替换/@ConditionalOnProperty），并在 UPSTREAM\_PATCHES.md 登记

- 先读已完成模块的代码，对齐风格后再动手

### 2.2 数据库规范

- 新表放 `sql/custom/` 按编号脚本，与上游 SQL 分离

- 金额一律 int 分；业务表带 tenant\_id/deleted/审计字段

***

## 3. 测试纪律

**及时测试新功能（强制）**：

- **完成即测试**：每个功能 ID 开发完成立即测试——单元测试 + 接口/页面冒烟自检；测试通过才允许 commit，跟踪表状态才能改「已完成」

- **不积压**：禁止"先全部写完再统一测试"——测试与开发同步进行，不攒到里程碑末尾集中补

- **提交前必跑**：commit 前跑通本模块相关测试；push 后确认 CI 绿，CI 红灯当天修复

- **核心域单测硬性要求**：支付幂等、优惠管道、协同购物车、提现状态机四个核心域必须带单元测试（核心逻辑覆盖率 ≥70%）

- **三方对接必须实测**：支付/云打印/配送/定位等功能，上线前必须在沙箱或测试环境真实链路验证，不允许只靠单测 mock 通过

***

## 4. Git 仓库与提交纪律

### 4.1 远程配置（M0/X-03 建立）

| remote     | 指向                      | 权限                        |
| ---------- | ----------------------- | ------------------------- |
| `origin`   | **GitHub 私有仓库**         | 读写（所有工作推送目标）              |
| `upstream` | ruoyi-vue-pro 官方（gitee） | 只 fetch/merge，**永不 push** |

### 4.2 分支模型

`main`（稳定，设分支保护）+ `feature/<ID或里程碑>-<简述>`；一个任务单一个分支；合并 main 由人操作（或明确授权后 AI 操作）。

### 4.3 提交规范

Conventional Commits + 功能 ID，例：`feat(store): A-05 桌号管理-批量导入与二维码导出`；type ∈ feat/fix/refactor/docs/test/chore。

### 4.4 提交与推送节奏（铁律）

- 每完成一个功能 ID **立即 commit**；每次 commit 后**立即 push 到 origin**；半天至少一次 commit

- push 后确认 CI 通过（红灯当天修复）

- 会话/工作日收尾前必须 push，工作区保持干净（无未跟踪代码文件）

- 绝不允许攒一大堆改动不提交就结束任务或会话

### 4.5 更新日志（CHANGELOG.md）

- 仓库根维护 `CHANGELOG.md`（Keep a Changelog 格式：Unreleased / 版本+日期 / Added·Changed·Fixed·Removed 分类），与代码同仓库、同推送

- 每完成一个功能 ID，**在同一个 commit 内**向 `Unreleased` 段追加条目：分类 + 功能 ID + 一句话说明（例：`- Added: A-05 桌号管理（批量导入/二维码导出）`）

- Bug 修复进 `Fixed`，返工/行为变化进 `Changed`——与 commit type 对应

- 里程碑验收通过后，将 `Unreleased` 归档为版本号 + 日期，再新开空的 `Unreleased`

- **禁止事后补写、批量补记**——changelog 条目与 commit 一一同步，缺条目视为任务未闭环

### 4.6 禁止事项

force push · 修改 git config · 提交 `application-local.yaml` / `*.env` / 密钥证书 / `node_modules/` / `dist/` / `reference/` / `.uploads/`（均已或应加入 .gitignore）· commit 前必须 `git status` 复查 · main 上的合并用普通 merge

### 4.7 上游同步

每月 merge `upstream/master-jdk25` 一次（三条纪律见计划 §4.4：新功能开新模块/改上游走扩展点/维护 UPSTREAM\_PATCHES.md）。

***

## 5. 关键技术方案（实现规格）

### 5.1 多人协同点餐（服务端权威）

```jsonc
// 客户端意图
{ "action": "add_item", "skuId": 1001, "qty": 2,
  "opId": "uuid", "sessionId": "S123", "version": 42 }
// 服务端广播
{ "type": "cart_snapshot", "version": 43,
  "items": [...], "members": [...] }
// 409 冲突 → 拉全量快照 → 重放未提交操作
```

服务端唯一事实源；`cart.version` 乐观锁 + `opId` 幂等（5min 窗口）；多实例 Redis Pub/Sub 广播；会话生命周期与订单域事件解耦。

### 5.2 优惠计算管道 + 快照

会员价→券→积分→储值，管道式计算；优惠快照（用什么/抵多少/行级分摊）整体序列化入订单；规则修改不影响历史与退款。

### 5.3 挂单/取单与云打印

- 挂单：pos\_order 状态 `parked` + 挂单区队列；取单恢复编辑；超时提醒（工作台红点）

- 打印：下单→RocketMQ→打印消费者→print\_task 状态机→指数退避×5→人工补打；后厨分单按分类路由、故障降级备用机

### 5.4 配送调度抽象（D9 双轨）

```
DeliveryDispatcher 接口
 ├── ThirdPartyAdapter   UU跑腿/达达/蜂鸟（下单/取消/状态回调/运费查询）
 └── SelfRiderService    自有骑手（呼叫/抢单/派单/状态流）（P2）
```

外卖单创建时按门店配置路由配送渠道；状态回调统一归一到订单配送状态机。

### 5.5 提现与资金合规

- 订单资金：普通商户模式直接进商家账户（平台不碰）——无二清

- 商家/骑手提现：属平台代发场景（平台营销补贴、骑手佣金），走**微信商家转账 API**（需另立主体与合同），提现申请-审核-打款-对账四步留痕

- 储值预付卡：属地单用途预付卡规定调研后决定开通范围

### 5.6 库存与对账

Redis Lua 预扣 → DB 乐观锁兜底 → 超时回滚；每日拉微信/支付宝账单 ↔ 本地流水逐笔核对，差异人工队列。

***

## 6. 数据库设计

### 6.1 核心表

```
-- 门店与桌台（store）
store / store_table / table_category(桌面分类)
table_session / table_session_member / cart_item
reserve_rule / reserve_tag / reserve_slot / reserve_order(预约订单)

-- 商品餐饮化
product_ext（估清/时段价/后厨分类）

-- 订单
order_dine（dine_mode: dine_in/pickup/delivery/reserve/cashier）
order_park(挂单暂存) / store_order 并单关系

-- 收银与打印（pos）
pos_order（含挂单状态）/ printer / print_task

-- 营销
member_card_def(卡片) / member_card_user(购买记录)
score_*（从 yshop 移植：score_product/score_order/运费模板三表）

-- 配送（delivery，P2）
rider / rider_apply / rider_account / delivery_order / delivery_fee_rule

-- 财务（finance，P2）
finance_bill(收支明细) / withdraw_account(提现账户)
withdraw_order(提现单：状态机 申请→审核→打款→到账) / withdraw_flow

-- SaaS
saas_plan / tenant_subscription / usage_daily
```

### 6.2 yshop 移植表对照

| yshop 表                                      | 用途         | 处理                                           |
| -------------------------------------------- | ---------- | -------------------------------------------- |
| yshop\_store\_product\_attr\*                | 规格三表       | 重命名对齐芋道 product 域后移植                         |
| yshop\_coupon / coupon\_user                 | 优惠券        | 与芋道 promotion coupon 二选一（推荐芋道为主，yshop 补餐饮字段） |
| yshop\_score\_\*                             | 积分商城       | 整体移植改前缀                                      |
| yshop\_express                               | 快递公司       | 移植 + 快递100 接口                                |
| yshop\_material(\_group) / shopads / service | 素材/广告/服务菜单 | 移植（素材库可评估直接用芋道 infra file）                   |
| yshop\_user\_bill                            | 用户账单       | 参考设计，账单体系并入 finance\_bill                    |

***

## 7. CI/CD 与监控

- push 触发 CI：构建 + 单测 + license 扫描 + 镜像；迁移脚本随流水线（先迁移后部署）

- 监控：Actuator+Prometheus+Grafana；业务告警：回调失败率/打印失败率/对账差异/断连率；多实例后集中日志（Loki/ELK）

***

## 8. 行为红线

**AI 行为红线**：

- 动手前查「实现方式」列，按途径执行（芋道复用/yshop 移植/自研）——禁止从零造轮子

- 不确定的三方 API 必须查官方文档或本地 SDK 源码——禁止编造参数

- 完成每个任务的闭环链：**测试通过（单测+冒烟）→ 立即 commit（含 CHANGELOG.md 条目）→ 立即 push → CI 绿 → 更新跟踪表**，缺一不可；禁止"先写完回头再测"

- **敏感域完成后必须停下等待人工复核**（支付回调/退款、提现打款、数据库迁移脚本、租户越权测试、合并 main）——完整职责表见计划 §6.4，复核通过前不得继续下游任务

**会话收尾三件套**（AI 必须输出，供人对账）：

1. 完成的 ID 清单（含测试结果）
2. commit+push 记录（git log 摘要 + CI 状态全绿确认）
3. 下一步建议

