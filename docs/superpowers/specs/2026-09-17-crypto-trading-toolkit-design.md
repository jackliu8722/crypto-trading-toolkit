# crypto-trading-toolkit (ctt) 设计文档

日期：2026-09-17
状态：待评审

## 1. 定位

`ctt` 是**面向业务系统的交易执行工具箱**。Rust crate 为核心交付物，gRPC 为薄封装的跨语言接入层。

**职责边界：业务层决定"买什么、什么时候买、买多少"，`ctt` 决定"怎么买得更好、更安全、更省"。**

业务不直接调用交易所。业务提交 `OrderIntent`，`ctt` 负责拆单、择时、路由、风控、状态追踪与异常恢复，并返回执行进度与成交流水。

### 1.1 我们不是什么（边界的负向定义，比正向定义更重要）

| 不是 | 说明 |
| --- | --- |
| **不是策略框架** | 不提供 `Strategy` trait、策略生命周期、信号/因子、调度宿主。业务自己决定何时交易 |
| 不是回测框架 | 仿真只用于验证执行代码，不是产品功能 |
| 不是资金托管/账务系统 | 只暴露成交流水与持仓视图，不做总账与清算 |
| 不是行情数据库 | 不存储历史行情，不为研究提供数据平台 |
| 不是 Portfolio 管理平台 | 不做资本分配、组合优化 |

**判别规则**：任何需要"预测价格方向、判断机会是否值得做"的计算都不属于 `ctt`；任何"已经决定要交易，如何让这笔交易更好成交"的能力都属于 `ctt`。

按此规则，资金费率预测、基差情景收益、持有期经济性（Carry/Borrow/入场退出成本）**属于业务侧策略逻辑**，`ctt` 不实现。

### 1.2 已确认范围

| 维度 | 决策 |
| --- | --- |
| 调用形态 | Rust crate（同进程）+ gRPC 薄封装，共享同一套语义模型 |
| 交易所 | 首期仅 Binance |
| 品种 | 现货 + U 本位永续；交割合约不做 |
| 账户体系 | 普通账户与 Portfolio Margin 统一账户 |
| 算法 | TWAP / VWAP / POV / Iceberg / Chase / Sniper + 自建 SOR + 跨品种对冲 |
| 资金模式 | 自营资金，多账户 + 多策略 |

---

## 2. 与 Crypto Lab 设计的关系：借鉴与取舍

仓库 `crypto-trading-toolkit`（P0–P2c 已实现，220 个 `.rs` 文件，crate 前缀同为 `ctt-`）的总体架构设计是本项目的**主要参考源**。本节明确取舍，避免无意识照搬。

### 2.1 借鉴（直接采纳其设计结论与实现）

