# AGENTS.md

本文件是仓库工作守则。改动代码前先读完。子目录 `AGENTS.md` 只能细化，不得削弱本文件的安全与风险边界。

## 1. 项目定位

`ctt` (crypto-trading-toolkit) = **面向业务系统的交易执行工具箱**，Rust 实现。

**业务层决定"买什么、什么时候买、买多少"，`ctt` 决定"怎么买得更好、更安全、更省"。**

业务不直接调用交易所，只提交 `OrderIntent`；`ctt` 负责拆单、择时、路由、风控、状态追踪、异常恢复。

### 1.1 我们不是策略框架（最重要的一条边界）

**判别规则：**

- 需要"预测价格方向、判断机会是否值得做"的计算 → **不属于** `ctt`
- "已经决定要交易，如何让这笔交易更好成交" → **属于** `ctt`

因此**不实现**：信号与因子、策略生命周期 trait、策略宿主(StrategyHost)、策略调度、回测框架、资金费率预测、基差情景收益、持有期经济性（Carry/Borrow/入场退出成本）、组合优化与资本分配、历史行情数据库、策略自身业务状态管理。

**不创建 `ctt-strategy` crate。** 收到上述需求时先确认，不要直接实现。此条为一票否决项，见 §2.0 V1。

### 1.2 当前范围

Binance 现货 + U 本位永续；普通账户与 Portfolio Margin；自营资金多账户多策略；算法为 TWAP/VWAP/POV/Iceberg/Chase/Sniper + 自建 SOR + 跨品种对冲；交付形态为 Rust crate + gRPC。

设计文档：`docs/superpowers/specs/2026-09-17-crypto-trading-toolkit-design.md`

## 2. 不可妥协的不变量

### 2.0 一票否决（Veto Rules）

以下两条不需要权衡、不存在"以后再说"，出现即拒绝。

**V1. 任何"预测方向 / 判断机会是否值得做"的计算都不得进入本仓库。**

包括：信号与因子、资金费率预测、基差情景收益、持有期经济性（Carry / Borrow / 入场退出成本）、目标仓位优化、组合资本分配、回测框架。

判别是二元的，不是时序的——不属于就是不属于，**不接受"先加进来、以后再拆"的过渡状态**。

典型被拒表述：*"这只是纯计算，不依赖 Runtime，所以放进来没关系。"* ——纯计算不是准入理由，**语义归属才是**。

**V2. 不得基于"独占账户"或"无其他写入者"的假设做设计。**

我们是被业务调用的工具，业务可能保留自己的交易路径、人工下单或其他执行器。因此：

- 不得假设账户内所有订单都由 `ctt` 产生；
- 不得把"无外部写入者"作为正确性前提；
- 不得因检测到外部订单而自动认领、自动取消或自动计入某个 Task。

正确做法：`ctt` 只对自己管理的意图与订单范围负责；检测到外部订单时**观察并告警**，并暂停受影响的新增风险直到完成对账。

唯一例外：`writer_domain_id` 的排他性——同一时刻只能有一个 `ctt` 实例持有同一 writer domain 的写权限，由 Ownership/Epoch 机制保证。这与"账户内无其他写入者"是两件事，不要混为一谈。

### 2.1 数值与单位

- 权威交易逻辑中的价格、数量、金额、费用、保证金一律使用 `DecimalValue`（`fastnum` 后端）。**禁止 `f64` 表示金额/价格/数量**，f64 只用于统计指标。
- 精确运算失败必须返回结构化错误，**禁止静默近似或舍入**（`1/3` 应显式失败）。
- 单位类别不可混用：`Price` / `BaseQuantity` / `QuoteQuantity` / `ContractQuantity` / `BaseDelta` / `ContractDelta` / `Amount` / `Rate`。
- **现货库存与衍生品有符号仓位必须分开建模**，禁止用一个 `signed Decimal` 同时表达两者。
- tick/lot/最小名义/价格带来自已验证的交易所规则；舍入方向显式输入，不自动增量跨过最小阈值。

### 2.2 订单状态、幂等与 Unknown

- 命令结果必须是三态：`Acknowledged` / `DefinitivelyRejected` / `OutcomeUnknown`。**禁止只返回 `Result<OrderId>`。**
- 超时、断线、解析失败 = `OutcomeUnknown`，**不等于**交易所已拒单。
- 只有取得 `DefinitivelyNotApplied` 证据，或 Adapter 明确保证该命令类可幂等重发（创建订单要求相同 `client_order_id`）时，才允许再次 Dispatch。否则保持 Unknown、剩余量持续计入潜在敞口、继续对账。
- `client_order_id` 由 `(owner, intent_id, leg, 分片序号)` 确定性生成。请求摘要由 `ctt` 计算，业务提供的值不权威。
- 启动、重连、恢复必须完成对账后才允许新增风险；**恢复时不得盲目重放非幂等实盘命令**。

### 2.3 时间与行情

