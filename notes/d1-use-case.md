# Reactive 用例拆解卡：巨鲸大额转账监控 (Whale Tracker)

## 1. 监听哪条链、哪个合约的什么事件？
- **源链**：Ethereum Sepolia 测试网
- **合约**：某个特定的 ERC-20 代币合约（例如 USDC）
- **事件**：`Transfer(from, to, value)`

## 2. 满足什么条件时触发？
- 在 `react()` 函数中进行判断：
- `from` 地址是我们要监控的“巨鲸”地址。
- `value`（转账金额）大于设定的阈值（例如 10,000 代币）。
- **decimals 换算**：USDC 是 6 位小数，链上 `value` 是整数，阈值 10,000 USDC 要写成 `10_000 * 10**6 = 1e10`；不同代币精度不同（USDT 6 位、DAI 18 位），比较前必须先确认代币的 decimals，不能直接拿“人类读数的金额”比。

## 3. 在哪条链做什么动作？
- **目标链**：Base Sepolia（统一用它；Reactive 侧在 Lasna 上订阅和跑 `react()`，Lasna 不是目标链）
- **动作**：调用目标链上的回调合约（Callback），触发一个 `Alert` 事件，在链上永久记录一条“巨鲸异动”日志，以便后续链下程序读取并推送到 Telegram。
- **目标链费用**：回调由 Base Sepolia 的回调代理收费，公式与 Reactive 侧相同 —— `fee = max(gasprice, basefee) × (系数/1000) × (实际消耗 gas + extraGas)`；回调 gas limit 最低 100,000，代理还要预留 `_extraGas` 和 `_maxChargeGas`，所以回调合约必须提前用 Base Sepolia 的 ETH 充值（`depositTo`），余额不足回调会被拒绝。

## 4. 不用 Reactive 的痛点 vs Reactive 的优势与代价
- **不用 Reactive 怎么做**：需要自己租用云服务器，写 Node.js/Python 脚本去 7×24 小时监听链上事件。服务器可能宕机，且要把私钥暴露在服务器环境变量里，有安全隐患。
- **Reactive 简单在哪**：不用写链下代码，不用跑机器人。只需写一个合约订阅事件，交由 Reactive Network 底层代劳。
- **代价是什么**：Reactive 合约的 reserves 一旦扣完就会产生 debt，**只要 debt > 0，合约立即停止执行**（`trigger()` 的第一步就是检查 debt，跳过整个 `react()`，状态变 inactive 并发 `BlacklistContract` 事件）；必须充值（`depositTo()` 或 `pay()`）把 debt 清零才恢复（`WhitelistContract`）。此外跨链回调有一定的网络延迟，无法做到绝对实时（不适用于秒级套利）。

## 🗺️ 链路简图

```text
[源链: Sepolia]                    [Reactive Network]                  [目标链: Base Sepolia]
ERC-20 发生 Transfer  ──监听──▶  react() 检查金额与地址 ──回调请求──▶ Callback Proxy
(大额代币转出)                     (是否大于阈值?)                      └─▶ 触发 Alert 事件
```