| 主题 | 采纳内容 | 理由 |
| --- | --- | --- |
| 数值表示 | `DecimalValue`（`fastnum` 后端）：有界解析、精确运算、**精度丢失返回错误而非近似** | 交易系统最基础的正确性来源；已有完整边界测试，不应重造 |
| 单位类别 | `Price` / `BaseQuantity` / `QuoteQuantity` / `ContractQuantity` / `BaseDelta` / `ContractDelta` / `Amount` / `Rate` | 现货与合约数量不能混用；比 `Price/Qty` 二分严谨得多 |
| 现货 vs 合约 | 现货库存（Inventory / Frozen / Borrowed / Liability）与衍生品有符号仓位（Signed Position / Slot / Native Contracts / Multiplier）**分开建模** | 防止 `signed Decimal` 同时表达两种截然不同的事实 |
| Adapter SPI 分层 | Capability / Command / Outcome / Query / Evidence / Stream / Barrier 分离 | 这是整个可靠性的地基 |
| 命令结果三态 | `Acknowledged` / `DefinitivelyRejected` / `OutcomeUnknown` | 超时 ≠ 失败；重复下单的根源都在这里 |
| Venue Evidence | `ConfirmedApplied` / `DefinitivelyNotApplied` / `StillUnknown` | 只有确凿证据才允许重发 |
| Exposure Projection | 条件数量投影，计入已成交、Working、Cancel-pending、**Unknown**、剩余可能成交量 | PM 账户风险必须基于"所有可达状态"而非"当前净仓" |
| 确定性 Testkit | `ManualClock` + Script 事件时间线，无墙上时钟依赖 | 让部分成交/乱序/断线/重启可复现 |
| TCA 六类分项 | 机械成本 / 事前敞口风险费用 / 实际临时敞口 PnL / 尾损 / 约束违反 / 未完成机会成本 | **对工具定位尤其重要**：这是证明"工具带来多少价值"的唯一客观依据 |
| 约束只收紧不放宽 | `Requested ∩ 授权 ∩ 风险策略 ∩ 生产授权 = Effective Constraints` | 业务不能通过请求放宽平台限制 |
| 执行策略正交组合 | Coordinator × Slicer × ChildOrderPolicy × Urgency × PriceGuard × FailurePolicy | 避免每种算法一个类导致的组合爆炸 |
| ExecutionProfileRef | 业务选择授权过的命名配置，不拼装内部算法 | 加新算法不需要新 Client |
| 四域分离 | `route_id` / `writer_domain_id` / `margin_domain_id` / `exposure_group_id` | 跨账户模式与跨品种对冲的正确性前提 |
| LifecycleWritePolicy | `Ready` / `Reconciling` / `Draining` / `OwnershipLost` 的写权限矩阵 | 恢复与停机路径不能成为风控旁路 |
| 失败关闭 | 所有权丢失、存储不一致、私有流缺口无法对账、数据过期 → 拒绝新增风险 | 兜底行为必须显式 |

### 2.2 不采纳（属于策略框架，或超出工具定位）

| 不采纳 | 原设计中的形态 | 不采纳理由 |
| --- | --- | --- |
| `ctt-strategy` | funding 情景、持有期经济性、基差收益 | 属于"判断机会是否值得做"，是策略职责 |
| `StrategyHost` | 策略 start/stop/recover、事件分发、checkpoint | 我们不做策略宿主 |
| 策略生命周期与决策草稿 | `StrategyDecisionDraft` 的完整语义 | 简化为 `OrderIntent`，不含策略自身业务状态 |
| 期权扩展边界章节 | Greeks、波动率曲面 | 交割合约已排除，期权更远 |
| 通用 N 腿执行图 | N-leg Execution Graph | 首期最多两腿（对冲），不为假想需求建模 |

### 2.3 因定位不同而需要调整的部分

| 主题 | 原设计 | 本项目调整 | 原因 |
| --- | --- | --- | --- |
| 账户独占 | 要求专用账户、无任何其他写入者、不允许人工交易 | 降级为：**`ctt` 管理的意图与订单范围独占**；检测到外部订单时观察并告警，不自动认领/取消/计入某个 Task | 业务可能保留自己的其他交易路径；工具不能假设独占整个账户 |
| 多策略共享净仓 | 明确不做，需要 Allocation Ledger | 提供 **strategy 维度的敞口统计与限额**（`StrategyScope` 作为 Owner 标签），但**不做仓位归属账本** | 用户要求多策略；但我们不是账务系统，归属不是我们的权威 |
| 仓位归属 | Position Ownership 由 Runtime 权威持有 | `ctt` 只记录"哪个 strategy 提交了哪个 intent"及由此产生的敞口贡献；账户真实仓位始终以交易所事实为准 | 工具不拥有仓位，只追踪自己造成的增量 |
| SOR | 明确延后 | 首期实现（跨账户模式、跨 symbol 路径、跨品种对冲） | 用户明确要求 |
| 全套算法 | 仅 Fixed Slicing + Bounded IOC | 扩展 Slicer 与 ChildOrderPolicy 家族 | 用户明确要求 |
| gRPC 服务 | 延后到"第一个跨进程消费者" | 首期实现 | 用户明确要求 crate + gRPC 双形态 |

---

## 3. 核心对外契约

### 3.1 业务提交意图，不是订单

