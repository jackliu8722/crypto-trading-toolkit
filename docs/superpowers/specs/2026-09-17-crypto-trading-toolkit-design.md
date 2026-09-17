# crypto-trading-toolkit (ctt) 设计文档

日期：2026-09-17
状态：待评审

## 1. 定位

`ctt` 是**面向业务系统的交易执行工具箱**。Rust crate 为核心交付物，gRPC 为薄封装的跨语言接入层。

**职责边界：业务层决定"买什么、什么时候买、买多少"，`ctt` 决定"怎么买得更好、更安全、更省"。**

业务不直接调用交易所。业务提交 `OrderIntent`，`ctt` 负责拆单、择时、路由、风控、状态追踪与异常恢复，并返回执行进度与成交流水。

### 1.1 判别规则

- 需要"预测价格方向、判断机会是否值得做"的计算 → **不属于** `ctt`
- "已经决定要交易，如何让这笔交易更好成交" → **属于** `ctt`

因此不实现：信号与因子、策略生命周期 trait、策略宿主、策略调度、回测框架、资金费率预测、基差情景收益、持有期经济性（Carry/Borrow/入场退出成本）、组合优化与资本分配、历史行情数据库。

### 1.2 范围

| 维度 | 决策 |
| --- | --- |
| 调用形态 | Rust crate（同进程）+ gRPC 薄封装，共享同一套语义模型 |
| 交易所 | 首期仅 Binance |
| 品种 | 现货 + U 本位永续；交割合约不做 |
| 账户体系 | 普通账户与 Portfolio Margin 统一账户 |
| 算法 | TWAP / VWAP / POV / Iceberg / Chase / Sniper + 自建 SOR + 跨品种对冲 |
| 资金模式 | 自营资金，多账户 + 多策略 |

---

## 2. 设计原则

以下六条从"交易执行的物理约束 + 工具定位"直接推导，是后续所有设计的依据。

### P1. 外部事实不可本地推定

交易所是订单、成交、余额、持仓的**唯一外部权威**。本地缓存、日志、内存状态都只是"当前认知"，可能过时。

**推导出的强制约束：**
- 写命令结果必须三态：`Acknowledged` / `DefinitivelyRejected` / `OutcomeUnknown`。**禁止只返回 `Result<OrderId>`**，因为它把"不知道"压缩成了"失败"。
- 超时、断线、解析失败 = `OutcomeUnknown`，**不等于**交易所已拒单。
- **查询未找到 ≠ 交易所没接受**：存在索引延迟、私有流缺口、历史 API 覆盖不足。
- 只有取得确凿"未生效"证据，或 Adapter 明确保证该命令可幂等重发时，才允许重发。
- 对账不是可选优化，是必需品；启动、重连、恢复必须先对账再接受新风险。

### P2. 风险裁决必须有唯一单点

如果"能不能发这笔单"有多个判定入口，就一定会漏。

**推导出的强制约束：**
- 全流程只有一条写入链：`意图 → 风控预留 → 持久化 → 发送闸门批准 → 传输`。
- **恢复、撤单、紧急处置不得建立旁路**。故障路径是最容易破防的地方。
- 约束只能收紧：`业务请求 ∩ 调用方授权 ∩ 风险策略 ∩ 当前生产授权 = 生效约束`。
- 交易所风险证据、本地保证金估算、经济敞口是三种不同性质的东西，不得把估算当事实。

### P3. 工具不拥有仓位

我们是被调用的工具，业务可能有自己的交易路径、人工下单或其他执行器。

**推导出的强制约束：**
- 不得假设账户内所有订单都由 `ctt` 产生。
- 检测到不属于 `ctt` 的订单：观察并告警，**不自动认领、不自动取消、不计入任何 Task**，并暂停受影响的新增风险直到对账完成。
- 多策略维度只做**敞口统计与限额**，不做仓位归属账本——归属不是工具的权威。
- 唯一保留的排他性是：同一时刻只有一个 `ctt` 实例持有同一账户的写权限（由所有权租约保证），这与"账户内无其他写入者"是两回事。

### P4. 算法描述目标，不描述步骤

TWAP/VWAP/POV 回答的是"**何时切、切多少**"；Iceberg/Chase 回答的是"**这一笔怎么挂**"。它们是两个正交维度，不是同一层的互斥选项。把它们做成并列的"算法类"会导致组合爆炸。

**推导出的强制约束：**
- 执行策略 = `协调器 × 切片器 × 子单策略 × 紧迫度 × 价格保护 × 失败策略` 的组合。
- 新增算法 = 新增一个切片器或子单策略 + 一个命名配置，**不新增 `XxxExecutionClient`、不改公共 Schema**。
- 业务选择授权过的命名配置，不拼装内部算法参数。

### P5. 不可判定时失败关闭

状态不明确时，正确的默认行为是**停止新增风险并保留证据**，而不是猜。

**推导出的强制约束：**
- 所有权丢失、存储不一致、私有流缺口无法对账、行情过期、品种元数据不可靠 → 拒绝新风险。
- 剩余可能成交量、未知结果订单、撤单待确认订单**始终保守计入**敞口。
- **Cancel ≠ Flatten**：平仓本身是实盘交易，需提前授权或单独批准。

### P6. 性能声明必须有基准

**推导出的强制约束：**
- 不得在没有目标函数、假设、基准与可复现实证时称某算法"最优"。
- 执行质量分六类独立报告，**实际临时敞口盈亏必须与事前风险费用分开**——否则危险算法会因碰巧方向盈利而被误判为更优。

---

## 3. 核心对外契约

### 3.1 业务提交意图

