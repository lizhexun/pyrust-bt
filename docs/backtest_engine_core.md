# 回测核心架构与流程（Strategy First 设计）

本方案以策略作者的使用体验为起点，重新规划回测引擎：让“写策略”只关注行情、指标和下单，其他繁琐环节（事件调度、仓位校验、撮合、统计）全部由引擎代劳，同时保持当前 Rust 内核的性能优势。

---

## 设计原则

- **一根 K 线一个决策**：策略只需实现 `on_bar(ctx)`，其余生命周期回调保持可选。
- **API 接近交易直觉**：统一使用 `ctx.order.buy(...)` 与 `ctx.order.sell(...)`，通过 `quantity_type` 指定按手数、金额或权重下单（将权重设为 0 即可清仓）。
- **即时反馈**：卖出、买入在同一根 bar 内执行；成交会同步反映到 `ctx`，策略侧无需手动等待事件。
- **多标的天然支持**：`ctx.symbols` 携带所有标的最新价和持仓，按字典访问即可。
- **默认安全配置**：禁止裸卖空，仓位、现金检查和成本计算自动完成；需要扩展时再显式开启。
- **交易日规则可配置**：默认按照 A 股 T+1 交割，可在初始化时为特定标的声明 T+0，从而允许当日回转交易。
- **指标预计算**：回测前可一次性算好所需指标并加载进引擎，策略阶段只读结果，避免重复计算开销。

---

## 三步快速上手

1. **定义策略类**：继承 `Strategy`，实现 `on_bar(self, ctx)`。
2. **在 `on_bar` 中调用下单 Helper**：例如`ctx.order.sell(symbol="510100.SH", quantity=0.3, quantity_type="weight")`。
3. **启动回测**：传入策略与数据，其他流程（撮合、仓位记录、统计输出）由引擎自动完成。

无需手动管理事件循环、仓位状态或净值计算，写策略像写交易脚本一样简单。

---

## 策略视角的工作流

1. **引擎喂给策略一个 `BarContext`**：包含本根 bar 的行情（单标或多标）、账户现金、当前持仓均价、净值等。
2. **策略在 `on_bar` 中写业务逻辑**：直接调用 Helper 下单，或返回一个简单的订单列表。
3. **引擎在同一根 bar 内完成执行**：
   - 先处理卖单释放仓位/现金。
   - 再处理买单，可按手数、金额或目标权重下单。
   - 自动计算滑点、佣金和成交价（默认使用收盘价）。
4. **引擎更新 `PortfolioState` 并立刻刷新 `ctx`**：让策略在下一根 bar 看到新的仓位与现金。
5. **回测结束后**：引擎生成净值曲线、成交列表、指标统计，策略作者无需自行整理。

---

## 核心组件（策略友好视角）

| 组件 | 简化职责 |
| --- | --- |
| `SimpleDataFeed` | 预载基准 K 线，按时间逐根返回行情快照。 |
| `BarContext` | 每根 bar 的状态快照，包含 `bar`, `position`, `cash`, `equity`, `symbols`, `indicators`。 |
| `OrderHelper` | 提供 `buy` 与 `sell` 两个 API，结合 `quantity_type ∈ {"count","cash","weight"}` 实现按手数/金额/目标权重下单。 |
| `ExecutionEngine` | 同一根 bar 内顺序撮合：卖单 → 买单；转换金额/权重指令，计算滑点与佣金。 |
| `PortfolioState` | 管理现金、持仓、均价、浮亏、成交记录；拒绝超仓位卖出。 |
| `MetricsRecorder` | 累积净值、回撤、成交列表，回测结束后输出结果。 |

Rust 侧仍负责性能关键路径（数据预提取、状态更新、指标计算），但这些细节不再暴露给策略端。

---

## 下单能力

### OrderHelper（买/卖 + 模式）

| 方法 | 参数 | 描述 | 示例 |
| --- | --- | --- | --- |
| `ctx.order.buy(symbol=None, quantity=1.0, quantity_type="count")` | `quantity_type` 可取：`count`（手数，默认）、`cash`（投入金额）、`weight`（目标组合权重，0~1，自动换算缺口手数） | 按指定模式买入或增持某标的 | `ctx.order.buy("513500.SH", quantity=50)`；`ctx.order.buy("513500.SH", quantity=20000, quantity_type="cash")`；`ctx.order.buy("513500.SH", quantity=0.3, quantity_type="weight")` |
| `ctx.order.sell(symbol=None, quantity=1.0, quantity_type="count")` | `quantity_type` 语义同上；`quantity` 表示要减仓的手数/金额，或目标权重（0~1） | 按指定模式卖出或减仓 | `ctx.order.sell("513500.SH", quantity=10)`；`ctx.order.sell("513500.SH", quantity=10000, quantity_type="cash")`；`ctx.order.sell("513500.SH", quantity=0.0, quantity_type="weight")` |