```rust
struct OrderIntent {
    intent_id:     IntentId,       // 业务侧幂等键
    owner:         StrategyScope,  // (AccountId, StrategyId)：敞口统计与限额维度
    legs:          Vec<Leg>,       // 1~2 腿
    objective:     Objective,      // 优化目标：完成率 / 价格改善 / 低冲击 / 成本
    constraints:   Constraints,    // 只能收紧，不能放宽
    profile:       Option<ProfileRef>, // 授权过的命名执行配置
    dry_run:       bool,
}

struct Leg {
    route:     RouteId,            // 安全别名；ctt 内部解析为账户/品种/账户模式
    delta:     LegDelta,           // SpotInventoryDelta | DerivativePositionDelta
    price_guard: PriceGuard,
}
```

返回 `ExecutionHandle`：查询进度、请求取消、订阅成交流。

gRPC：`SubmitIntent / CancelIntent / GetTask / StreamFills / StreamProgress`，以及**单独授权**的 `Admin`（KillSwitch / Suspend / EmergencyFlatten）。

### 3.2 不建议出现的设计

```
TwapExecutionClient / HedgeExecutionClient / SorExecutionClient   ← 禁止
```

新增算法只增加 `Slicer` 或 `ChildOrderPolicy` 实现与一个 `ProfileRef`，不新增 Client、不改公共 Schema。

### 3.3 不可妥协的约束

1. **幂等**：`client_order_id` 由 `(owner, intent_id, leg, 分片序号)` 确定性生成；请求摘要（canonical digest）由 `ctt` 计算，业务提供的值不权威。
2. **`Unknown` 是一等状态**：超时/断线进入 `Unknown`，只有 `DefinitivelyNotApplied` 证据或 Adapter 明确保证幂等才允许重发。
3. **约束只收紧**：业务请求只能与平台配置取交集，不能覆盖。
4. **单一发送边界**：所有到达 Adapter 的命令必须已经耐久登记并通过 `CommandGate`。
5. **权威来源不混淆**：交易所事实 > `ctt` 持久状态 > 内存投影；`Submit` ≠ 成交，`CancelRequest` ≠ 已撤单，`Timeout` ≠ 失败。

---

## 4. 架构分层与 crate

| 层 | 职责 | crate | 与旧项目关系 |
| --- | --- | --- | --- |
| L0 领域 | 数值、单位、身份、时间、品种、账户事实 | `ctt-domain` | 复用 |
| L1 纯工具 | 规则/舍入、盘口、费用、基差、能力检查、流与屏障声明 | `ctt-tools` | 复用 |
| L2 Adapter SPI | 能力、命令、结果、查询、证据、流、对账屏障、端口 | `ctt-adapter-spi` | 复用 |
| L3 执行核心 | Input/Context、Child、Slicing、ChildPolicy、Leg、Group、Route、Exposure | `ctt-execution` | 复用并扩展算法家族 |
| L4 Binance 适配器 | 认证、签名、权重限频、WS、命令翻译、证据 | `ctt-adapter-binance` | **新建** |
| L5 Runtime | 任务协调、账户监督、预留账本、持久化 Outbox、对账、CommandGate | `ctt-runtime` | **新建** |
| L6 接口 | Rust 门面 + gRPC 服务 | `ctt-api` / `apps/ctt-server` | **新建** |
| 支撑 | 确定性测试、TCA、可观测 | `ctt-testkit` / `ctt-tca` / `ctt-observe` | testkit/tca 复用 |

**依赖方向**：`api → runtime → {execution, adapter-spi, tools, domain}`；`adapter-binance → adapter-spi + domain`；`execution → {tools, adapter-spi, domain}`，不依赖具体 Adapter 与 Runtime。

**不创建 `ctt-strategy`。**

---

## 5. 领域与数值