- 区分 Event Time / Exchange Time / Receive Time / Monotonic Time；系统时钟回拨不得破坏状态机。
- 下单前校验行情新鲜度、订单簿完整性、序列连续性。过期、断档、乱序、来源不明的数据不得驱动实盘订单。

### 2.4 风险与授权边界

- 所有实盘命令必须经过：`TaskCoordinator 提议 → AccountSupervisor 串行化并预留 → 持久化 Outbox → CommandGate 批准 → Adapter 传输`。**不得为恢复或撤单建立 Adapter 旁路。**
- 约束只能收紧：`Requested ∩ 授权 ∩ 风险策略 ∩ 生产授权 = Effective Constraints`。业务请求不得覆盖更严格的平台配置。
- PM 风险必须在**账户级**评估，不能只看当前 Task 的腿；**禁止跨 margin domain 净额计算保证金**。
- 交易所风险证据、本地保证金估算、经济敞口三者必须区分，不得把估算伪装成交易所事实。
- 失败关闭：所有权丢失、存储不一致、私有流缺口无法对账、数据过期 → 拒绝新增风险并保留诊断证据。
- **Cancel ≠ Flatten**；Flatten 本身是实盘交易，需提前授权或单独批准。
- 账户副作用（Auto Borrow / Auto Repay / Transfer）首期只实现 `Forbidden`；现货卖超可用库存直接拒绝。

### 2.5 凭证与实盘授权

- 能够实现实盘功能 ≠ 获准使用。凭证只来自 env 或 secret manager，不得硬编码、不得进入日志/错误/测试。
- 日志、错误、HTTP/WS 追踪必须脱敏；签名原文、认证头、完整私有 payload 不得进入常规日志。
- 测试禁止使用真实凭证、真实资金账户或生产端点。`LIVE=true` 必须显式设置；未设置时连接非 testnet 一律拒绝启动。
- 生产授权需明确：环境、账户别名（不在对话中暴露凭证）、品种范围、允许操作、最大名义/仓位限额、有效期、停止方式。

## 3. 目录结构与依赖方向

```
proto/ctt.proto              gRPC 契约（权威数据源）
crates/
  ctt-domain                 数值、单位、身份、时间、品种、账户事实
  ctt-tools                  规则/舍入、盘口、费用、基差、能力检查
  ctt-adapter-spi            能力、命令、结果、查询、证据、流、屏障、端口
  ctt-execution              纯执行核心：Input/Context、Child、Slicing、ChildPolicy、Leg、Group、Route、Exposure
  ctt-adapter-binance        Binance 认证/签名/限频/WS/命令翻译/证据
  ctt-runtime                Journal/Outbox、AccountSupervisor、预留、CommandGate、对账、生命周期
  ctt-api                    Rust 门面 + gRPC 服务
  ctt-testkit                确定性时钟与脚本时间线
  ctt-tca                    基准、机械成本、组合指标、分项报告
apps/ctt-server  apps/ctt-cli
```

**依赖严格单向**：`api → runtime → {execution, adapter-spi, tools, domain}`；`adapter-binance → adapter-spi + domain`。

- 纯执行核心不得：访问网络、读数据库、读系统墙上时间、直接调用 Adapter
- Adapter 不得：拥有业务 Task、做多腿协调、做账户风险裁决
- 发现需要反向依赖 = 抽象位置错了，回来改结构

## 4. Binance 适配陷阱（改 `ctt-adapter-binance` 必读）

1. **限频是权重制**：`REQUEST_WEIGHT`(1min 滑窗) + `ORDERS` 双维度。必须读 `x-mbx-used-weight-1m` 做反馈式动态限流；**为撤单与风控动作预留下单额度**。
2. **三种账户模式端点不同**：Spot `/api/v3/*`、UM Futures `/fapi/v1/*`、**PM `/papi/v1/*`**。同一 key 只属一种模式，启动时探测校验，不匹配 fail-fast。
3. **PM 的保证金是跨品种净额计算**，必须走 `pmAccountInfo`（uniMMR），不能只算永维持仓。PM 下现货下单端点以 `docs/binance-pm-notes.md` 实测结论为准。
4. **listenKey 每 30 分钟续期**；失败或断线 → 重建 + 强制全量对账，完成前禁止新增风险。
5. **订单簿 `U`/`u` 连续性校验**，gap 立即重拉快照，禁止"带洞"的簿子参与决策。
6. **`positionSide` 单/双向持仓模式**必须显式配置并启动探测校验。
7. **条件单 `workingType` 默认 MARK_PRICE**，`priceProtect` 对 stop-limit 必须开启。
8. **持续校准 server time offset**，`recvWindow` 超限本地拒绝下单。
9. **交易所原生 SOR 与自建 SOR 禁止叠加**，配置层互斥校验。
10. **PM 无官方 testnet**，现货与 UM futures testnet 凭证不通用；不要假设能用 testnet 覆盖 PM 逻辑。

## 5. 算法实现规则

算法按正交维度组合，**不要每种算法写一个大类**：