```rust
struct OrderIntent {
    intent_id:      IntentId,          // 业务侧幂等键
    owner:          OwnerScope,        // (账户, 策略)：敞口统计与限额维度
    legs:           Vec<Leg>,          // 1~2 腿
    exposure_group: ExposureGroupId,   // 声明哪些腿在经济上互相对冲
    objective:      Objective,
    constraints:    Constraints,       // 只能收紧
    profile:        Option<ProfileRef>,// 授权过的命名执行配置
    target_completion: Option<Timestamp>, // 软目标：尽力在此之前完成
    hard_deadline:     Timestamp,         // 硬截止：到达后不再发起新子单
    on_deadline:       DeadlineAction,    // 软目标未达成时的处置，必须显式选择
    dry_run:           bool,
}

struct Leg {
    route:       RouteId,       // 安全别名；ctt 内部解析为账户/品种/账户模式
    delta:       LegDelta,
    price_guard: PriceGuard,    // 限价 / 最大滑点 / 挂盘口
    deadline:    Option<Timestamp>,
}

/// 现货库存与永续仓位必须分开表达，禁止合并为一个有符号十进制
enum LegDelta {
    SpotInventory {
        base_delta: BaseDelta,          // 标的量增量，正为买入
    },
    ContractPosition {
        position_side: PositionSide,
        contract_delta: ContractDelta,  // 张数增量，方向语义见下
    },
}

enum PositionSide {
    Net,    // 单向持仓模式：唯一仓位槽，增量正负即增减
    Long,   // 双向持仓模式：多仓槽
    Short,  // 双向持仓模式：空仓槽
}

enum Objective {
    CompletionRate,     // 优先完成数量
    PriceImprovement,   // 优先价格改善
    LowImpact,          // 优先低市场冲击
    MinTotalCost,       // 优先综合成本（费用 + 滑点 + 冲击）
    PairedCompletion,   // 多腿：优先两腿同步成交，最小化单边裸敞口时长
}
```

`contract_delta` 一律解释为**对该槽位的增减**，符号**不**表示买卖方向：

| 持仓模式 | 增量符号 | 含义 |
| --- | --- | --- |
| `Net` | 正 | 净仓向多头方向移动 |
| `Net` | 负 | 净仓向空头方向移动 |
| `Long` | 正 | 开多 / 加多 |
| `Long` | 负 | 减多 / 平多 |
| `Short` | 正 | **开空 / 加空** |
| `Short` | 负 | 减空 / 平空 |

**槽位增减到买卖方向的映射由 `ctt` 负责**（例如双向持仓账户中 `Short` 槽的正增量对应卖出）。业务只需表达"增加还是减少某个槽位"，**不应自行推导买卖方向**——这正是最容易搞反的地方：直觉上"开空"像卖出所以想写负号，但在本契约里它是 `Short` 槽的正增量。

**`position_side` 必须与账户实际持仓模式匹配**：单向持仓账户提交 `Long`/`Short` 会被直接拒绝，双向持仓账户提交 `Net` 同样被拒绝。启动时探测校验，提交时再次校验——**不静默适配，不猜测业务意图**。

`exposure_group` 决定风控如何聚合风险：同一组内的腿按对冲后的净敞口评估；不声明则被当作独立风险，额度按 `|腿A| + |腿B|` 占用而非 `|腿A − 腿B|`。

### 3.2 提交结果

`submit` 同步完成准入校验与初始风控，返回五态结果：

```rust
enum SubmitOutcome {
    Accepted(ExecutionHandle),       // 已接受为执行任务并持久化；成交异步进行
    Preview(DryRunReport),           // dry_run：返回计划与风控结论，不产生任务
    Rejected(Rejection),             // 明确拒绝，未产生任何任务
    OutcomeUnknown(IntentKey),       // 提交结果未知
    Conflict(IdempotencyConflict),   // 相同幂等键但内容不同
}

/// OutcomeUnknown 的恢复凭据；幂等作用域是二元组，缺一不可
struct IntentKey {
    owner:     OwnerScope,
    intent_id: IntentId,
}

struct DryRunReport {
    accepted:        bool,                  // 是否通过准入与风控
    rejections:      Vec<Rejection>,        // 未通过的原因，可能多条
    risk_assessment: RiskAssessment,        // 预留估算、额度占用、账户级影响
    plan:            Option<ExecutionPlan>, // 通过时才有
    as_of:           Timestamp,
}

struct ExecutionPlan {
    slices:         Vec<PlannedSlice>,   // 每片：预计时间窗、数量、子单策略、保护价
    estimated_cost: Option<Amount>,      // None = 无法估算；Some(0) = 明确为零
    warnings:       Vec<PlanWarning>,    // 精度舍入、低于最小名义、数量尾差等
}
```

**`Accepted` 不代表已成交、不代表子单已发出**，只代表意图通过准入与初始风控并被持久化。这与 P1 一致：既不把"不知道"压缩成"失败"，也不把"已接受"膨胀成"已成交"。

**`Preview` 只在 `dry_run: true` 时出现**——它与 `Accepted` 互斥，因为 dry-run 刻意不产生持久化任务。这样业务拿到的一定是明确的一类结果，不必自己判断"Accepted 里的任务到底是不是真的"。

**`OutcomeUnknown` 时业务只能用同一个 `IntentKey` 查询，禁止换 ID 重发**——换 ID 重发是重复下单的主要来源。恢复方式见 §3.3 的 `find_by_intent`。

`estimated_cost` 同样区分 `None`（无法估算）与 `Some(0)`（明确为零），与 §3.4 的费用语义一致。

gRPC 对应 `SubmitIntentResponse`，同一套语义，不做二次映射。