- 数值：`DecimalValue`（`fastnum`），精确运算失败即错误，不静默近似；`1/3` 这种情况显式失败
- 单位类别：`Price`、`BaseQuantity`、`QuoteQuantity`、`ContractQuantity`、`BaseDelta`、`ContractDelta`、`Amount`、`Rate`
- 现货：Total Inventory / Available / Frozen / Borrowed / Liability
- 永续：Signed Native Position、Position Mode(OneWay/Hedge)、Slot、Base-equivalent、Multiplier、Mark Price
- 时间：区分 Event Time / Exchange Time / Receive Time / Monotonic Time；系统时钟回拨不得破坏状态机
- 账户副作用（Auto Borrow / Auto Repay / Transfer）：Schema 保留，**首期只实现 `Forbidden`**。现货卖超可用库存直接拒绝，不自动解释为借币卖出

---

## 6. Adapter SPI 契约（复用既有设计）

```
Capability     粗粒度能力 + 精确组合/重复声明，绑定来源版本与观察时间
Command        Submit / Cancel / Amend，内部不可变请求
Outcome        Acknowledged | DefinitivelyRejected | OutcomeUnknown
Query          精确查询契约 + 覆盖范围 + 索引延迟声明
Evidence       Applied | NotApplied | StillUnknown，绑定完整命令与尝试
Stream         公共/私有流声明 + 帧级来源绑定（Data/Heartbeat/Gap/Disconnected/Ended）
Barrier        SharedSequence | QueryDrain，声明证据要求而非对账完成证明
Port           CatalogRead / QueryRead / StreamRead / AdapterRead（不含 CommandWrite 公开）
```

Adapter 负责协议与 I/O，**不拥有业务 Task、不做多腿协调、不做账户风险裁决、不能成为绕过 Runtime 的旁路**。

---

## 7. Binance 接入（新建）

三种账户模式必须显式建模并在启动时探测校验，不匹配即 fail-fast：

| 能力 | Spot | UM Futures | Portfolio Margin |
| --- | --- | --- | --- |
| 下单 | `/api/v3/order` | `/fapi/v1/order` | `/papi/v1/um/order`；现货走 `/papi/v1/margin/order` |
| 交易规则 | `/api/v3/exchangeInfo` | `/fapi/v1/exchangeInfo` | `/papi/v1/exchangeInfo` |
| 余额 | `/api/v3/account` | `/fapi/v1/balance` | `/papi/v1/balance` |
| 风险 | — | `/fapi/v1/positionRisk` | `/papi/v1/um/positionRisk` + `/papi/v1/pmAccountInfo` |
| user stream | `/api/v3/userDataStream` | `/fapi/v1/listenKey` | `/papi/v1/listenKey` |

其他要点：

- **权重制限频**：`REQUEST_WEIGHT`(1min 滑窗) + `ORDERS` 双维度；读取 `x-mbx-used-weight-1m` 做反馈式动态限流；**为撤单与风控动作预留下单额度**，避免算法打满额度导致风控指令发不出去
- **listenKey 每 30 分钟续期**；续期失败或断线 → 重建 + 强制全量对账，对账完成前禁止新增风险
- **订单簿 `U`/`u` 连续性校验**，出现 gap 立即重拉快照，禁止"带洞"的簿子参与决策
- **`positionSide` 单/双向持仓模式**显式配置并启动探测校验
- **条件单 `workingType` 默认 MARK_PRICE**，`priceProtect` 对 stop-limit 必须开启
- **持续校准 server time offset**，`recvWindow` 超限本地拒绝
- **交易所原生 SOR 与自建 SOR 禁止叠加**，配置层互斥校验
- 签名支持 Ed25519（推荐）/ HMAC / RSA；密钥只从 env 或 secret manager 读取

> 前置任务：PM 下现货下单的确切端点与参数需实测确认，产出 `docs/binance-pm-notes.md`。

---

## 8. 账户、策略与四域

四个标识不可混淆：

| 标识 | 含义 |
| --- | --- |
| `route_id` | 某条腿实际使用的配置执行路径 |
| `writer_domain_id` | 实盘写权限的排他单位 |
| `margin_domain_id` | 共享保证金/余额/负债/清算风险的范围 |
| `exposure_group_id` | 经济上互相对冲的一组腿 |

