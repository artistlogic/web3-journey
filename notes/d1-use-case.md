# Reactive 用例拆解卡：巨鲸大额转账监控 (Whale Tracker)

## 1. 监听哪条链、哪个合约的什么事件？
- **源链**：Ethereum Sepolia 测试网
- **合约**：某个特定的 ERC-20 代币合约（例如 USDC）
- **事件**：`Transfer(from, to, value)`

## 2. 满足什么条件时触发？
- 在 `react()` 函数中进行判断：
- `from` 地址是我们要监控的“巨鲸”地址。
- `value`（转账金额）大于设定的阈值（例如 10,000 代币）。

## 3. 在哪条链做什么动作？
- **目标链**：Base Sepolia（或 Reactive Lasna）
- **动作**：调用目标链上的回调合约（Callback），触发一个 `Alert` 事件，在链上永久记录一条“巨鲸异动”日志，以便后续链下程序读取并推送到 Telegram。

## 4. 不用 Reactive 的痛点 vs Reactive 的优势与代价
- **不用 Reactive 怎么做**：需要自己租用云服务器，写 Node.js/Python 脚本去 7×24 小时监听链上事件。服务器可能宕机，且要把私钥暴露在服务器环境变量里，有安全隐患。
- **Reactive 简单在哪**：不用写链下代码，不用跑机器人。只需写一个合约订阅事件，交由 Reactive Network 底层代劳。
- **代价是什么**：Reactive 合约里的 lREACT 余额一旦耗尽，服务会静默停止；且跨链回调有一定的网络延迟，无法做到绝对实时（不适用于秒级套利）。

## 🗺️ 链路简图

```text
[源链: Sepolia]                    [Reactive Network]                  [目标链: Base Sepolia]
ERC-20 发生 Transfer  ──监听──▶  react() 检查金额与地址 ──回调请求──▶ Callback Proxy
(大额代币转出)                     (是否大于阈值?)                      └─▶ 触发 Alert 事件