### 3.3 客户端与执行句柄

```rust
impl ExecutionClient {
    /// 从幂等键恢复句柄 —— OutcomeUnknown 后的唯一安全恢复方式
    ///
    /// 幂等作用域是 (OwnerScope, IntentId) 二元组，必须完整提供：
    /// 不同策略完全可能使用相同的 IntentId，只传 IntentId 无法唯一确定该恢复哪个任务。
    /// **客户端不隐式绑定 OwnerScope**，每次查询显式携带，避免跨策略误恢复。
    async fn find_by_intent(
        &self,
        owner:     OwnerScope,
        intent_id: IntentId,
    ) -> Result<Option<ExecutionHandle>>;
}

impl ExecutionHandle {
    fn task_id(&self) -> TaskId;

    /// 当前一致快照；始终是权威数据来源
    async fn get_task(&self) -> Result<TaskSnapshot>;

    /// 等待进入终态
    async fn wait_terminal(&self, timeout: Option<Duration>) -> Result<TaskSnapshot>;

    /// 请求取消；返回不代表已撤单（见 §3.5）
    async fn request_cancel(&self, reason: CancelReason) -> Result<CancelReceipt>;

    async fn stream_fills(&self) -> Result<impl Stream<Item = FillEvent>>;
    async fn stream_progress(&self) -> Result<impl Stream<Item = ProgressEvent>>;
}
```

### 3.4 任务快照

```rust
struct TaskSnapshot {
    task_id:      TaskId,
    intent_id:    IntentId,
    revision:     Revision,          // 快照版本
    status:       TaskStatus,        // Working | Canceling | PartiallyFilled | Filled | Canceled | Expired | Failed
    cancel_state: CancelState,       // None | Requested | Confirmed
    legs:         Vec<LegSnapshot>,
    fills:        Vec<Fill>,
    errors:       Vec<TaskError>,
    as_of:        Timestamp,         // 快照生成时刻
}

struct LegSnapshot {
    leg_index:       u8,
    route:           RouteId,
    target:          Delta,          // 目标增量
    filled:          Delta,          // 已成交
    remaining:       Delta,
    avg_price:       Option<Price>,  // 无成交时为 None，不是零
    status:          LegStatus,
    working_orders:  u32,            // 在途订单
    unknown_orders:  u32,            // 未知结果订单
}

struct Fill {
    leg_index:      u8,
    child_order_id: ClientOrderId,
    venue_order_id: Option<VenueOrderId>,
    price:          Price,
    quantity:       Quantity,        // 原生单位
    fee:            Option<Amount>,  // None = 未知；Some(0) = 明确为零
    exchange_time:  Timestamp,
    received_time:  Timestamp,
}
```

两条容易被忽略但必须遵守的规则：

- **`None` 与零必须区分**：`fee: None` 是费用未知，`Some(0)` 是明确无费用。把未知当零会**静默低估成本**，是执行质量数据失真的头号原因。
- **`unknown_orders` 必须对业务可见**：它是判断真实风险敞口的依据。把它藏在内部等于让业务基于不完整信息做决策，违背 P1。

### 3.5 取消语义

`CancelReceipt` 只表示**取消请求被接受**，不代表订单已撤销、Task 已终态或风险已清除。

- 取消生效后停止产生新的普通子单，但已授权的风控与补偿动作仍可执行；
- 已发送、在途、结果未知的订单继续归集回报；**迟到成交仍计入，不得因取消而丢弃**；
- **不能仅凭取消请求释放风险预留**。

确认取消真正完成：轮询 `get_task` 直到 `status` 进入终态，或用 `wait_terminal`。**除终态外没有中间可信状态。**

### 3.6 幂等

幂等作用域为 `(OwnerScope, IntentId)`。相同 ID 相同内容 → 返回同一 Task；相同 ID 不同内容 → `Conflict`。

请求摘要由 `ctt` 对解码后的语义模型计算，**业务提供的摘要不权威**。子单 `client_order_id` 由 `(owner, intent_id, leg, 分片序号)` 确定性生成，保证重启重连后不重复下单。

### 3.7 错误分类

按**业务该做什么**分类，不按内部模块分类：

| 错误 | 含义 | 业务动作 |
| --- | --- | --- |
| `InvalidRequest` | 参数或语义不合法 | 修正后重发，**不可沿用同一 ID** |
| `CapabilityUnsupported` | 交易所或账户不支持该能力 | 换能力或换路由 |
| `RiskRejected(RiskReason)` | 风控拒绝 | 调整意图或申请授权 |
| `QuotaExceeded` | 额度不足 | 释放额度后重试 |
| `SystemNotReady` | 对账中 / 所有权丢失 / 系统降级 | **可安全重试** |
| `OutcomeUnknown` | 提交结果未知 | **只能用同一 ID 查询，禁止换 ID 重发** |

### 3.8 其他调用语义

**软目标与硬截止必须分开**，两者不可混为一谈：

| 字段 | 性质 | 含义 |
| --- | --- | --- |
| `target_completion` | **软** | 希望在此之前完成；未达成即触发 `on_deadline` |
| `leg.deadline` | **硬（腿级）** | 该腿更早的截止时间；必须 ≤ 意图 `hard_deadline`，提交时校验 |
| `hard_deadline` | **硬（意图级）** | 授权边界；到达后一律停止新增子单 |

`on_deadline` 只作用于**软目标未达成**的情形：

| `DeadlineAction` | 行为 |
| --- | --- |
| `CancelRest` | 撤销未完成部分，保留已成交 |
| `CompleteAggressively` | 放宽紧迫度追完剩余（**仍受价格保护上限约束**） |
| `KeepWorking` | 继续按原节奏挂单 |

