# D2 · 读懂费用｜Lasna 系统合约三参数实测

> 链：Lasna 测试网 `chainId 5318007`
> 系统合约：`0x8888888888888888888888888888888888888888`
> RPC：`https://lasna-rpc.rnk.dev/`
> 日期：2026-11-17

---

## 一、三条 cast 命令与真实输出

```bash
cast call 0x8888888888888888888888888888888888888888 "_gasPriceCoeffPer1000()(uint256)" --rpc-url https://lasna-rpc.rnk.dev/
# 1000

cast call 0x8888888888888888888888888888888888888888 "_extraGas()(uint256)" --rpc-url https://lasna-rpc.rnk.dev/
# 100000 [1e5]

cast call 0x8888888888888888888888888888888888888888 "_maxChargeGas()(uint256)" --rpc-url https://lasna-rpc.rnk.dev/
# 50000 [5e4]
```

| 参数 | 实测值（hex） | 实测值（dec） | 含义 |
|---|---|---|---|
| `_gasPriceCoeffPer1000()` | `0x3e8` | 1000 | 价格系数，千分制：1000 ÷ 1000 = **1.0 倍** |
| `_extraGas()` | `0x186a0` | 100,000 | 每次执行的**固定附加 gas**（进门费） |
| `_maxChargeGas()` | `0xc350` | 50,000 | **结算 / `pay()` 的 gas 预算上限**，不进费用公式 |

---

## 二、一次 react() 的费用怎么算？

官方公式（Economy 文档）：

```text
fee = max(tx.gasprice, block.basefee) × (_gasPriceCoeffPer1000 ÷ 1000) × (g_used + _extraGas)
```

代入实测值（系数 1000 ÷ 1000 = 1.0）：

```text
fee = max(tx.gasprice, block.basefee) × (实际消耗 gas + 100,000)
```

算例：基础 gas 价格 1 Gwei，`react()` 逻辑实际消耗 30,000 gas：

```text
计费 gas = 30,000 + 100,000 = 130,000
fee = 130,000 × 1 Gwei × 1.0 = 0.00013 ETH
```

注意 `_maxChargeGas`（50,000）**不参与**这个公式：它是系统合约在你欠费时调用 `pay()` 结算所允许消耗的 gas 预算上限，属于“收账的手续成本”，不是费用本身。

---

## 三、balance、reserves、debt 分别是什么？为什么有 debt 就停？

| 概念 | 谁拿着 | 通俗解释 |
|---|---|---|
| **balance** | 合约自己 | 合约钱包里的真金白银，直接转账就会增加；但它不会自动交费，合约要主动 `pay()` / `_coverDebt()` 把钱交给系统合约 |
| **reserves** | 系统合约（Reactive 侧）/ 回调代理（目标链侧） | 预付费余额，用 `depositTo()` 充值；每次扣费**先扣 reserves** |
| **debt** | 系统合约 / 回调代理 | 已经发生、但还没付的钱；reserves 扣光了才产生 debt |

规则：**扣费先扣 reserves，欠了才记 debt；充值先还 debt，剩下的才进 reserves**，所以两者不会同时存在。

**为什么“只要有一点 debt，合约就停止执行”？**

- `trigger()` 的**第一步就是查 debt**，在任何 gas 花出去之前执行；只要 `debt > 0` 直接跳过，连 `react()` 都不会被调用。
- 这是网络的止损阀：Reactive 采用“先扣费、后干活”模型，网络不替任何人垫付。有欠款还继续跑，只会越欠越多，风险会转嫁给整个验证网络，所以一刀切断。
- 副作用是**静默停服**：合约不会大声报错，只是“死掉”——状态从 `active` 变 `inactive`，链上发 `BlacklistContract` 事件；把 debt 清零（`depositTo()` 或 `pay()`）才恢复，恢复时发 `WhitelistContract` 事件。

---

## 四、react() 回滚了，还收费吗？对设计意味着什么？

**收费，而且全额收费。**

收费发生在**执行之后**，按实际消耗的 gas 计算。即使 `react()` 里 `revert`，系统合约会捕获回滚、记录一条 `ReactiveContractReverted` 事件，然后照常扣费，不退款。唯一例外：回调发往一个**没有代码的地址**，那笔会在收费前被拒绝。

对合约设计的四个含义：

1. **“不该处理”用 `return`，不要用 `revert`。** 金额不到阈值、不是订阅的事件、次数用完——都是正常业务判断，静默退出即可；`revert` 除了白交钱没有任何收益。
2. **订阅要窄。** 每次 `react()` 都是一笔真实开销，先靠 `chainId + 合约地址 + topic0`（必要时 topic1 的 indexed 参数）过滤；别用 `address(0)` 订阅全量 Transfer，否则一天几万次、每次 10 万 gas 起步。
3. **给自动化上刹车。** 用次数上限（`_limit`）和额度上限，用完主动 `unsubscribe`；否则余额耗尽后合约静默停服，等于自动化悄悄死了。
4. **重逻辑放回调侧，但回调也收费。** 目标链回调代理按同一公式收费，且回调 gas limit 最低 100,000，代理还要预留 `_extraGas` 和 `_maxChargeGas`；所以回调要便宜、要幂等，别让回调合约欠费停摆。

一句话：**react() 的每一次“看一眼”都要钱，回滚的“看一眼”同样要钱——所以让它看得准、看得少、看得便宜。**