- 卖单先执行，买单后执行，确保现金与仓位约束。
- `quantity_type="weight"` 支持一次传入字典批量调仓：`ctx.order.buy({"513500.SH": 0.3, "MSFT": 0.2}, quantity_type="weight")`（可在实现中扩展语法）。
- 所有模式自动应用滑点、佣金并进行风险校验。

---

## 多标的支持

- `BarContext` 中的 `ctx.bars` 是 `dict[symbol] -> BarSnapshot`。
- `ctx.positions` 返回每个标的的 `position`, `avg_cost`, `market_value`。
- `ctx.order` Helper 支持传入 `symbol` 参数，默认使用主标的。
- 支持在 `on_bar` 中一次性做多资产再平衡，不需要手动同步时间线。

---

## 性能保持简单同时高效

- 行情批量预提取+顺序遍历，保证缓存友好。
- Rust 内核维护单线程主循环，避免锁；统计分析可在回测结束后用 Rayon 并行。
- Python 与 Rust 仅在每根 bar 交换一次 `BarContext`；上下文对象复用以减少分配。
- 数量换算（金额/权重）在 Rust 内部完成，避免策略端重复计算。
- 指标在初始化阶段统一计算并缓存，回测循环中直接读取，显著降低重复计算的成本。

---

## 扩展能力（可选）

- **高级撮合**：可在配置中切换 `execution_mode = "close" | "open" | "vwap"`，默认仍是收盘价。
- **风险规则**：默认启用“仓位必须≥卖出量”；可选启用“最大仓位/资金占比”等。
- **交割周期配置**：通过引擎初始化参数 `t0_symbols={"510300","159915"}` 等指定 T+0 ETF，其他标的仍按默认 T+1；撮合逻辑会基于此决定当日是否允许卖出。
- **自定义指标注入**：策略可在初始化阶段注册指标，由引擎在 Rust 内侧批量计算后注入 `ctx.indicator`。
- **事件钩子**：提供 `on_start`、`on_bar`、`on_trade`、`on_stop` 等回调，策略可按需实现。

---

## 策略接口概览

```python
class Strategy:
    def on_start(self, ctx): ...
    def on_bar(self, ctx): ...
    def on_trade(self, fill, ctx): ...
    def on_stop(self, ctx): ...
```

`ctx` 结构：

- `ctx.datetime`
- `ctx.bar`（单标）或 `ctx.bars[symbol]`
- `ctx.period`（当前数据频率，例 "1d" / "1h"）
- `ctx.cash`
- `ctx.equity`
- `ctx.position` / `ctx.positions[symbol]["position"]`，并包含 `weight`、`avg_cost` 等字段
- `ctx.benchmark`（当前基准净值、收益等）
- `ctx.state`（策略持久化字典，生命周期内共享）
- `ctx.calendar`（交易日工具，支持 `is_rebalance_day` 等）
- `ctx.order`（下单 Helper）
- `ctx.indicator`（策略注册的指标结果）
- `ctx.factors`（若初始化传入因子数据）
- `ctx.is_tradable(symbol)`（辅助过滤停牌/不可交易标的）

---

## 策略实现伪代码结构