**任何 `DeadlineAction` 都不得突破 `hard_deadline`**——它是授权边界，不是执行目标。`KeepWorking` 与 `CompleteAggressively` 只决定"到达软目标后要不要更激进"，不等于"可以一直执行下去"。到达硬截止后只保留三类动作：已发送订单的回报归集、必要的撤单、预授权的补偿（§8.3）。

**`on_deadline` 必须由业务显式选择，不设默认值**——三种行为对应的资金后果完全不同，替业务选就是替业务承担风险。

**`dry_run`**。完整执行准入校验、风控评估、算法切片与价格保护计算，**但不发送任何命令、不产生持久化任务**；结果通过 `SubmitOutcome::Preview(DryRunReport)` 返回（见 §3.2），因此与 `Accepted` 互斥。**不保证与真实成交一致，其中的预估成本不得用于成本核算。**

**并发提交**。同一账户的多个意图可并发提交，风控预留串行化执行（§9）。额度按提交顺序竞争，不足时返回 `QuotaExceeded`。意图之间互不阻塞，但可能因额度竞争被拒。

**流与快照**。流用于实时感知，**快照是唯一权威**。订阅中断后重新 `get_task` 拉取一致快照来补齐，**首期不提供流的断点续传**（见 §11）。

### 3.9 端到端示例

两腿合约对冲（BTC 永续多 + ETH 永续空，同一 PM 账户）：

```rust
use anyhow::anyhow;
use ctt_api::{ExecutionClient, OrderIntent, Leg, LegDelta, PositionSide,
              PriceGuard, Objective, Constraints, FailurePolicy,
              DeadlineAction, ProfileRef, ExposureGroupId,
              OwnerScope, RouteId, ContractDelta, Notional};

let intent = OrderIntent {
    intent_id:      IntentId::new("hedge-20260917-001")?,
    owner:          OwnerScope { account: "pm-main".into(), strategy: "basis-arb".into() },
    exposure_group: ExposureGroupId::new("btc-eth-basis-001")?,

    legs: vec![
        Leg {
            route:       RouteId::from("binance-pm-perp-btc"),
            delta:       LegDelta::ContractPosition {
                position_side:  PositionSide::Long,
                contract_delta: ContractDelta::from_str("1.5")?,   // 开多 1.5 张（Long 槽正增量）
            },
            price_guard: PriceGuard::MaxSlippageBps(10),
            deadline:    None,
        },
        Leg {
            route:       RouteId::from("binance-pm-perp-eth"),
            delta:       LegDelta::ContractPosition {
                position_side:  PositionSide::Short,
                contract_delta: ContractDelta::from_str("30")?,    // 开空 30 张（Short 槽正增量）
            },
            price_guard: PriceGuard::MaxSlippageBps(10),
            deadline:    None,
        },
    ],

    objective:   Objective::PairedCompletion,   // 优先两腿同步，最小化单边裸敞口
    constraints: Constraints {
        max_temp_exposure: Some(Notional::from_str("50000")?),
        failure_policy:    FailurePolicy::Compensate(Compensation::UnwindFilledLeg),
        ..Default::default()
    },
    profile:           Some(ProfileRef::from("hedge-perp-ioc-v1")),
    target_completion: Some(soft_deadline),          // 软目标：尽力在此之前完成
    hard_deadline:     hard_deadline,                // 硬截止：到点一律停止新增子单
    on_deadline:       DeadlineAction::CancelRest,   // 软目标未达成则撤单留仓
    dry_run:           false,
};

// 提交：同步返回五态之一
let handle = match client.submit(intent).await? {
    SubmitOutcome::Accepted(h)   => h,
    SubmitOutcome::Preview(p)    => return Err(anyhow!("unexpected dry-run report: {p:?}")),
    SubmitOutcome::Rejected(r)   => return Err(anyhow!("rejected: {r:?}")),
    SubmitOutcome::Conflict(c)   => return Err(anyhow!("idempotency conflict: {c:?}")),
    SubmitOutcome::OutcomeUnknown(key) => {
        // 关键：只能用同一个 (owner, intent_id) 恢复，禁止换 ID 重发
        client.find_by_intent(key.owner, key.intent_id).await?
            .ok_or(anyhow!("cannot resolve unknown outcome for {:?}", key.intent_id))?
    }
};

// 实时感知（快照始终是权威，流只用于及时性）
let mut fills = handle.stream_fills().await?;
tokio::spawn(async move {
    while let Some(fill) = fills.next().await {
        tracing::info!(leg = fill.leg_index, price = %fill.price, qty = %fill.quantity, "fill");
    }
});

// 等待终态并核对每腿真实进度
let snap = handle.wait_terminal(None).await?;
for leg in &snap.legs {
    println!("leg {}: filled={} remaining={} avg={:?} unknown_orders={}",
        leg.leg_index, leg.filled, leg.remaining, leg.avg_price, leg.unknown_orders);
}
```

对应 gRPC：

```protobuf
service Execution {
  rpc SubmitIntent(SubmitIntentRequest)      returns (SubmitIntentResponse);   // 内含 outcome 五态
  rpc FindByIntent(FindByIntentRequest)      returns (TaskSnapshot);           // 必须携带 owner
  rpc GetTask(GetTaskRequest)                returns (TaskSnapshot);
  rpc CancelIntent(CancelIntentRequest)      returns (CancelReceipt);
  rpc StreamFills(StreamFillsRequest)        returns (stream FillEvent);
  rpc StreamProgress(StreamProgressRequest)  returns (stream ProgressEvent);
}
```