```
Coordinator(Single|HedgeSaga) × Slicer(Fixed|TWAP|VWAP|POV)
× ChildOrderPolicy(BoundedIOC|Iceberg|MakerFirst|Chase)
× Urgency × PriceGuard × FailurePolicy
```

- 新增算法 = 新增 `Slicer` 或 `ChildOrderPolicy` 实现 + 一个授权 `ProfileRef`。**不新增 `XxxExecutionClient`，不改公共 Schema。**
- 每个算法必须明确：输入意图、优化目标、约束、行情过期/深度不足/限频/交易所降级时的行为、部分成交与尾量处理、撤单与成交竞态处理、可解释的基准。
- **不得为追求成交率绕过价格保护、风险限额或授权边界。**
- **不得在没有目标函数、假设、基准和可复现实证时称某算法"最优"。**
- 执行质量必须分六类独立报告：机械成本、事前敞口风险费用、实际临时敞口 PnL、尾损、约束违反、未完成机会成本。**实际临时敞口 PnL 必须与事前风险费用分开报告**，否则危险算法可能因碰巧盈利而被误评。

## 6. 编码规范

- 非测试代码禁止 `unwrap` / `expect` / `panic!` / `unreachable!`，用 `thiserror` 传播错误
- 错误需分类为 `Retryable` / `RateLimited` / `Rejected` / `Fatal`，重试策略由分类驱动而非匹配字符串
- 异步统一 `tokio`；禁止阻塞 IO 跨 `.await`，禁止 `std::sync` 锁跨 `.await`
- 日志统一 `tracing`，禁止 `println!`；成交、撤单、风控拦截、kill-switch 变更必须有审计级日志
- 新增配置必须给保守默认值，不允许默认放开风险；不引入调试后门或未解释的宽松默认值
- 提交前必须全绿：
  ```
  cargo fmt --all -- --check
  cargo test --workspace --locked --offline
  cargo clippy --workspace --all-targets --locked --offline -- -D warnings
  cargo build --release --locked --offline
  ```

## 7. 动工前必读

| 要改的东西 | 先读 |
| --- | --- |
| 下单/状态机/幂等 | `crates/ctt-runtime/src/command_gate.rs`、`crates/ctt-execution/src/child/` |
| 算法 | `crates/ctt-execution/src/slicing/`、`crates/ctt-execution/src/ioc/` |
| 风控与预留 | `crates/ctt-runtime/src/account_supervisor/`、`crates/ctt-execution/src/exposure/` |
| 交易所接入 | `crates/ctt-adapter-spi/src/`（能力/命令/结果/证据/屏障）+ `docs/binance-pm-notes.md` |
| 对外接口 | `proto/ctt.proto` |
| 持久化与对账 | `crates/ctt-runtime/src/journal/`、`crates/ctt-adapter-spi/src/barrier.rs` |

## 8. 常见任务 SOP

- **新增算法**：实现 `Slicer` 或 `ChildOrderPolicy` → 注册 `ProfileRef` → 补属性测试（总量守恒、不超价、不超参与率）→ 补确定性场景 → 补 TCA 基准
- **新增风控规则**：实现 `RiskRule` → 在 CommandGate 前注册 → 补默认阈值与单测 → 更新设计文档规则表
- **新增交易所/账户模式**：实现 Adapter SPI → 补齐能力声明与数量映射 → 补对账屏障 → 补契约测试
- **新增对外接口**：先改 `proto/ctt.proto` → 生成代码 → 在 runtime 实现逻辑。**禁止**在 `ctt-api` 写业务逻辑
- **改任何下单路径**：同时确认——是否经 AccountSupervisor 预留、是否落 Outbox、是否幂等、超时是否进 Unknown、是否有确凿证据才重发

## 9. 验证与报告纪律

- 先发现仓库实际使用的 workspace 与脚本，再选命令；不得臆造不存在的测试或构建流程
- 按影响范围逐步扩大：聚焦单测 → 属性测试 → 确定性场景 → 回放 → 受影响包门禁 → 授权的 testnet → 单独授权的生产 canary
- 事实状态标注 `Confirmed`（已直接检查/执行）/ `Inferred`（强推导）/ `Unverified`（未检查）。**不得把 `Inferred` 或 `Unverified` 写成 `Confirmed`。**
- 不得把跳过、失败、部分执行或不可访问的检查描述为通过；不得把本地合并写成远端 CI 已通过
- 测试避免：未受控网络访问、依赖真实墙上时钟、未记录 seed 的随机性、真实凭证与真实资金

## 10. 提交规范

格式 `<type>(<scope>): <description>`，`type` 为 `feat/fix/docs/test/bench/perf/refactor/chore`，scope 用 crate 短名，描述用英文祈使句，首行不超过 72 字符。

```
feat(execution): add iceberg child order policy
fix(runtime): keep reservation until unknown child order resolves
```

涉及下单、风控、数值、资金的改动**必须附带测试**，并在 PR 描述说明资金安全影响面。提交信息不得包含 AI 署名尾注。