```text
# 1. 引擎初始化
symbols = ["513500.SH", "159941.SZ", "518880.SH", "511090.SH", "512890.SH"]  # 五大资产 ETF
data = load_market_data(symbols, start, end)
indicators = precompute_indicators(data, [
    ("momentum_60d", calc_period_return, 60),
    ("momentum_120d", calc_period_return, 120),
    ("volatility_20d", calc_realized_vol, 20),
])
benchmark_curve = load_benchmark("000300.SH")
engine = BacktestEngine(
    symbols=symbols,
    data=data,
    indicators=indicators,
    benchmark=benchmark_curve,
    period="1d",              # 明确当前是日频数据，可换成 "1h"/"5m" 等
    initial_cash=1_000_000,
    t0_symbols={"513500.SH", "159941.SZ"},  # 可选：支持 T+0 的 ETF
    slippage=0.0005,
    commission=0.0003,
    rebalance_freq="weekly",  # 引擎可辅助调仓节奏（也可在策略内控制）
)

# 2. 动量轮动策略
class MomentumRotation(Strategy):
    def on_start(self, ctx):
        ctx.log.info("动量轮动策略启动，资金: %f", ctx.cash)
        ctx.log.info("数据周期: %s", ctx.period)
        ctx.state["last_rebalance"] = None
        ctx.state["top_k"] = 2

    def on_bar(self, ctx):
        bars = ctx.bars
        indicators = ctx.indicator
        positions = ctx.positions

        # Step 1: 判断是否到达调仓窗口（例如每周最后一个交易日）
        if not ctx.calendar.is_rebalance_day("weekly", ctx.datetime, ctx.state["last_rebalance"]):
            return

        # Step 2: 计算动量得分（组合 60D、120D 收益，并惩罚高波动）
        scores = {}
        for symbol in ctx.symbols:
            if not ctx.is_tradable(symbol):
                continue
            m60 = indicators["momentum_60d"][symbol]
            m120 = indicators["momentum_120d"][symbol]
            vol = indicators["volatility_20d"][symbol]
            scores[symbol] = 0.6 * m60 + 0.4 * m120 - 0.2 * vol

        # Step 3: 选出动量排名前 K 的资产
        top_k = ctx.state["top_k"]
        winners = sorted(scores, key=scores.get, reverse=True)[:top_k]
        if not winners:
            return

        target_weight = 1.0 / len(winners)

        # Step 4: 先卖掉非赢家资产
        for symbol, pos in positions.items():
            if symbol not in winners and pos["position"] > 0:
                ctx.order.sell(symbol=symbol, quantity=0.0, quantity_type="weight")

        # Step 5: 为赢家资产分配等权（或自定义权重）
        for symbol in winners:
            ctx.order.buy(symbol=symbol, quantity=target_weight, quantity_type="weight")

        ctx.state["last_rebalance"] = ctx.datetime

    def on_trade(self, fill, ctx):
        ctx.log.info(
            "成交回报 symbol=%s side=%s qty=%f price=%f fee=%f order=%s",
            fill.symbol,
            fill.side,
            fill.filled_quantity,
            fill.price,
            fill.fee,
            fill.order_id,
        )

    def on_stop(self, ctx):
        ctx.log.info("回测结束，净值: %f", ctx.equity)
        ctx.log.info("基准净值: %f，超额收益: %f", ctx.benchmark.final_nav, ctx.equity / ctx.benchmark.final_nav - 1)

# 3. 运行回测
result = engine.run(MomentumRotation())
render_report(result)
```

---

## 多因子选股策略伪代码

```text
# 1. 数据与因子准备
universe = load_universe("CSI500")
data = load_market_data(universe, start, end)
factors = compute_factors(data, [
    ("quality", calc_quality_factor),
    ("momentum", calc_momentum_factor),
    ("volatility", calc_volatility_factor),
])

engine = BacktestEngine(
    symbols=universe,
    data=data,
    factors=factors,
    period="1d",
    initial_cash=5_000_000,
    slippage=0.0008,
    commission=0.0005,
    max_position_per_symbol=0.05,  # 单票不超过 5%
)

# 2. 策略实现
class MultiFactorLongOnly(Strategy):
    def on_start(self, ctx):
        ctx.log.info("多因子策略启动，标的数量: %d，数据周期: %s", len(ctx.symbols), ctx.period)
        ctx.state["target_count"] = 50

    def on_bar(self, ctx):
        # Step 1: 取出最新因子得分并做标准化/合成
        scores = {}
        for symbol in ctx.symbols:
            if not ctx.is_tradable(symbol):
                continue
            quality = ctx.factors["quality"][symbol]
            momentum = ctx.factors["momentum"][symbol]
            volatility = ctx.factors["volatility"][symbol]
            scores[symbol] = 0.4 * quality + 0.4 * momentum - 0.2 * volatility

        # Step 2: 选出前 N 名组成持仓目标
        top_symbols = sorted(scores, key=scores.get, reverse=True)[: ctx.state["target_count"]]
        if not top_symbols:
            return

        target_weight = 1.0 / len(top_symbols)

        # Step 3: 先卖掉不在目标名单中的持仓
        for symbol, pos in ctx.positions.items():
            if symbol not in top_symbols and pos["position"] > 0:
                ctx.order.sell(symbol=symbol, quantity=0.0, quantity_type="weight")

        # Step 4: 为入选标的按等权买入
        buy_plan = {symbol: target_weight for symbol in top_symbols}
        ctx.order.buy(buy_plan, quantity_type="weight")

    def on_trade(self, fill, ctx):
        ctx.log.info(
            "成交回报 symbol=%s side=%s qty=%f price=%f fee=%f",
            fill.symbol,
            fill.side,
            fill.filled_quantity,
            fill.price,
            fill.fee,
        )

    def on_stop(self, ctx):
        ctx.log.info("策略结束，总资产: %f", ctx.equity)

# 3. 运行与结果
result = engine.run(MultiFactorLongOnly())
render_factor_report(result, factors=["quality", "momentum", "volatility"])
```