**单独授权**的 Admin 接口不在此 service 内：`KillSwitch / Suspend / EmergencyFlatten / ForceTakeover`（见 §11）。

---

## 4. 系统结构与 crate 划分

```
Cargo.toml
proto/ctt.proto              gRPC 契约（对外语义的权威定义）
crates/
  ctt-core        精确数值、单位类型、时间语义、标识符、错误分类、领域模型
  ctt-venue       交易所抽象：trait、能力声明、限频、重试、时间同步、流重连
  ctt-binance     Binance 实现：签名、三种账户模式、命令翻译、结果证据
  ctt-market      行情与订单簿：快照+增量、连续性校验、新鲜度、盘口估算
  ctt-oms         订单与任务管理：状态机、幂等、日志与发件箱、对账、恢复
  ctt-algo        执行算法：切片器、子单策略、路由与对冲协调
  ctt-risk        风控：限额、风险预留、价格保护、熔断
  ctt-api         Rust 门面 + gRPC 服务实现
  ctt-sim         paper trading、撮合模拟、录制回放
apps/
  ctt-server      gRPC 服务二进制
  ctt-cli         运维与手动干预工具
```

### 4.1 依赖方向

```
api → oms → {algo, risk, market, venue}
algo → {market, risk, core}      算法不直接持有下单能力
risk → {market, core}
market → {venue, core}
binance → {venue, core}
sim → {venue, core}
```

`core` 不依赖任何上层，且是**所有 crate 的公共基础**（图中省略这条传递依赖，Cargo 中仍需显式声明）。**发现需要反向依赖 = 抽象位置错了，回来改结构。**

### 4.2 各层禁止事项

| 层 | 禁止 |
| --- | --- |
| `ctt-algo` | 直接调用交易所写接口（只能通过 `oms` 提供的受控端口） |
| `ctt-binance` | 拥有业务 Task、多腿协调、账户风险裁决 |
| `ctt-api` | 写业务逻辑（只做鉴权、参数绑定与转发） |
| `ctt-market` | 驱动下单决策以外的任何写操作 |

### 4.3 为什么是这些 crate

- **`core` 与 `venue` 分离**：数值与领域模型必须能在没有网络、没有交易所的环境里被完整测试。
- **`venue` 与 `binance` 分离**：trait 的存在首先是为了**可替换的假实现**——`ctt-sim` 需要一个 fake venue 才能做确定性测试。这不是为假想的多交易所预留。
- **`oms` 独立**：状态机、幂等、对账、恢复是同一件事的不同面，拆开就会出现"状态在 A、恢复逻辑在 B"的割裂。
- **`risk` 独立**：保证 P2 的"唯一单点"在物理上可见。
- 首期不建独立的能力契约层（SPI）——只有 Binance 一个实现时那是过度设计，等第二个交易所出现再抽。
- **执行质量度量不单独立 crate**——M6 才需要，此前由 `ctt-oms` 承载决策证据记录，避免空壳（见 §13）。

---

## 5. 领域模型与数值

### 5.1 数值

- 价格、数量、金额、费用、保证金通过 `DecimalValue` 新类型承载，使用精确十进制，**禁止 `f64`**。f64 只用于统计指标与图形。
- **精确运算失败（除不尽、溢出、精度丢失）必须返回错误，禁止静默近似或舍入**（如 `1/3` 应显式失败）。静默近似是资金差错的头号来源。
- 溢出、下溢、除零、负值行为必须显式定义。
- 底层数值库在 M0 阶段评估，**第一评估标准是"除不尽时能否返回错误而非舍入"**；选型封装在 `DecimalValue` 内部，不泄漏到业务代码。

### 5.2 单位

| 类型 | 用途 |
| --- | --- |
| `Price` | 价格，必须为正 |
| `BaseQuantity` / `QuoteQuantity` | 现货标的量 / 计价币量 |
| `ContractQuantity` | 合约张数 |
| `BaseDelta` / `ContractDelta` | 有符号增量 |
| `Amount` | 金额，可为负 |
| `Rate` | 比率，`1 = 100%`，`0.0001 = 1bp` |

**现货库存与永续仓位必须分开建模**，禁止用一个有符号十进制同时表达两者——它们的经济含义、保证金影响、可用量语义完全不同。

- 现货：总库存、可用、冻结、借入、负债
- 永续：有符号张数、持仓模式（单向/双向）、仓位槽位、合约面值、标记价格、结算币

### 5.3 时间

区分事件时间、交易所时间、本地接收时间、单调时钟。延迟、超时、TTL、重试与新鲜度各自使用合适的时钟语义；**系统时钟回拨不得破坏状态机**。

### 5.4 账户副作用

借币、自动还币、划转的 Schema 保留，**首期只实现 `Forbidden`**。现货卖超可用库存直接拒绝，不自动解释为借币卖出。

---

## 6. Binance 接入

三种账户模式必须显式建模并在启动时探测校验，不匹配即 fail-fast：

| 能力 | Spot | UM Futures | Portfolio Margin |
| --- | --- | --- | --- |
| 下单 | `/api/v3/order` | `/fapi/v1/order` | `/papi/v1/um/order`；现货走 `/papi/v1/margin/order` |
| 交易规则 | `/api/v3/exchangeInfo` | `/fapi/v1/exchangeInfo` | `/papi/v1/exchangeInfo` |
| 余额 | `/api/v3/account` | `/fapi/v1/balance` | `/papi/v1/balance` |
| 风险 | — | `/fapi/v1/positionRisk` | `/papi/v1/um/positionRisk` + `/papi/v1/pmAccountInfo` |
| 私有流 | `/api/v3/userDataStream` | `/fapi/v1/listenKey` | `/papi/v1/listenKey` |

