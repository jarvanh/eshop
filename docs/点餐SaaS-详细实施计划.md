# 点餐 SaaS · 详细实施计划

> 版本：v1.0（定稿）
> 日期：2026-09-05
> 范围：从底座选型到上线加固的完整实施路线
> 状态：待评审

***

## 目录

- [1. 项目概述](#1-项目概述)

- [2. 总体架构](#2-总体架构)

- [3. 开源资源策略](#3-开源资源策略)

- [4. 里程碑计划](#4-里程碑计划)

- [5. 关键技术方案](#5-关键技术方案)

- [6. 数据库设计](#6-数据库设计)

- [7. 工程规范与基础设施](#7-工程规范与基础设施)

- [8. 团队分工与并行计划](#8-团队分工与并行计划)

- [9. 风险登记册](#9-风险登记册)

- [10. 上线后任务清单](#10-上线后任务清单)

***

## 1. 项目概述

### 1.1 商业模式与前提假设

| 项    | 假设                                            | 变更影响                 |
| ---- | --------------------------------------------- | -------------------- |
| 商业模式 | 多租户 SaaS，卖给商家，平台收订阅费                          | 若自用则可砍掉 M6 的 SaaS 计费 |
| 技术栈  | Java 25 + Spring Boot 4.1（芋道 master-jdk25 分支） | 详见 2.3 与 4.1 决策门     |
| 团队   | 4 人：后端 ×2、Web 前端 ×1、小程序 ×1                    | ≤2 人需重排里程碑优先级        |
| 部署   | Docker Compose 起步，单体模块化                       | 后期可按模块拆微服务           |
| 资金合规 | 支付走商家自有商户号（普通商户模式），平台不经手资金                    | 规避二清；若改服务商模式需重新评估    |

### 1.2 功能需求全景

| 模块 | 能力                           | 实现方式                                      |
| -- | ---------------------------- | ----------------------------------------- |
| 点餐 | 外卖 / 自取、提前预约、桌台扫码（单人 / 多人协同） | 堂食/自取/外卖 M2；预约 M2；协同 M5（自研核心）             |
| 商品 | 多规格 SKU、图片素材库                | 芋道 product + 餐饮化扩展（M1）                    |
| 门店 | 多门店、店铺管理、商家中心                | 芋道多租户 + 自研 store 模块（M1）                   |
| 营销 | 优惠券、积分兑换（积分+金额）、充值、会员卡       | 芋道 promotion/member/pay wallet + 组合改造（M4） |
| 收银 | 收银台（扫码枪 / 扫码盒子）、云小票打印        | 自研（M3），设计参考 Jeepay / Floreant             |
| 运营 | 订单管理、微信公众号、自定义装修、SaaS 多租户    | 芋道 trade/mp/diy + SaaS 计费自研（M6）           |

### 1.3 关键决策记录（ADR 摘要）

| #  | 决策                                                   | 理由                                                                                                                                                                                               |
| -- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| D1 | **底座用 ruoyi-vue-pro（单体完整版）**，而非从 0 或深度魔改 yshop-drink | mall/pay/member/mp/promotion/diy 覆盖需求 60%+；MIT 可商用；社区最大、文档最全；单体版 1 个 JAR + MySQL + Redis，4 人团队运维得起                                                                                               |
| D2 | **不用 yudao-cloud 微服务版**                              | 4 人团队养不起 Nacos/Gateway/Seata/XXL-Job 全家桶；单体 Maven 模块化设计，后期可拆                                                                                                                                     |
| D3 | **选 master-jdk25 分支（JDK 25 + SB 4.1）起步**，替代 jdk17    | SB 3.5 开源支持已于 2026-06-30 结束，升 4 不可避免；day-0 升级比 day-300（生产有真实支付流量时）便宜一个数量级；上游已正式适配 Jackson 3 / Security 7 / Spring Cloud 2025.1；MyBatis Plus 3.5.16+ 已修复 SB4 兼容。**M0 设半天体检决策门，红灯退回 master-jdk17** |
| D4 | **多人协同点餐自研**（服务端权威模型）                                | 所有国内开源餐饮项目均无此能力或实现粗糙；是产品差异化护城河                                                                                                                                                                   |
| D5 | **支付走普通商户模式**，租户各自收款                                 | 平台不经手资金，规避二清合规风险                                                                                                                                                                                 |
| D6 | **跟踪式 fork 芋道**，少碰上游文件                               | 持续白拿社区修复（安全补丁、微信 API 跟进）；详见 3.3                                                                                                                                                                  |

***

## 2. 总体架构

### 2.1 分层架构

```
┌────────────────────────────────────────────────────────────┐
│ 接入层                                                      │
│  顾客端 uniapp（小程序/H5） · 商家中心 Vue3 · 收银台 Vue3 ·   │
│  平台后台 Vue3                                              │
├────────────────────────────────────────────────────────────┤
│ 应用层（单体 yudao-server，1 个 JAR）                        │
│  ┌─────────────── 自研核心（产品差异化）───────────────┐      │
│  │ yudao-module-store   门店/桌台/桌台会话(协同)         │      │
│  │ yudao-module-pos     收银台/聚合支付B扫C/云打印       │      │
│  │ yudao-module-saas    套餐计费/配额/租户开通           │      │
│  └──────────────────────────────────────────────────┘      │
│  ┌─────────────── 底座复用（芋道现成）────────────────┐      │
│  │ system 多租户/RBAC · infra 代码生成/文件/OSS        │      │
│  │ product 商品SKU · promotion 券/装修 · trade 订单     │      │
│  │ member 会员/积分 · pay 支付/钱包 · mp 公众号        │      │
│  └──────────────────────────────────────────────────┘      │
├────────────────────────────────────────────────────────────┤
│ 基础设施层                                                  │
│  MySQL 8（主数据/订单）· Redis（缓存/会话/分布式锁）·        │
│  RocketMQ（事件驱动：打印/通知/对账）· OSS+CDN（素材）        │
├────────────────────────────────────────────────────────────┤
│ 外部服务                                                    │
│  微信生态（支付/小程序/公众号/开放平台第三方）·                │
│  云打印（易联云/飞鹅）· 短信                                 │
└────────────────────────────────────────────────────────────┘
```

### 2.2 底座能力映射（省力清单）

| 需求             | 芋道现成能力                         | 处理方式                     |
| -------------- | ------------------------------ | ------------------------ |
| 多规格 SKU、素材库    | product 模块 + infra file（OSS）   | 复用 + 餐饮化字段（估清、时段价、做法规格）  |
| 支付/退款          | pay 模块（微信/支付宝/钱包）              | 直接复用                     |
| 充值储值           | pay wallet（充值套餐、余额支付）          | 复用 + 本金/赠金分账扩展           |
| 积分             | member point + promotion point | 复用 + **积分+金额混合抵扣**自研     |
| 优惠券            | promotion coupon               | 复用 + 餐饮场景（首单券、配送减免、券互斥组） |
| 会员卡等级          | member level                   | 复用；**计次卡自研**             |
| 微信公众号          | mp 模块                          | 复用 + 第三方平台授权改造（M6）       |
| 自定义装修          | promotion diy 模块               | 复用 + 餐饮组件扩展（M6）          |
| 多租户/RBAC/审计    | framework                      | 直接复用                     |
| **门店/桌台/协同点餐** | 无                              | **自研（store 模块）**         |
| **收银台/云打印**    | 无                              | **自研（pos 模块）**           |
| 外卖/自取/预约时段     | 无                              | 自研（trade 扩展）             |
| SaaS 套餐计费      | 无                              | **自研（saas 模块）**          |

### 2.3 技术栈清单（定稿）

| 层    | 选型                                       | 说明                                                          |
| ---- | ---------------------------------------- | ----------------------------------------------------------- |
| 后端   | JDK 25 (LTS) + Spring Boot 4.1           | 芋道 master-jdk25 分支；虚拟线程默认开启                                 |
| ORM  | MyBatis Plus 3.5.16+                     | SB4 兼容修复版本                                                  |
| 缓存/锁 | Redis 7 + Redisson                       | <br />                                                      |
| 消息   | RocketMQ 5                               | 打印任务、订单事件、对账                                                |
| 数据库  | MySQL 8.0                                | 金额一律 int 分                                                  |
| 实时通信 | WebSocket（原生 STOMP 或裸 WS）                | 协同购物车；多实例时 Redis Pub/Sub 广播                                 |
| 管理端  | yudao-ui-admin-vue3（Vue3 + Element Plus） | 商家中心 + 平台后台共用底座                                             |
| 顾客端  | uniapp（自研）                               | 参考 yshop-drink-uniapp-vue3 与 yudao-mall-uniapp 的页面结构/API 封装 |
| 收银台  | Vue3 独立应用                                | 扫码枪 HID 键盘模拟                                                |
| 文档   | Knife4j（boot4 专用 starter 5.0.x）          | <br />                                                      |
| 部署   | Docker Compose → K8s（可选）                 | 单体 1 JAR + MySQL + Redis + RocketMQ                         |

### 2.4 部署形态

- **开发**：本地起 MySQL/Redis/RocketMQ（docker compose），`yudao-server` IDEA 直跑

- **测试/生产**：Nginx（前端静态 + 反代）+ app（可水平扩容，会话在 Redis）+ MySQL（主从）+ Redis（哨兵）+ RocketMQ

- **扩容路径**：单体多实例 → 热点模块（协同 WS、订单）拆独立服务 → 全量微服务化（仅在必要时）

***

## 3. 开源资源策略

### 3.1 参考项目矩阵

> 工作区已 clone：`yshop-drink-boot3`（后端）、`yshop-drink-vue3`（管理端）、`yshop-drink-uniapp-vue3`（顾客端）。
> `reference/` 目录已建（已加入 .gitignore）存放其余参考仓库，永不进主仓库。
> **所有项目 License 已于 2026-09-05 核验完毕，证据见** **[开源授权核验报告](开源授权核验报告.md)** **与** **`docs/license-evidence/`。**

| 项目                              | License                                        | 技术栈                    | 对口里程碑     | 用法                                                                                                          |
| ------------------------------- | ---------------------------------------------- | ---------------------- | --------- | ----------------------------------------------------------------------------------------------------------- |
| **ruoyi-vue-pro**（master-jdk25） | MIT                                            | Java25/SB4/Vue3        | 底座        | fork 跟踪（见 3.3）                                                                                              |
| **yshop-drink**（工作区已有）          | **MIT（已核验 LICENSE）**                           | Java17/SB3/Vue3/uniapp | M1/M2/M4  | **可移植代码（保留版权声明）**；优先抄：多门店模型、积分+金额兑换逻辑、充值/会员卡、装修组件、uniapp 页面结构                                               |
| fuintCatering                   | ⚠️ 无 LICENSE + 商用需购买授权；主仓 fuint 为 **AGPL-3.0** | SpringBoot/uniapp      | M3/M4     | **严格只读**；抄：收银台扫码枪交互、云打印对接、卡券核销（仅设计层面）                                                                       |
| **BiteGo 点点餐**                  | **MIT（已核验）**                                   | NestJS/MongoDB         | **M5/M1** | 法律上可移植（TS→Java 无法直接移植，实际以精读文档为主）：opId 幂等 + cart.version 乐观锁 + Redis Pub/Sub、非库存规格 priceItemKey、平台/租户/门店三层授权 |
| TKOB\_QROrderSystem             | MIT                                            | Node                   | M2/M5     | 整体架构对照；多租户 QR + KDS + 实时订单流                                                                                 |
| TastyIgniter                    | MIT                                            | PHP                    | M1/M2     | 产品设计对标：按桌容量的预约、餐段菜单可见性、门店配送区域+起送价、KDS 看板交互                                                                  |
| Floreant POS                    | MRPL 1.2                                       | Java Swing             | M3        | 只读参考（协议要求改后开源，禁止拷代码）；抄领域规则：并台/拆单/反结账授权/钱箱对班/打印机分组路由与故障降级                                                    |
| Jeepay 计全支付                     | LGPL-3.0                                       | Java/SB3               | M3        | 只读参考（LGPL 传染，禁止拷代码）；抄设计：B扫C、对账、回调 MQ+重试、服务商/普通商户双模式                                                         |
| FlashWaimai                     | MIT                                            | Java/Vue（5 年未更新）       | M2        | 概念参考：外卖三端订单流转                                                                                               |

### 3.2 License 合规矩阵（红线）

| 用途                                               | 允许的项目                                                                                       |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| 抄设计思想、表结构思路、交互流程                                 | 任何协议都安全                                                                                     |
| 拷贝代码进主仓库                                         | 仅 MIT（芋道、yshop-drink、BiteGo、TKOB、TastyIgniter、FlashWaimai）；**必须保留原版权声明**，在 `NOTICE.md` 登记来源 |
| LGPL（Jeepay）/ MRPL（Floreant）/ AGPL 及无授权（fuint 系） | 严格只读，禁止任何代码拷贝                                                                               |

> ⚠️ 依赖注意项：mysql-connector-j 为 GPLv2 + FOSS Exception——SaaS 服务端使用无风险；若未来做私有化部署交付，需换 MariaDB 驱动或重新评估。hutool 为 MulanPSL-2.0（宽松，保留声明）。

CI 中引入 license-maven-plugin 扫描依赖与源码；`NOTICE.md` 记录所有借用的 MIT 代码出处。

### 3.3 跟踪式 fork 策略（同步芋道更新）

**Git 结构**

```bash
git remote add upstream https://gitee.com/zhijiantianya/ruoyi-vue-pro.git
# 定期同步（跟住 master-jdk25 分支）
git fetch upstream && git merge upstream/master-jdk25
```

**三条纪律**

| 纪律        | 做法                                                                |
| --------- | ----------------------------------------------------------------- |
| 新功能开新模块   | 门店/桌台/收银/SaaS 全放 `yudao-module-store / pos / saas` 自建模块，不往上游模块塞代码 |
| 改上游行为用扩展点 | Spring 事件监听、策略 Bean 替换、`@ConditionalOnProperty` 开关，不直接编辑上游文件      |
| 维护改动清单    | `UPSTREAM_PATCHES.md` 记录"改了哪个上游文件、为什么、改了什么"，升级时按清单 reapply        |

**数据库分开管**：上游升级用上游增量 SQL；自建表（store、table\_session、print\_task 等）放 `sql/custom/` 独立目录 + 编号迁移脚本，不混进上游 `sql/`。

**分级同步预期**

- framework / system / infra → 持续同步（安全修复、代码生成器改进）

- mall / pay / member 基础能力 → 前 6-12 个月选择性 cherry-pick（微信 API 变更跟进）

- 深度餐饮化改造部分（trade 并单、promotion 优惠管道）→ 冻结不再同步

- 跨大版本升级 → 不走 merge，专项立项

***

## 4. 里程碑计划

> 阶段按依赖关系排序，不做日历排期；每阶段给出目标 / 任务 / 验收标准 / 参考。

### M0 · 底座就绪与合规审计

**目标**：干净的单体底座跑起来，长周期资质启动，选型决策门通过。

**任务**

1. **jdk25 决策门（半天，最先做）**

   - checkout master-jdk25 最新稳定 tag，`mvn clean package` 构建全绿

   - 扫描仓库 issue：jdk25/SB4 相关的 pay/mall/mp 报错不堆积、有修复响应

   - 冒烟：空租户 + 微信支付沙箱回调跑通

   - **绿灯 → jdk25 起步；红灯 → 退回 master-jdk17（JDK17+SB3.5），并在上线后第一低峰期执行升 4 专项**
2. 裁剪模块：保留 `system / infra / pay / member / mp / product / promotion / trade`，砍掉 BPM/CRM/ERP/报表/AI/IoT/MES/IM
3. License 审计：**已完成（2026-09-05，见** **[开源授权核验报告](开源授权核验报告.md)）**——芋道/yshop-drink/BiteGo/TKOB/TastyIgniter 均 MIT；fuint 系（AGPL + 无授权 + 商用收费）、Jeepay（LGPL）、Floreant（MRPL）划只读红线；剩余：license-maven-plugin 进 CI、建 NOTICE.md
4. 工程化：Git 分支策略、CI（构建+单测+镜像）、dev/prod docker-compose（MySQL/Redis/RocketMQ）、统一异常/校验/MapStruct 规范
5. **微信资质启动（第一天就办，周期长）**：微信开放平台账号 + 认证（目标：第三方平台代小程序）；先用自有小程序调试
6. 数据库规范定稿：金额 int 分、`tenant_id` 由芋道自动注入、软删/审计字段
7. 建 `reference/` 目录并 clone 参考仓库（BiteGo/TKOB/TastyIgniter/Floreant/Jeepay/fuint）

**验收**：新租户后台创建并数据隔离；demo 小程序微信授权登录跑通；CI 全绿；决策门结论落档（`docs/ADR/jdk25-decision.md`）。

### M1 · 门店与商品（商家中心 v1）

**目标**：商家能开店、建桌、上菜。

**任务**

1. `store` 模块：门店信息、营业时段（支持多时段/跨天）、外卖/自取开关、配送范围（圆形半径起步，参考 TastyIgniter 的多边形+起送价设计留扩展）
2. `store_table`：区域/桌号、桌码生成（tableId + 签名防伪）、批量导出 PDF 物料
3. 商品：多规格 SKU + 估清 + 时段价（参考 TastyIgniter 餐段可见性、BiteGo 非库存规格 priceItemKey 思路：辣度/做法类规格不占 SKU）
4. 素材库：挂 infra file（OSS），图片分组管理
5. 商家中心页面（yudao-ui-admin-vue3 二开）
6. 从 yshop-drink（MIT）移植可复用的门店/商品字段设计与部分实现

**验收**：一租户建 2 门店、每店 10 桌、30 个多规格菜品；桌码 PDF 可下载打印；估清后顾客端不可下单。

### M2 · 点餐闭环 + 支付（最小可上线版本）

**目标**：真机能完整走一单。

**任务**

1. uniapp 顾客端：扫桌码/门店码进入、菜单、购物车、下单（堂食/自取/外卖三模式）
2. 预约点餐：取餐时段 slots（营业时段切分 + 每时段容量上限）
3. 订单域：状态机（待支付→已支付→制作中→完成/取消）；**加菜 = 同桌子订单并单**（trade 扩展点实现，不改上游文件）
4. 支付：微信 JSAPI + 小程序支付，回调幂等（分布式锁 + 状态机防重）
5. 商家端订单管理：接单/拒单/完成/退款
6. 库存防超卖：Redis 预扣 + DB 乐观锁兜底（见 5.5）

**验收**：真机小程序走通 堂食下单→支付→接单→退款 全链路；并发 100 单不超卖；支付回调重放不影响订单状态。

### M3 · 收银台 + 云打印（可与 M2 后半段并行）

**目标**：店里能收款、能出票。

**任务**

1. 收银台 Web（Vue3 独立应用）：菜品检索（B 扫商品码）、开台/关联桌台、代客下单
2. 扫码枪/扫码盒：HID 键盘模拟（监听 keydown、回车截断），自动识别微信/支付宝付款码，走聚合支付 B 扫 C（设计参考 Jeepay：渠道路由、回调 MQ+重试）
3. 组合支付：现金 + 扫码（余额支付 M4 接入）；抹零、整单折扣
4. 云打印：**适配器模式**同时对接易联云 + 飞鹅（厂商 SDK 直连，不抄 Jeepay 代码）
5. 打印可靠性：打印任务表 + 状态机（pending/printed/failed）+ 指数退避重试（最多 5 次）+ 离线告警（见 5.3）
6. 小票模板：结账单 + 后厨分单（按菜品分类路由到不同打印机；打印机故障降级策略参考 Floreant：厨房打印机故障自动转前台）

**验收**：扫码盒实测收款到账；下单后 3 秒内出票；模拟打印机离线有告警、恢复后自动补打；两台后厨打印机按菜品分类正确分单。

### M4 · 营销体系

**目标**：一单能组合多种优惠且账算得清。

**任务**

1. 优惠券：券互斥组、适用门店/品类范围（复用 promotion coupon + 扩展）
2. 积分：消费得分（含会员加速）、**积分+金额混合兑换**（比例配置如 100 积分 = 1 元）、积分过期
3. 储值：充值套餐送赠金（本金/赠金分开记账，**退款只退本金**）；单用途预付卡属地合规调研（见风险 R4）
4. 会员卡：等级卡（折扣/积分加速，复用 member level）+ **计次卡自研**（次数扣减、有效期）
5. **优惠计算管道**：会员价 → 券 → 积分抵扣 → 储值支付；优惠快照 JSON 落订单 + 行级金额分摊（见 5.2）
6. 从 yshop-drink（MIT）移植积分兑换与充值套餐的参考实现

**验收**：一单组合使用 券+积分+余额，分摊金额正确、总额一致；部分退款后券/积分按规则回退；优惠规则修改不影响历史订单。

### M5 · 多人协同点餐（核心自研，产品护城河）

**目标**：同桌多人实时共享购物车、可分开结账。

**任务**

1. `table_session` 桌台会话：首人扫码开桌、他人扫码加入、成员列表、开桌通知
2. 实时购物车：WebSocket 房间 = sessionId；**服务端权威模型**（客户端只发操作意图，服务端广播版本化快照）；断线重连拉全量（见 5.1）
3. 协同下单：桌面合集购物车下单，clientToken 幂等去重；菜品标记"谁点的"
4. 并单结账：整单支付（任一人代付）+ 分单支付（按归属拆子支付单）
5. 生命周期：开桌→点餐→加菜→结账→清台、超时自动清台、转台/并台
6. 边界防护：会话限流、弱网重放、库存变更冲突提示
7. 精读 BiteGo 实现文档（7.1 桌台协同）交叉验证方案：opId 幂等 + cart.version 乐观锁 + Redis Pub/Sub 跨实例广播

**验收**：3 台真机同扫一桌，购物车同步延迟 <500ms；两人各付各的菜成功且金额正确；中途断网重连后状态一致；并发加菜不丢不重（压测脚本验证）。

### M6 · SaaS 运营 + 装修 + 公众号 + 上线加固

**目标**：能卖、能装、能扛。

**任务**

1. 微信第三方平台：代小程序授权/代码上传/发布（租户小程序免开发接入）；公众号授权发模板消息（订单状态通知）
2. 自定义装修：基于 diy 模块扩展餐饮组件（轮播、商品组、券礼包、公告、图片魔方）+ uniapp 渲染器（组件白名单映射，小程序不支持动态组件名）
3. SaaS 计费：套餐 = 功能开关矩阵 + 门店数/订单量配额；试用期、到期停用（`saas` 模块）
4. 上线加固：压测（下单 TPS 目标）、**每日支付对账**（见 5.6）、监控告警、备份策略、**租户越权安全测试**（重点：横向越权扫全接口）

**验收**：新租户 30 分钟完成 注册→授权小程序→装修→上线；对账连续 7 天零差异；越权测试零高危；压测报告达标。

***

## 5. 关键技术方案

### 5.1 多人协同点餐（服务端权威）

**消息协议**

```jsonc
// 客户端 → 服务端：操作意图（带幂等键与版本）
{ "action": "add_item", "skuId": 1001, "qty": 2,
  "opId": "uuid-xxx",        // 幂等键：服务端 5min 去重窗口
  "sessionId": "S123", "version": 42 }

// 服务端 → 房间广播：版本化快照
{ "type": "cart_snapshot", "version": 43,
  "items": [{ "skuId": 1001, "qty": 2, "addedBy": "用户A" }],
  "members": [{ "userId": 1, "nickname": "A", "joinedAt": "..." }] }

// 冲突：版本不一致返回 409 → 客户端拉全量快照，重放本地未提交操作
```

**关键设计**

- 服务端为唯一事实源；客户端零本地决策，杜绝"各改各的互相覆盖"

- `cart.version` 乐观锁 + `opId` 幂等键，解决同桌并发写入与弱网重放（BiteGo 同思路交叉验证）

- 多实例部署：进程内 `Map<tableSessionId, Set<WsClient>>` + Redis Pub/Sub 跨实例广播

- 断线重连：客户端带 last version，服务端返回增量或全量快照

- 会话生命周期钩子与订单域解耦（事件驱动），清台触发订单终态校验

### 5.2 优惠计算管道 + 快照

```
会员价 → 优惠券 → 积分抵扣 → 储值支付
   （管道模式：每步只消费上一步的金额结果）
```

- 计算完成后将**优惠快照**整体序列化进订单：用了什么、抵了多少、如何分摊到订单行

- 规则后续修改不影响历史订单与部分退款（退款按快照行级分摊回退）

- 积分抵扣比例、券互斥、会员折扣叠加顺序全部配置化

### 5.3 云打印可靠性

```
下单 → 发 RocketMQ → 打印消费者调云打印 API
     → 写 print_task 状态（pending/printed/failed）
     → 失败进重试队列（指数退避，最多 5 次）
     → 仍失败 → 商家端红点告警 + 人工补打按钮
```

- 适配器模式：`PrinterDriver` 接口 + 易联云/飞鹅实现，新厂商只加实现类

- 后厨分单按菜品分类路由（printer\_group）；单台故障自动降级到备份打印机（参考 Floreant）

- 打印内容与小票模板分离（模板引擎渲染 → 驱动发送）

### 5.4 多租户微信生态

- **租户小程序**：微信开放平台第三方平台模式，平台代开发/代上传/代发布；租户授权后免开发拥有自己的小程序

- **支付**：普通商户模式，钱直接进商家自己商户号，平台不经手资金（规避二清）；每租户配置自己的商户参数（租户级 pay channel 配置）

- **公众号**：第三方平台授权接管，订单状态模板消息推送

- 回调地址按租户路由；access\_token 集中管理（进程内 Map + Redis 双层 + 分布式锁防击穿）

### 5.5 库存防超卖

```
下单：Redis Lua 预扣（原子 check-and-decrement）
     → 支付成功后落 DB 扣减（乐观锁 version 校验）
     → 超时未支付：回滚 Redis + DB
```

- DB 层兜底：`stock = stock - #{qty} WHERE stock >= #{qty}`

- 估清（时段售罄）与库存（数量）分离：估清是开关，库存是计数

### 5.6 支付与对账

- 回调幂等：状态机 + 分布式锁 + 唯一索引（out\_trade\_no）三层防重

- 每日对账任务：拉微信/支付宝账单 ↔ 本地支付流水逐笔核对；差异进人工处理队列，连续告警

- 退款：原路退回，退款单独立状态机；部分退款按行级分摊快照计算

***

## 6. 数据库设计

### 6.1 规范

- 金额一律 **int 分**（bigint 亦可，全链路统一）；比例/积分同 int

- 所有业务表带 `tenant_id`（芋道自动注入过滤）、`deleted`（软删）、`creator/create_time/updater/update_time`

- 自建表放 `sql/custom/` 编号迁移脚本（`V2026.09.05.01__create_store.sql` 风格），与上游 SQL 分离

### 6.2 新增核心表（概要）

```sql
-- 门店（store 模块）
store              (id, tenant_id, name, logo, address, lng, lat,
                    business_hours json, dine_in/pickup/delivery 开关,
                    delivery_radius_m, min_order_amount, status)
store_table        (id, store_id, area_name, table_no, capacity,
                    qr_token, qr_signature, status)
table_session      (id, store_id, table_id, status(open/dining/settled/closed),
                    opened_at, settled_at, timeout_at)
table_session_member (id, session_id, user_id, nickname, joined_at)
cart_item          (id, session_id, sku_id, qty, spec_snapshot json,
                    added_by_user_id, op_id UNIQUE, version)

-- 商品餐饮化扩展
product_ext        (id, spu_id, sku_id, estimated_clear bool,
                    time_price json, kitchen_category)

-- 订单餐饮化扩展
order_dine         (id, order_id, session_id, table_id,
                    dine_mode(dine_in/pickup/delivery),
                    reserve_slot_id, reserved_at)

-- 营销
member_card        (id, tenant_id, user_id, card_type(level/times),
                    level_id, times_balance, valid_from, valid_to)
reserve_slot       (id, store_id, slot_date, slot_time, capacity, used)

-- 收银与打印（pos 模块）
pos_order          (id, store_id, cashier_id, table_id, pay_methods json,
                    total, discount, wipe_off(抹零))
printer            (id, store_id, vendor(yilianyun/feiyun),
                    device_sn, api_key_enc, kitchen_category, backup_printer_id)
print_task         (id, order_id, printer_id, template_type(receipt/kitchen),
                    content, status(pending/printed/failed),
                    retry_count, next_retry_at, printed_at)

-- SaaS（saas 模块）
saas_plan          (id, name, feature_flags json, max_stores, max_monthly_orders,
                    price_monthly, price_yearly)
tenant_subscription (id, tenant_id, plan_id, period_start, period_end,
                     status(trialing/active/expired), usage_cache json)
usage_daily        (id, tenant_id, stat_date, order_count, ... UNIQUE(tenant_id, stat_date))
```

### 6.3 迁移管理

- 上游升级 SQL 与自建表 SQL 两套目录，互不污染

- 每次发版迁移脚本向前兼容（只加列不删列；删列分两次发版）

***

## 7. 工程规范与基础设施

### 7.1 代码规范

- 跟随芋道既有分层（controller/service/dal/convert），自建模块同构

- MapStruct 做对象转换；禁止跨层直接引用 DO

- 单测覆盖：协同购物车、优惠管道、支付幂等三个核心域必须有单测；其余以集成测试为主

### 7.2 Git 工作流

- `main`（可发布）← `develop` ← `feature/m5-collab-session` 功能分支

- upstream 同步：每月一次 `merge upstream/master-jdk25` 到 develop，冲突按 `UPSTREAM_PATCHES.md` 处理

### 7.3 CI/CD

- CI：构建 + 单测 + license 扫描 + 镜像打包（GitHub Actions / Jenkins）

- CD：测试环境自动部署；生产手动审批发布

- 数据库迁移随发版流水线执行（先迁移后部署）

### 7.4 监控告警

- 基础：Spring Boot Actuator + Prometheus + Grafana

- 业务告警：支付回调失败率、打印失败率、对账差异、协同会话异常断连率

- 日志：ELK 或 Loki（多实例后必须有集中日志）

***

## 8. 团队分工与并行计划

| 角色           | 里程碑主线                      |
| ------------ | -------------------------- |
| 后端 A（业务向）    | M1 商品门店 → M2 订单支付 → M4 营销  |
| 后端 B（实时/硬件向） | M0 底座 → M3 收银打印 → M5 协同    |
| Web 前端       | 商家中心各期页面 → 收银台 → M6 装修编辑器  |
| 小程序          | M2 起 uniapp 顾客端 → M5 协同 UI |

**并行关系**：M2 后半段 ‖ M3 前半段可双线；M5 强依赖 M2 订单域完成；M4 与 M5 可部分并行（不同人）。

```
M0 ── M1 ── M2 ────── M4 ──┐
  │            └ M3 ────────┤
  └───────────────── M5 ────┴── M6
```

***

## 9. 风险登记册

| #  | 风险                                           | 等级 | 应对                                                        |
| -- | -------------------------------------------- | -- | --------------------------------------------------------- |
| R1 | 微信第三方平台认证周期长（数周）                             | 高  | M0 第一天启动申请；开发期用自有小程序                                      |
| R2 | SB4 生态残余兼容问题（jdk25 分支用户基数小）                  | 中  | M0 决策门体检；业务 bug 交叉查 jdk17 分支 issue；红灯退 jdk17              |
| R3 | LGPL/AGPL/MRPL 代码污染（fuint 系、Jeepay、Floreant） | 高  | 已核验并划红线（见核验报告）；CI license 扫描 + NOTICE.md 登记 + 只读清单（见 3.2） |
| R4 | 储值预付卡属地合规（单用途预付卡规定）                          | 中  | 资金走商家自己商户号；上线前咨询属地政策；必要时储值功能按地区灰度                         |
| R5 | 云打印机型号兼容                                     | 中  | 只承诺易联云/飞鹅，输出认证硬件清单；适配器模式留扩展                               |
| R6 | 协同点餐状态复杂（弱网/并发/断连）                           | 中  | 服务端权威 + 版本快照回放；先单人后多人渐进上线；M5 验收含压测                        |
| R7 | 上游芋道大改导致 merge 冲突剧增                          | 中  | 三条纪律（新模块/扩展点/改动清单）；深度改造区冻结不同步                             |
| R8 | 多租户横向越权                                      | 高  | M6 越权安全测试专项：扫全接口（改 tenant\_id/user\_id 重放）；CI 加越权用例       |
| R9 | 4 人人力不足                                      | 中  | M2 后重估；优先保 M2/M3/M5 主链路，M4 营销可裁剪范围（先券后卡）                  |

***

## 10. 上线后任务清单

| 任务                                | 触发条件                 | 说明                                  |
| --------------------------------- | -------------------- | ----------------------------------- |
| 每日支付对账巡检                          | 上线起                  | 连续 7 天零差异后转周报                       |
| 性能压测复测                            | 商户数 >20 或单店日订单 >1000 | 下单 TPS、协同广播延迟                       |
| （若 M0 红灯退 jdk17）升 master-jdk25 专项 | 上线后第一个低峰期            | OpenRewrite + 上游 merge；全量回归支付/打印/协同 |
| 数据库容量与慢查询治理                       | 每月                   | 订单/打印任务表分区评估                        |
| 上游同步例会                            | 每月                   | merge upstream，review 安全补丁          |
| 安全渗透测试                            | 商户数 >50 或接大客户前       | 外部或内部红队                             |

***

## 附：文档维护

- 本文档为活文档，每里程碑结束更新状态

- 架构级决策变更须新增 ADR（`docs/ADR/`），不在本文档内改写历史结论

- `UPSTREAM_PATCHES.md`、`NOTICE.md` 与本文档同级维护