- 同一 PM 账户的 Spot 与 Perp：**margin domain 相同**，必须在账户级评估风险，不能只看当前 Task 的两条腿
- 跨交易所/跨账户：**禁止跨 margin domain 净额计算保证金**
- 每个 `margin_domain_id` 任一时刻只有一个权威账户投影、一个预留账本、一个风险仲裁边界

**多策略处理**：`StrategyScope(account, strategy)` 作为 Owner 标签用于敞口统计与限额，**不建立仓位归属账本**。多个策略共享账户净仓时，配置 `netting_policy`（Net / Isolated）显式声明是否允许跨策略自然对冲。

**外部订单**：检测到不属于 `ctt` 的订单时，观察并告警，不自动认领、不自动取消、不计入任何 Task，并暂停受影响的新增风险直到完成对账。

---

## 9. 执行算法：正交组合而非算法类

```
Coordinator       Single | HedgeSaga(两腿)
Slicer            Fixed | TWAP | VWAP | POV
ChildOrderPolicy  BoundedIOC | Iceberg | MakerFirst | Chase
Urgency           Passive | Neutral | Aggressive
PriceGuard        Limit | MaxSlippageBps | PegToBbo
FailurePolicy     Halt | Continue | Compensate(pre-authorized)
```

TWAP/VWAP/POV 本质是 **Slicer**（何时切、切多少）；Iceberg/Chase 是 **ChildOrderPolicy**（怎么挂、怎么追）。二者可组合，例如 `TWAP Slicer + Iceberg ChildPolicy`。

纯执行核心不得：访问网络、读数据库、读系统墙上时间、直接调用 Adapter。Clock、随机源、行情、账户状态、品种元数据全部显式输入。

**SOR** 首期三类路由维度：跨账户模式、跨 symbol 路径（直接对 vs 三角）、跨品种对冲（Spot 多腿 + Perp 空腿）。两腿是 Saga 不是原子事务：一腿成交另一腿失败时进入预授权的 `Unwind` 补偿路径。决策输入含盘口深度、手续费模型、滑点估计、资金费率、可用额度。

---

## 10. 风控与发送边界

```
TaskCoordinator 提议 Effect
→ AccountSupervisor 串行化并预留风险（账户级，含 PM 非线性）
→ Durable Transaction 写入状态与 Outbox
→ CommandGate 最终批准 Dispatch
→ Adapter 传输已批准的命令
```

- `TaskCoordinator` 与 `AccountSupervisor` 都不能直接调用 Adapter 写
- 风险计算模型可替换，**风险裁决与最终发送边界不可绕过**
- 交易所风险证据、本地保证金估算、经济敞口三者必须区分，不得把估算伪装成交易所事实
- 两腿任务至少检查四个极端：都不成交 / 只成交 A / 只成交 B / 都成交；且必须纳入该 margin domain 的所有 Live、Cancel-pending、Unknown 订单与剩余可能成交量
- 三级 kill-switch（全局 / 账户 / 策略）；**Cancel ≠ Flatten**，Flatten 本身是实盘交易，需提前授权或单独批准
- 写权限矩阵：`Ready` 允许正常命令；`Reconciling` 只允许已持久化的恢复命令；`Draining` 只允许停机策略命令；`OwnershipLost` 全部禁止（含撤单）

---

## 11. 状态、持久化与对账

内部状态分三层：`Task/Group`（目标、约束、多腿进度、Saga 状态）→ `LegExecution`（进度、Policy、敞口贡献）→ `ChildOrder`（命令世代、Venue 身份、Ack、Unknown、Partial、Terminal）。

持久化：append-only Journal + Snapshot/Revision + Transactional Outbox + Inbox 去重。不引入 Kafka 或完整 CQRS。

