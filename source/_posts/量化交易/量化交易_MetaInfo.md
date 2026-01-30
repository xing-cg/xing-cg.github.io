---
title: 量化交易_MetaInfo
categories:
  - - 量化交易
tags:
date: 2025/11/6
updated: 2025/11/6
comments:
published:
---
合约交易 MetaInfo 字段解释

# spot 现货

# swap 合约

在金融领域，**Swap**的中文是“**互换**”或“**掉期**”。它是一种重要的金融衍生工具，核心可以概括为：**交易双方在预先约定的时间内，相互交换一系列现金流的合约**。

下表总结了互换的主要类型和核心特征。

| 互换类型       | 交换的标的物                       | 主要目的              |
| ---------- | ---------------------------- | ----------------- |
| **利率互换**   | 同种货币的利息支付（如固定利率利息 vs 浮动利率利息） | 管理利率风险，降低融资成本     |
| **货币互换**   | 不同货币的本金和利息                   | 管理汇率风险，转换资产/负债的币种 |
| **商品互换**   | 与商品价格（如石油、黄金）挂钩的现金流          | 对冲商品价格波动的风险       |
| **信用违约互换** | 针对特定债务的信用风险（类似保险）            | 转移和规避信用风险         |
与其他衍生品的区别
作为金融衍生品家族的四大支柱之一（另外三者是期货、期权和远期），互换有其独特性：
与期货/远期相比：期货和远期通常涉及在未來某个时点交割一项资产，而互换则是在一段时间内分期、多次地交换现金流。
与期权相比：期权赋予持有者“选择执行与否”的权利，而互换则是一种“承诺”，双方都必须按约定履行支付义务。
# 概念
## VolumeMultiple 是什么？
短答：
- **VolumeMultiple**（有些文档叫“合约乘数/数量乘数”）=「每**1 手/1 张**合约包含的**标的数量**」。这是交易规则里的一个“乘数”。([Shanghai Futures Exchange](https://www.shfe.com.cn/services/technology/technical_download/202007/P020240320687787646086.pdf?utm_source=chatgpt.com))
- **现货 BTC-USDT**：没有“张/手”的概念，数量直接用 **BTC 数量**计。很多带这个字段的通用 API 对现货会把 **VolumeMultiple 设为 1**（占位），基本用不到。([dict.thinktrader.net](https://dict.thinktrader.net/dictionary/future.html?utm_source=chatgpt.com))
- **合约（永续/交割）**：VolumeMultiple 才有意义，常与“**合约面值/contractSize**”一起决定 1 张代表多少标的或多少美元。

**那“VolumeMultiple 就是 contractSize”吗？**

**不完全等同**。从定义上看：
- **VolumeMultiple/合约乘数** = 每张（或每手）里包含的“**单位数**”（用于把“张数”换成“标的数量”）。
- **contractSize/合约面值（或 ctVal）** = 每张对应的“**标的数量（BTC）**”或“**美元面值**”。

- 在很多加密合约里 **ctMult=1**，这时 **“每张有多少标的”** 看起来就和 contractSize 一样，所以有人会把它们当成一个概念；但在**有单独面值字段**（例如 OKX 的 **ctVal** 与 **ctMult**）或**反向合约以 USD 面值计**的场景，两者**不是同一字段**，计算也不同。([OKX](https://www.okx.com/docs-v5/zh/?utm_source=chatgpt.com))


# 现货 vs. 合约怎么对应？

## 现货 BTC-USDT

- 下单数量 = 你买/卖的 **BTC 数量**。
- VolumeMultiple 通常 = **1**；实际下单受 **最小下单数量/步进** 约束，而不是乘数。([dict.thinktrader.net](https://dict.thinktrader.net/dictionary/future.html?utm_source=chatgpt.com))
## 合约（两种常见规格）

**A. USDT 本位（线性）**

- 平台会给出“**合约面值（contractSize/ctVal）**”，单位通常是 **BTC/张**（比如 1 张 = 0.01 BTC），以及“**合约乘数（ctMult）**”（多数产品为 1）。
- 1 张代表的 **BTC 数量 = contractSize × ctMult**。
- 成交名义价值（USDT）= 价格 × 张数 × contractSize × ctMult。
- 例：OKX 说明里，**BTCUSDT 合约面值 = 0.01 BTC**，**乘数 = 1** ⇒ 1 张 = 0.01 BTC。([OKX](https://www.okx.com/zh-hans/help/i-future-contracts?utm_source=chatgpt.com))

**B. 币本位/反向（如 BTCUSD）**

- 面值以 **USD/张**给出（例如 100 USD/张），乘数多为 1。
- 1 张对应的 **BTC 数量 =（USD 面值 × 乘数）/ 价格**（随价变动）。
- 例：OKX 文档示例，**BTCUSD 面值 = 100 USD**，**乘数 = 1**。([OKX](https://www.okx.com/zh-hant/help/i-future-contracts?utm_source=chatgpt.com))

（补充：Binance USDT-M 合约是线性合约，契合上面 A 的计算思路；其规格文档也强调线性计价/结算。([Binance](https://www.binance.com/en/support/faq/detail/360033161972?utm_source=chatgpt.com))）

- **futures（合约）里的 VolumeMultiple**：在 Gate 就用 **quanto_multiplier**。对「正向/线性」合约（type="direct"）它等价于“每 1 张合约代表的标的数量”（常见是 0.001 BTC/张，但请以 API 返回为准）。对「反向」合约（type="inverse"），估值/盈亏会用到 1/price 的关系，quanto_multiplier 用作计价币→结算币的换算系数（详见下面公式解释）。[Gate.com.tr](https://www.gate.hk/docs/developers/apiv4/zh_CN/)
- **futures（合约）的 PriceTick**：用 **order_price_round**。[Gate.com.tr](https://www.gate.hk/docs/developers/apiv4/zh_CN/)
- **futures 的 MinLimitOrderVolume / MaxLimitOrderVolume**：分别用 **order_size_min** / **order_size_max**。[Gate.com.tr](https://www.gate.hk/docs/developers/apiv4/zh_CN/)
- **futures 的 VolumeTick（数量步进）**：用 **1**。Gate 的合约下单数量（以及持仓 size）是整数张，步进为 1 张。[Gate.com.tr](https://www.gate.hk/docs/developers/apiv4/zh_CN/)
- **futures 的 MinNotional（最小名义金额）**：**Gate 合约的合约详情模型里没有单独给出**。一般以最小张数限制为准；如果你需要“估算的”名义金额，可用

- 线性合约：order_size_min × quanto_multiplier × mark_price；
- 反向合约：order_size_min × quanto_multiplier ÷ mark_price（价值与 1/price 成正比）。  
    但这些是**派生值**，不是官方字段。[Gate.com.tr+1](https://www.gate.hk/docs/developers/apiv4/zh_CN/)

- **spot（现货）的对应关系**（你前面也问到现货）：

- 现货没有“合约乘数/面值”这个概念，**VolumeMultiple 固定看作 1**（买卖就是以基础币的数量计）。
- 现货的 **VolumeTick**（数量步进）可由 **amount_precision** 推出（步进=10^-amount_precision）。
- 现货的 **MinNotional**（最小名义金额）就是 **min_quote_amount**。

  

  

**futures 的 `VolumeTick`（数量步进）**：用 **1**。Gate 的合约下单数量（以及持仓 `size`）是整数张，步进为 1 张。

这是从哪得出来来的

  

**ChatGPT said:**

  

Thought for 44s

  

对的，我说 “**VolumeTick=1（按 1 张为步进）**” 的依据就是 **Gate 官方 API 文档**里对合约“数量”的类型定义：

- **下单与订单**：FuturesOrder.size 明确是 **integer(int64)**，并写明“下单数量（size）为正表示买，负表示卖”，说明只能用**整数张**，不存在小数张。[Gate.com](https://www.gate.com/docs/futures/api/index.html)
- **仓位**：Position.size 也是 **integer(int64)**，仓位规模用整数计。[Gate.com](https://www.gate.com/docs/futures/api/index.html)
- **合约规格**：Contract.order_size_min / order_size_max 都是 **integer(int64)**（最小/最大下单张数为整数）。文档没有提供任何“数量精度/步进”字段（只有价格精度 order_price_round），因此**默认步进就是 1 张**。[Gate.com](https://www.gate.com/docs/futures/api/index.html)
- **中文说明**：在中文 API 说明里也写得很直接：**“下单时指定的是合约张数 size，而非币的数量；每张对应的币数量看 quanto_multiplier。”** 这也侧面印证 size 以**张数**、且为整数计。[Gate.com](https://www.gate.io/docs/apiv4/zh_CN/index.html?utm_source=chatgpt.com)

综上：Gate 永续合约的下单与持仓数量都是“**张**”，类型是整数 → 数量步进自然就是 **1 张**。

  

  

  

  

**实操小抄**

- **现货**：直接用 **数量（BTC）× 价格**；忽略 VolumeMultiple。([dict.thinktrader.net](https://dict.thinktrader.net/dictionary/future.html?utm_source=chatgpt.com))
- **USDT 本位合约**：名义价值 = 价格 × 张数 × **contractSize** × **VolumeMultiple**（多为 1）。([OKX](https://www.okx.com/docs-v5/zh/?utm_source=chatgpt.com))
- **币本位/反向合约**：名义价值（USD）= 张数 × **contractSize(USD)** × **VolumeMultiple**；对应 **BTC 数量 = 名义价值 / 价格**。([OKX](https://www.okx.com/zh-hant/help/i-future-contracts?utm_source=chatgpt.com))

如果你告诉我具体是哪家交易所/哪个 API 字段名（例如 OKX 的 ctVal/ctMult、Binance 的 contractSize 等），我可以把它们一一对上，并给你写好计算示例。


# USDT、USDC、USD
| 术语类别      | 术语                 | 核心含义与区别                                                                         |
| --------- | ------------------ | ------------------------------------------------------------------------------- |
| **基础货币**​ | **USD**​           | 美元，美国的官方法定货币，是USDT和USDC的价值锚定对象。                                                 |
|           | **USDT**​          | 由Tether公司发行的稳定币，与美元1:1挂钩。特点是**流动性极高**，交易对最丰富，但储备金透明度和监管合规性历来存在争议。               |
|           | **USDC**​          | 由Circle和Coinbase联合发行的稳定币，同样与美元1:1挂钩。特点是**监管合规性和透明度高**（每月公开审计报告），更受注重安全和合规的机构青睐。 |
| **交易类型**​ | **现货**​            | 直接买卖并持有真实的加密货币（如BTC），一手交钱一手交货。                                                  |
|           | **USDT永续/USDC永续**​ | 使用USDT或USDC作为保证金进行结算的**永续合约**。**没有到期日**，可无限期持有。通过“资金费率”机制使合约价格锚定现货价格。           |
|           | **USDT交割/USDC交割**​ | 使用USDT或USDC作为保证金进行结算的**交割合约**。**有明确的到期日**，到期后合约会自动按约定价格平仓结算。                    |
|           | **反向合约**​          | 也称“币本位合约”。以交易对象本身（如BTC）作为保证金和结算单位。盈亏以该加密货币数量计算。                                 |
|           | **期权**​            | 赋予持有者在未来特定时间以特定价格买入或卖出标的资产的权利，但**不是义务**。买方最大亏损为购买期权时支付的权利金w。                    |