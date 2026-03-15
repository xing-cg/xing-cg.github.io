---
title: xmrig_编译_修改捐赠配置
categories:
  - - 量化交易
tags:
date: 2026/2/6
updated: 2026/2/6
comments:
published:
---
# 挖矿的来龙去脉
2026 年，狂潮退却了。

1. 核心原因是以太坊合并。
    - **机制改变：** 2022年9月，以太坊（ETH）完成了从 **PoW（工作量证明）** 到 **PoS（权益证明）** 的升级。在此之前，ETH是显卡挖矿绝对的主力军（因为比特币早已用专业ASIC矿机挖，普通显卡挖不动）。
    - **算力失效：** 升级后，以太坊不再需要显卡进行复杂的哈希计算来维护网络，显卡瞬间失去了最大的“用武之地”。 
    - **无处可去：** 虽然还有其他小币种（如ETC、RavenCoin等）可以挖，但它们的市值和流动性根本承载不了从ETH溢出的巨大算力，导致挖这些币的收益极低，甚至不够电费。
2. 政策原因：中国政府的全面禁令
   中国曾是全球最大的算力中心，政策转向对矿圈打击巨大。
    - **断电断网：** 2021年5月起，中国国务院金融委明确提出“打击比特币挖矿和交易行为”。随后，内蒙古、四川、新疆等水电/火电资源丰富的省份开始清理矿场，直接切断电力供应。
    - **定性非法：** 2021年9月，发改委等部门联合发布通知，将虚拟货币“挖矿”列为淘汰类产业，全面禁止新增项目，清理存量项目。这使得国内的大型矿场被迫关停或向海外（如美国、哈萨克斯坦）转移，中小矿工则因风险剧增而退场。
3. 经济原因：币价崩盘与能源成本
    - **加密寒冬：** 2022年开始，受美联储加息、Terra (LUNA) 崩盘、FTX交易所暴雷等一系列黑天鹅事件影响，加密货币市场进入深熊。以太坊价格从最高点的4800美元一路狂泻至1000美元以下，挖矿收益直接腰斩再腰斩。
    - **能源危机：** 俄乌冲突导致全球（特别是欧洲）能源价格飙升。电费是挖矿最大的成本，当“币价下跌”遇上“电费暴涨”，挖矿彻底变成了赔本生意

## PoS（Proof of Stake, 权益证明）是什么
它是区块链的一种**共识机制**（即决定谁有资格记账并获得奖励的规则）。
把它和之前的 **PoW（工作量证明）** 做一个生动的对比：
- **PoW（挖矿模式）：比谁力气大**
    - **规则：** 就像是一群人在解一道超难的数学题。谁的电脑（矿机/显卡）算得快，谁就先解出来，谁就能获得记账权和奖励（比特币/以前的以太坊）。
    - **后果：** 大家为了赢，疯狂买显卡、耗费巨量电力。这就是“矿潮”的根源。
    - **门槛：** 拼硬件、拼电费。
- **PoS（质押模式）：比谁钱多**
    - **规则：** 就像是银行存款吃利息。你想参与记账？不需要买显卡，只需要把你持有的币（比如ETH）“抵押/存”在系统里。系统会根据你存的钱多少、存的时间长短，随机抽签选人来记账。存的越多，被选中的概率越大，获得的奖励也越多。
    - **后果：** 既然是比谁存的币多，那就完全**不需要显卡**去进行复杂的计算了。
    - **门槛：** 拼资产（币的数量）。
### PoS 机制的核心逻辑
1. **质押（Staking）：** 验证者必须将一定数量的加密货币锁定在网络中作为“保证金”。（例如以太坊要求质押 32 个 ETH 才能成为独立验证者）。
2. **验证：** 系统不再需要全网算力竞赛，而是随机选择验证者来打包区块。
3. **奖惩：**
    - **奖励：** 如果你诚实记账，你会获得系统发的币作为利息。
    - **惩罚（Slashing）：** 如果你作恶（比如乱记账），你的保证金会被系统没收。