- Outbox 是 At-Least-Once，**不宣称 Exactly-Once**；经济幂等依赖 `command_id`、命令世代、确定性 `client_order_id` 与对账
- 提交用 Revision/Epoch CAS；冲突则丢弃旧决策、重读状态、重新规划，**不发送旧命令**
- 启动屏障：恢复未终态 Task → 对账 Order/Fill/Balance/Position/Margin → 通过后才进入 `Ready`
- **不假设交易所 REST 快照与私有流有统一序列**；Binance 采用 REST 快照 + 私有流水位，辅以 Client Order ID 查询与保守静默期规则
- 恢复时**不得盲目重放非幂等实盘命令**

---

## 12. 对外接口

- **Rust crate**：`ExecutionHandle`（同进程）。公共 trait 最小化：`ExecutionSubmitter` / `TaskObserver` / `TaskController`
- **gRPC**：`tonic` + `prost`，`proto/ctt.proto`；服务端只做鉴权、参数绑定与转发，**禁止写业务逻辑**
- 聚合 Facade 可以提供，但不得把传输层类型泄漏进业务语义
- 取消语义必须明确：`CancelReceipt` 只表示取消请求被接受，**不代表已撤单**；迟到成交仍须计入；**不能仅凭取消请求释放预留**
- 首期通过 `GetTask` 快照查询状态；事件订阅历史回放按真实需求后置
- Admin 能力（KillSwitch / Suspend / EmergencyFlatten / 强制接管）**单独授权**，不进入普通 `TaskController`

---

## 13. TCA（工具价值的证明）

每次执行输出六类**独立**归因，不合并成一个总分：

| 分项 | 说明 |
| --- | --- |
| Mechanical Execution Cost | 相对冻结基准的机械成交成本，含费用 |
| Ex-ante Exposure Risk Charge | 事前临时敞口风险费用 |
| Realized Temporary-exposure PnL | 实际临时敞口盈亏（**与事前风险费用分开报告**，否则危险算法可能因碰巧盈利而被误评） |
| Tail Loss | 尾损 |
| Constraint Breach | 约束违反类别 |
| Incomplete Opportunity Cost | 未完成的机会成本 |

多腿使用组合级 Synthetic Basis/Spread 基准，不能只分别计算两腿 Arrival Mid。

每个决策保留：行情快照与序列、账户快照与 Revision、品种元数据版本、Adapter 能力版本、Route 快照、Profile/Policy 版本、风险模型版本与适用范围。

**不得在没有目标函数、假设、基准和可复现实证的情况下称某个算法"最优"。**

---

## 14. 测试策略

分层，逐级放大：

1. **单元**：数量与单位、舍入方向、费用、状态机、算法决策
2. **属性测试**：Pending 与重复传输效果在对账前始终保守计入；不超过 Effective Constraints；重复/乱序事件不产生重复经济效果；取消生效后不再发送普通命令且预留不提前释放；Unknown/过期数据/所有权丢失不产生无界新风险
3. **确定性场景**：部分成交、拒单、超时、撤单竞态、迟到成交、重复/乱序事件、断线、重启、Unknown 订单、私有流缺口、所有权丢失
4. **Adapter 契约测试**：元数据、数量映射、能力、Client Order ID、成交标准化、重连、查询可见窗口、对账屏障
5. **验证阶梯**：纯单元 → Fake Venue → 故障注入 → 录制回放 → 授权后的只读连接 → 单独授权的 testnet 写入 → 严格限额的生产 canary

**关键约束**：Binance **PM 无官方 testnet**，现货 testnet 与 UM futures testnet 相互独立、凭证不通用。PM 差异层通过录制回放（cassette）验证，再用极小额度真实资金兜底。

测试禁止：未受控网络访问、依赖真实墙上时钟、未记录 seed 的随机性、真实凭证与真实资金。

---

## 15. 里程碑