---

## 资产配置按月再平衡伪代码

```text
# 1. 初始化（例：股债商品现金四类资产）
symbols = ["510300.SH", "511880.SH", "518880.SH", "511990.SH"]
data = load_market_data(symbols, start, end, period="1d")
indicators = precompute_indicators(data, [
    ("exp_return_63d", calc_expected_return, 63),
    ("cov_63d", calc_covariance_matrix, 63),
])
engine = BacktestEngine(
    symbols=symbols,
    data=data,
    indicators=indicators,
    period="1d",
    initial_cash=2_000_000,
    commission=0.0002,
    slippage=0.0004,
    rebalance_freq="monthly",
)

# 2. 策略实现
class MonthlyAssetAllocation(Strategy):
    def on_start(self, ctx):
        ctx.log.info("资产配置策略启动，周期: %s", ctx.period)
        ctx.state["last_rebalance"] = None
        ctx.state["max_weight"] = 0.5

    def on_bar(self, ctx):
        if not ctx.calendar.is_rebalance_day("monthly", ctx.datetime, ctx.state["last_rebalance"]):
            return

        exp_return = ctx.indicator["exp_return_63d"]
        cov_matrix = ctx.indicator["cov_63d"]

        # 使用简单的风险平价或均值方差求解目标权重
        target_weights = solve_risk_parity(exp_return, cov_matrix)

        # 约束：单资产不超过 max_weight，且权重非负
        max_weight = ctx.state["max_weight"]
        for symbol, weight in target_weights.items():
            target_weights[symbol] = max(0.0, min(max_weight, weight))

        # 归一化权重并拆分卖/买
        total = sum(target_weights.values())
        if total <= 0:
            return
        target_weights = {k: v / total for k, v in target_weights.items()}

        sell_plan = {}
        buy_plan = {}
        for symbol, target in target_weights.items():
            current = ctx.positions.get(symbol, {}).get("weight", 0.0)
            diff = target - current
            if diff < 0:
                sell_plan[symbol] = abs(diff)
            elif diff > 0:
                buy_plan[symbol] = diff

        # 卖出不在目标内的资产
        for symbol, pos in ctx.positions.items():
            if symbol not in target_weights and pos["position"] > 0:
                sell_plan[symbol] = sell_plan.get(symbol, 0.0) + pos["weight"]

        if sell_plan:
            ctx.order.sell(sell_plan, quantity_type="weight")
        if buy_plan:
            ctx.order.buy(buy_plan, quantity_type="weight")

        ctx.state["last_rebalance"] = ctx.datetime

    def on_trade(self, fill, ctx):
        ctx.log.info(
            "成交回报 symbol=%s side=%s qty=%f price=%f fee=%f order=%s",
            fill.symbol,
            fill.side,
            fill.filled_quantity,
            fill.price,
            fill.fee,
            fill.order_id,
        )

    def on_stop(self, ctx):
        ctx.log.info("回测结束，资产净值: %f", ctx.equity)

# 3. 运行与评估
result = engine.run(MonthlyAssetAllocation())
render_allocation_report(result)
```

---

## 回测流程图

```text
加载数据 → for bar in timeline:
    ctx = build_context(bar, portfolio)
    orders = strategy.on_bar(ctx)
    execution_engine.handle(orders, ctx)
    portfolio.update()
    metrics.record(portfolio, bar.datetime)
生成结果 (returns, trades, equity_curve)
```

---

## 后续步骤

1. 在 Rust 内实现 `BarContext`、`OrderHelper`、`ExecutionEngine`，保持单根 bar 顺序执行。
2. 更新 Python 包的策略基类，提供新的 Helper API。
3. 编写文档与示例策略，展示金额下单、目标权重调仓、同一 bar 卖后买的场景。
4. 建立性能基准，确认新设计与旧实现相比性能无回退。

以上即“Strategy First”版回测核心设计，旨在让策略作者“拿到 K 线就能下单”，同时保留引擎的性能与可扩展性。*** End Patch