要点：

- **权重制限频**：`REQUEST_WEIGHT`（1 分钟滑窗）+ `ORDERS` 双维度；读 `x-mbx-used-weight-1m` 做反馈式动态限流；**为撤单与风控动作预留下单额度**，否则算法拆单会打满额度导致风控指令发不出去。
- **listenKey 每 30 分钟续期**；失败或断线 → 重建 + 强制全量对账，完成前禁止新风险。
- **订单簿 `U`/`u` 连续性校验**，缺口立即重拉快照，禁止"带洞"的簿子参与决策。
- **`positionSide` 单/双向持仓模式**显式配置并启动探测校验。
- **条件单 `workingType` 默认 `MARK_PRICE`**，`priceProtect` 对 stop-limit 必须开启。
- **持续校准服务器时间偏移**，`recvWindow` 超限本地拒绝下单。
- **交易所原生 SOR 与自建 SOR 禁止叠加**，配置层互斥校验。
- 签名支持 Ed25519（推荐）/ HMAC / RSA；密钥只来自环境变量或密钥管理器。
- **PM 的保证金跨品种净额计算**，必须走 `pmAccountInfo` 的账户级指标，不能只算永维持仓。

> 前置任务：PM 下现货下单的确切端点与参数需实测确认，产出 `docs/binance-pm-notes.md`。

### 6.1 能力声明

交易所差异（订单类型、TIF、post-only/reduce-only 语义、仓位模式、限频、改单能力）必须**显式声明并验证**，不在通用代码里散布交易所名称分支。请求不受支持的能力时返回结构化错误或走预定义的安全降级，**不得静默改变经济语义**。

---

## 7. 账户与策略

| 概念 | 含义 |
| --- | --- |
| `RouteId` | 某条腿实际使用的配置执行路径（安全别名） |
| 写权限域 | 实盘写权限的排他单位，同一时刻只能有一个持有者 |
| 保证金域 | 共享保证金、余额、负债与清算风险的范围 |
| 敞口组 | 经济上互相对冲的一组腿 |

- 同一 PM 账户的现货与永续：**保证金域相同**，必须按账户级评估风险，不能只看当前 Task 的腿。
- 跨账户、跨交易所：**禁止跨保证金域净额计算保证金**。
- 每个保证金域任一时刻只有一个权威账户投影、一个预留账本、一个风险仲裁边界。

**多策略**：`OwnerScope(账户, 策略)` 作为标签用于敞口统计与限额。多策略共享账户时显式配置 `netting_policy`（允许自然对冲 / 强制隔离），因为不同选择会改变真实的敞口与保证金占用。

**多腿**：`ExposureGroupId` 标识经济上互相对冲的一组腿。属于同一敞口组的腿按净敞口评估风险，否则各自独立占用额度。**敞口组只影响风险聚合，不改变保证金域归属**——同一敞口组的两条腿可以分属不同保证金域（跨所场景），此时仍禁止跨域净额。

---

## 8. 执行算法与 SOR

### 8.1 正交组合

```
协调器      单腿 | 两腿对冲
触发条件    立即 | 条件（价格 / 价差 / 时间窗满足后启动）
切片器      Fixed | TWAP | VWAP | POV
子单策略    BoundedIOC | Iceberg | MakerFirst | Chase
紧迫度      被动 | 中性 | 主动
价格保护    限价 | 最大滑点 | 挂盘口
失败策略    停止 | 继续 | 预授权补偿
```

**Sniper 不是一个独立算法类**，而是 `固定单片 + 条件触发 + 激进子单策略` 的组合。触发条件这个维度正是为了让"等机会再出手"这类行为通过组合表达，而不是每来一个需求就新增一个大类。

纯算法核心不得：访问网络、读数据库、读系统墙上时间、直接调用交易所。时钟、随机源、行情、账户状态、品种元数据全部显式输入——否则无法做确定性测试。

### 8.2 每个算法必须明确

输入意图、优化目标与优先级、约束（时间窗、价格保护、参与率、最大重试）、行情过期与深度不足时的行为、部分成交与尾量处理、撤单与成交竞态处理、可解释的基准。

**不得为追求成交率绕过价格保护、风险限额或授权边界。**

### 8.3 多腿协调与腿间策略

两腿是**补偿事务，不是原子事务**——交易所不提供跨品种或跨账户的原子下单。协调器必须显式配置以下行为：

| 参数 | 含义 |
| --- | --- |
| 腿间顺序 | 顺序（指定先后）或并发 |
| 单边容忍时长 | 一腿已成交而另一腿未成交时，允许等待的最长时间 |
| 单边超时动作 | 停止并告警 / 追单补另一腿 / 平掉已成交腿 |
| 补偿路径 | **必须预先授权**；补偿动作同样经过风控与发送闸门，不是旁路 |
| 部分成交处理 | 一腿部分成交时，另一腿目标量是否按比例缩减 |

风控至少检查四个腿级极端：**都不成交 / 只成交 A / 只成交 B / 都成交**；并纳入该保证金域所有在途订单、撤单待确认订单、未知结果订单与剩余可能成交量。

**未配置 `failure_policy` 时默认为"停止并告警"**，会留下裸敞口。这是显式选择，不是由默认值替业务承担风险。

### 8.4 SOR

路由维度（前三类为首期，第四类延后）：

1. **跨账户模式**：同一品种在现货 / 永续 / PM 下的可成交性与成本差异
2. **跨 symbol 路径**：直接交易对 vs 三角路径
3. **跨品种对冲**：现货腿 + 永续腿
4. **跨交易所**：同一品种在不同交易所之间的路由（需第二个交易所适配器，见 §8.5）