在以太坊从 PoW 切换到 PoS 之后（即 The Merge 升级）：
- **显卡彻底下岗：** 以前矿工买显卡是为了提供算力去“解题”。现在规则变了，不需要算力了，只要有币就行。显卡对于以太坊网络变得**毫无用处**。
- **能源消耗骤降：** 由于不需要显卡全负荷运转，以太坊网络的能耗瞬间降低了 **99.95%**。

总结：**PoS 就是一种“用资产说话”的制度。** 对于矿工来说，这不仅是玩法的改变，更是**生产资料的彻底变更**——从通过购买**显卡**（硬件）来赚钱，变成了通过购买**币**（资产）来赚钱。

# 2026 年，还能挖矿吗？


总体流程：
```bash
git clone https://github.com/xmrig/xmrig.git
mkdir xmrig/build && cd xmrig/build
cmake ..
make -j$(nproc)
```

需要修改的文件：
```bash
src/donate.h                     # 修改默认捐赠等级常量
src/core/config/Config_default.h # 修改嵌入式配置的默认值
src/core/config/usage.h          # 修改帮助信息中的默认值说明
```
donate.h
```cpp
constexpr const int kDefaultDonateLevel = 0;
constexpr const int kMinimumDonateLevel = 0;
```
Config_default.h
改的地方：`"donate-level": 0`
```cpp
#ifndef XMRIG_CONFIG_DEFAULT_H
#define XMRIG_CONFIG_DEFAULT_H
namespace xmrig {
// This feature require CMake option: -DWITH_EMBEDDED_CONFIG=ON
#ifdef XMRIG_FEATURE_EMBEDDED_CONFIG
const static char *default_config =
R"===(
{
    "api": {
        "id": null,
        "worker-id": null
    },
    "http": {
        "enabled": false,
        "host": "127.0.0.1",
        "port": 0,
        "access-token": null,
        "restricted": true
    },
    "autosave": true,
    "background": false,
    "colors": true,
    "title": true,
    "randomx": {
        "init": -1,
        "init-avx2": -1,
        "mode": "auto",
        "1gb-pages": false,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": false,
        "yield": true,
        "max-threads-hint": 100,
        "asm": true,
        "argon2-impl": null,
        "cn/0": false,
        "cn-lite/0": false
    },
    "opencl": {
        "enabled": false,
        "cache": true,
        "loader": null,
        "platform": "AMD",
        "adl": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "cuda": {
        "enabled": false,
        "loader": null,
        "nvml": true,
        "cn/0": false,
        "cn-lite/0": false
    },
    "donate-level": 0,
    "donate-over-proxy": 1,
    "log-file": null,
    "pools": [
        {
            "algo": null,
            "coin": null,
            "url": "donate.v2.xmrig.com:3333",
            "user": "YOUR_WALLET_ADDRESS",
            "pass": "x",
            "rig-id": null,
            "nicehash": false,
            "keepalive": false,
            "enabled": true,
            "tls": false,
            "tls-fingerprint": null,
            "daemon": false,
            "socks5": null,
            "self-select": null,
            "submit-to-origin": false
        }
    ],
    "print-time": 60,
    "health-print-time": 60,
    "dmi": true,
    "retries": 5,
    "retry-pause": 5,
    "syslog": false,
    "tls": {
        "enabled": false,
        "protocols": null,
        "cert": null,
        "cert_key": null,
        "ciphers": null,
        "ciphersuites": null,
        "dhparam": null
    },
    "user-agent": null,
    "verbose": 0,
    "watch": true,
    "pause-on-battery": false,
    "pause-on-active": false
}
)===";
#endif
} // namespace xmrig
#endif /* XMRIG_CONFIG_DEFAULT_H */
```
usage.h
把
```cpp
    u += " --donate-level=N donate level, default 1%% (1 minute in 100 minutes)\n";
```
改为
```cpp
    u += " --donate-level=N donate level, default 0%% (disabled)\n";
```