| 阶段 | 内容 | 验收 |
| --- | --- | --- |
| M0 | 领域与工具基础（数值、单位、品种、盘口、规则、能力）、确定性 Testkit | 精确运算边界与盘口/新鲜度测试全绿 |
| M1 | Adapter SPI 契约 + Binance 只读（行情、品种、账户）、PM 调研结论 | 只读链路稳定；`docs/binance-pm-notes.md` 产出 |
| M2 | 执行核心：Child / Slicing / ChildPolicy / Leg / Group / Route / Exposure | 确定性场景覆盖部分成交、Unknown、撤单竞态、重启 |
| M3 | Runtime：Journal/Outbox、AccountSupervisor、预留、CommandGate、启动屏障、对账 | 杀进程重启后状态精确恢复；无重复下单 |
| M4 | Binance 写入 + CommandGate + 三级 kill-switch + 可观测 | testnet 下单/撤单全通；风控拦截有审计 |
| M5 | 算法家族扩展（TWAP/VWAP/POV/Iceberg/Chase）+ 多账户多策略 | 每算法有 TCA 基准与不变量测试 |
| M6 | SOR + 跨品种对冲 + crate/gRPC 双形态 + CLI | 三角路由与两腿保护可用；双形态行为一致 |
| M7 | 混沌、压测、TCA 报告、小额真实资金验证 | 混沌用例全绿；延迟与限频达标 |

---

## 16. 风险与行动项

| 风险 | 应对 | 归属 |
| --- | --- | --- |
| PM 账户端点/参数不确定 | M1 前置调研，产出实测笔记 | M1 |
| PM 无 testnet | 录制回放 + cassette + 小额真实资金三层兜底 | M1/M7 |
| 权重限频被打满导致风控指令发不出 | ORDERS 维度独立限流 + 额度预留 | M4 |
| 多策略共享账户互相抵消敞口 | `netting_policy` 显式配置 + 单测 | M5 |
| 业务侧其他交易路径干扰 | 外部订单检测告警，不自动认领 | M3 |
| 与既有 `ctt-*` crate 命名空间冲突 | 已定：选项 A，在既有代码库演进（见第 17 节） | 已关闭 |

---

## 17. 决策：与既有 `ctt-*` 代码库的关系（已定：选项 A）

既有仓库 `crypto-trading-toolkit`（P0–P2c，完整 CI）与本设计使用**相同的 crate 前缀 `ctt-`**，且 `ctt-domain` / `ctt-tools` / `ctt-adapter-spi` / `ctt-execution` / `ctt-testkit` / `ctt-tca` 均已实现并通过验证，可直接承载本设计的 L0–L3 层。本设计真正需要新建的只有 `ctt-adapter-binance`、`ctt-runtime`、`ctt-api` 三个 crate，以及算法家族扩展。

**决策：采用选项 A——在既有代码库上演进。**

| 选项 | 做法 | 结论 |
| --- | --- | --- |
| A | 在既有仓库演进：删除 `ctt-strategy`，新增 adapter/runtime/api，扩展算法家族 | **采用**：无命名空间冲突、无双份维护、直接复用已验证的正确性基础设施与 CI |
| B | 新 workspace，通过 path/git 依赖复用既有 crate | 否决：需拆分可发布单元，跨仓版本同步成本高 |
| C | 完全重写 | 否决：重造数值、单位、能力、结果三态、Exposure、TCA、Testkit，重复投入并丢失已有正确性保障 |

### 落地步骤

1. 以既有仓库为代码基座，将本文档与 `AGENTS.md` 作为新的定位真源迁入；
2. 删除 `ctt-strategy`（`AGENTS.md` §2.0 V1 一票否决），移除总体架构设计中 Strategy 边界、StrategyHost 延后规则等相关章节；
3. 按 §2.3 改写独占账户假设与多策略处理（`AGENTS.md` §2.0 V2）；
4. 新增 `ctt-adapter-binance`、`ctt-runtime`、`ctt-api`；
5. 扩展 Slicer 与 ChildOrderPolicy 家族，实现 SOR。

### 里程碑影响

M0–M2 由"从零实现"压缩为"复用 + 补齐差异"：领域层、工具层、Adapter SPI、执行核心、Testkit、TCA 直接继承，只需按本设计补齐 SOR、算法家族与多账户多策略相关差异。M3（Runtime）起为全新建设。