决策输入含盘口深度、手续费模型、滑点估计、资金费率、可用额度。

### 8.5 跨所对冲（延后能力）

首期只有 Binance 适配器，**跨所对冲不在实现范围内**。但架构已预留接缝，且下列约束必须在设计阶段就明确——否则新增第二个交易所时会动摇状态与风险权威。

**已具备的接缝**：`RouteId` 抽象（业务只看到安全别名）、写权限域与保证金域分离、敞口组独立标识、协调器模型本身不假设单一交易所。

**跨所特有的硬约束**：

1. **禁止跨保证金域净额**：两个交易所各自清算与收取保证金，必须两边都占用。资金效率显著低于同所对冲，额度模型必须按**总占用**而非净敞口计算。
2. **单边暴露时间不可消除且更长**：跨所往返与成交回报延迟不对称，"一腿成交、另一腿未成交"的时间窗从毫秒级上升到百毫秒至秒级。必须配置单边容忍时长与超时动作。
3. **补偿成本更高**：平掉已成交腿需跨所执行，滑点与时间成本都更高；补偿路径必须预先授权并单独评估限额。
4. **资金分布与再平衡**：两所资金需调拨，而**划转属独立高风险权限**，不得由普通交易权限隐式获得；再平衡策略需单独设计。
5. **时钟与时间窗**：两所服务器时间分别校准；截止时间按各自交易所时间语义解释，不得混用。
6. **对账屏障差异**：不同交易所的对账机制不同（统一序列 vs 查询排水），必须分别实现，不能复用同一套屏障逻辑。
7. **单边所故障**：一个交易所宕机或维护时，另一所的腿成为裸敞口；必须预定义降级与补偿预案，并纳入失败关闭条件。
8. **价格保护基准不得跨所混用**：每条腿的价格保护必须基于**该腿所在交易所**的行情快照，不能用另一所的价格做保护基准。

**结论**：跨所对冲是"架构支持、实现延后"的能力。新增第二个交易所时，主要工作是适配器、对账屏障与风险模型适配，**不应改变本设计的领域语义、状态权威与风险裁决边界**。

---

## 9. 风控

写入链（唯一）：

```
算法/协调器 提议动作
→ 风控串行化并预留风险（账户级，含 PM 非线性）
→ 持久化日志与发件箱
→ 发送闸门最终批准
→ 交易所传输
```

- 提议方与风控都**不能直接调用交易所写接口**。
- 两腿任务至少检查四个极端：都不成交 / 只成交 A / 只成交 B / 都成交；且必须纳入该保证金域的所有在途订单、撤单待确认订单、未知结果订单与剩余可能成交量。
- 风险模型可替换，**风险裁决与最终发送边界不可绕过**。
- 三级熔断（全局 / 账户 / 策略）；紧急平仓需提前授权或单独批准。
- 写权限矩阵：`Ready` 允许正常命令；`Reconciling` 只允许已持久化的恢复命令；`Draining` 只允许停机策略命令；`OwnershipLost` 全部禁止（含撤单）。

---

## 10. 状态、持久化与恢复

状态分三层：`Task`（目标、约束、多腿进度、补偿状态）→ `LegExecution`（进度、策略、敞口贡献）→ `ChildOrder`（命令世代、交易所身份、确认、未知、部分成交、终态）。

持久化采用 **append-only 日志 + 快照/版本号 + 事务发件箱 + 收件箱去重**。不引入消息队列或完整 CQRS。

载体在 M2 阶段评估，评估标准依次为：崩溃后不丢失已确认写入、append 吞吐、可人工检视与恢复、不引入额外运维依赖。**倾向自管理 append-only 文件 + 定期快照**——交易系统场景下比嵌入式 KV 更可控、更易审计与人工恢复。评估结论记入 M2 记录。

- 发件箱是至少一次投递，**不宣称恰好一次**；经济幂等依赖确定性 `client_order_id`、命令世代与对账。
- 提交用版本号/租约 CAS；冲突则丢弃旧决策、重读状态、重新规划，**不发送旧命令**。
- 启动屏障：恢复未终态 Task → 对账订单/成交/余额/持仓 → 通过后才进入 `Ready`。
- **不假设交易所 REST 快照与私有流有统一序列号**；采用 REST 快照 + 私有流水位，辅以客户端订单号查询与保守静默期。
- **恢复时不得盲目重放非幂等实盘命令。**

---

## 11. 对外接口

**方法契约与数据结构以 §3 为准，本节只规定传输层特有的约束**，避免两处各写一份而逐渐分叉。

- **Rust crate**：同进程直接调用，无序列化开销；公共接口最小化——提交、按幂等键恢复、查询、取消、订阅。
- **gRPC**：`tonic` + `prost`，`proto/ctt.proto` 是对外语义的权威定义；protobuf 消息与 §3.4 的 Rust 结构一一对应。服务端只做鉴权、参数绑定与转发，**禁止写业务逻辑**。
- 聚合门面可以提供，但**不得把传输层类型泄漏进业务语义**。
- **实时流首期提供**：`StreamFills` 推送实时成交，`StreamProgress` 推送任务进度；订阅中断不隐含取消 Task。
- **历史事件回放延后**：不提供 `watch_task(after)`、游标续订或历史 Task 事件回放。只有真实消费者需要逐条补齐遗漏事件时，才单独设计事件版本、授权、保留期与重复/缺口处理。
- Admin 能力（熔断、暂停、紧急平仓、强制接管）**单独授权**，不进入普通调用接口。
- **本地与远程共享业务语义，但故障语义不同**：远程超时表示结果未知（`OutcomeUnknown`），不表示拒绝，也不表示 Task 不存在。**客户端库不得把远程超时翻译成本地错误**，否则业务会误判为重发时机。

---

## 12. 配置管理

配置按变更风险分三类，校验时机与变更方式不同：

| 类别 | 内容 | 校验时机 | 变更方式 |
| --- | --- | --- | --- |
| 身份与凭证 | 账户凭证、账户模式、持仓模式 | 启动时探测校验，不匹配 fail-fast | **必须重启** |
| 执行路径 | `RouteId` 映射（交易所、账户、品种、写权限域、保证金域、能力版本） | 启动时校验 + 每次提交时解析 | 需显式重载，重载期间拒绝新增风险 |
| 风险与算法 | 额度、限额、熔断阈值、价格保护默认值、算法 `ProfileRef` | 启动时校验 + 每次使用时读取 | **支持热更新** |

规则：

- 配置错误必须 fail-fast，不得带着可疑配置进入 `Ready`；
- 限额与熔断阈值支持运行时调整；**收紧类热更新立即生效，放宽类变更需显式确认并留下审计记录**；
- 算法 `ProfileRef` 一经任务接受即冻结版本，运行中的配置变更不影响已接受的任务；
- 配置版本与内容摘要纳入决策证据，保证事后可解释（§13）；
- 配置变更本身必须审计：变更者、时间、前后值、生效范围。

---

## 13. 执行质量度量

**归属**：M6 独立为 `ctt-tca` crate。M6 之前由 `ctt-oms` 记录决策证据（行情快照与版本、账户快照与版本号、品种元数据版本、能力版本、路由快照、策略配置版本、风险模型版本与适用范围），**不提前建空壳 crate**。

每次执行输出六类**独立**归因，不合并为总分：

| 分项 | 说明 |
| --- | --- |
| 机械执行成本 | 相对冻结基准的成交成本，含费用 |
| 事前敞口风险费用 | 执行前预估的临时敞口风险代价 |
| 实际临时敞口盈亏 | 实际发生的临时敞口盈亏（**必须与上一项分开报告**） |
| 尾损 | 不利情形下的损失 |
| 约束违反 | 违反的约束类别 |
| 未完成机会成本 | 未执行完部分的机会成本 |

多腿使用组合级基准，不能只分别计算两腿的到达中间价。

每个决策保留：行情快照与版本、账户快照与版本号、品种元数据版本、能力版本、路由快照、策略配置版本、风险模型版本与适用范围——**决策必须可事后解释**。

---

## 14. 测试策略

1. **单元**：数量与单位、舍入方向、费用、状态机、算法决策
2. **属性测试**：未决与重复传输在对账前始终保守计入；不超过生效约束；重复/乱序事件不产生重复经济效果；取消生效后不再发送普通命令且预留不提前释放；未知/过期数据/所有权丢失不产生无界新风险
3. **确定性场景**：部分成交、拒单、超时、撤单竞态、迟到成交、重复/乱序事件、断线、重启、未知订单、私有流缺口、所有权丢失
4. **交易所契约测试**：元数据、数量映射、能力、客户端订单号、成交标准化、重连、查询可见窗口
5. **验证阶梯**：纯单元 → 假交易所 → 故障注入 → 录制回放 → 授权后的只读连接 → 单独授权的 testnet 写入 → 严格限额的生产 canary

**关键约束**：Binance **PM 无官方 testnet**，现货 testnet 与 UM 永续 testnet 相互独立、凭证不通用。PM 差异层通过录制回放验证，再用极小额度真实资金兜底。

测试禁止：未受控网络访问、依赖真实墙上时钟、未记录 seed 的随机性、真实凭证与真实资金。

---

## 15. 里程碑

| 阶段 | 内容 | 验收 |
| --- | --- | --- |
| M0 | `core` + `market`：数值、单位、品种、订单簿、盘口估算、新鲜度 | 精确运算边界与连续性/新鲜度测试全绿 |
| M1 | `venue` + `binance` 只读：行情、品种、账户；PM 调研结论 | 只读链路稳定；`docs/binance-pm-notes.md` 产出 |
| M2 | `oms`：状态机、幂等、日志/发件箱、对账、恢复 | 确定性场景覆盖部分成交、未知、撤单竞态、重启 |
| M3 | `risk` + `binance` 写入 + 发送闸门 + 熔断 + 可观测 | testnet 下单/撤单全通；风控拦截有审计 |
| M4 | `algo`：切片器与子单策略家族 + 多账户多策略 | 每算法有基准与不变量测试 |
| M5 | SOR + 跨品种对冲 + `api` 双形态 + `cli` | 三角路由与两腿补偿可用；crate 与 gRPC 行为一致 |
| M6 | 混沌、压测、执行质量报告、小额真实资金验证 | 混沌用例全绿；延迟与限频达标 |

---

## 16. 风险与行动项

| 风险 | 应对 | 归属 |
| --- | --- | --- |
| PM 账户端点/参数不确定 | M1 前置调研，产出实测笔记 | M1 |
| PM 无 testnet | 录制回放 + 极小额度真实资金三层兜底 | M1/M6 |
| 权重限频被打满导致风控指令发不出 | `ORDERS` 维度独立限流 + 额度预留 | M3 |
| 多策略共享账户互相抵消敞口 | `netting_policy` 显式配置 + 单测 | M4 |
| 业务侧其他交易路径干扰 | 外部订单检测告警，不自动认领 | M2 |
| 多腿对冲的单边裸敞口 | 单边容忍时长 + 超时动作 + 预授权补偿；额度按总占用而非净敞口 | 延后（§8.5） |
