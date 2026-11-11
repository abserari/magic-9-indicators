# Magic-9 神奇九转（TD 序列）指标存档

TradingView 作者 **blackcat1402** 开源的 6 个「神奇九转 / TD Sequential」Pine Script 指标的完整源码存档。

- **原作者**：blackcat1402（TradingView 同名 ID；其主页另注 GitHub: wxt-coder）
- **协议**：Mozilla Public License 2.0（MPL 2.0），每份源码文件头保留原作者的协议声明与署名
- **来源**：2026-11-11 从各脚本 TradingView 页面公开的 Source code 标签页取得（作者被 DeMark 公司投诉后于 2026-10-26 重新上架的 V2 系列）
- **背景**：作者小红书帖《祖师爷亲自投诉我，我一怒之下全免费放送》

## 文件清单

| 文件 | TradingView 标题 | 版本 | 行数 | 一句话逻辑 |
|---|---|---|---|---|
| `01-magic-9-sequential-setup.pine` | L3 Magic-9 Sequential Setup | Pine v5 | ~120 | 基础版：纯连珠计数 5/6/7/8/9/12/13 + 窗口多数方向染 K 线色 |
| `02-magic-9-13-reversal-confirmation.pine` | L3 Magic-9/13 with Reversal Points Confirmation V2 | Pine v6 | ~194 | 计数 + 反转点确认：数到 9 需确认交叉才给 B/S 信号，数满 13 无条件给 |
| `03-projected-magic-9-sequence.pine` | L3 Projected Magic-9 Sequence | Pine v6 | ~260 | 计数 + 高周期投影区间门控：数满且价格触及投影带边缘才触发，带持仓状态机与 ATR 止盈止损 |
| `04-magic-9-macd-setup.pine` | L3 Magic-9 MACD Setup | Pine v6 | ~255 | 把计数搬到 MACD 线上数，另加确认窗口、可选高周期 MACD 斜率过滤、出场后锚点再入场 |
| `05-magic-9-sequential-macd.pine` | L3 Magic-9 Sequential MACD | Pine v6 | ~280 | 在 MACD 线上计数，默认在连珠「结束」的那根出信号（边沿触发），MACD 与信号线交叉出场 |
| `06-magic-9-sequential-highlights.pine` | L3 Magic-9 Sequential within Highlights | Pine v6 | ~308 | 不用 MACD：价格计数 + 自创 Magic-9 指数（类 RSI）超买 70 / 超卖 30 门控才给 BUY/SELL |

## 共用引擎（六份的骨架）

每一根 K 线，拿当前收盘价与 **3 根前的收盘价** 比较（代码里 `get_first_non_na_value(close, 4)` 取的是窗口内最老的一根，即 `close[3]`；MACD 版同理作用在 MACD 线上）。连续同方向就累加计数，断一根清零。计数恰好到 5、6、7、8、12 时打进度标记，到 9 与 13 时打关键标记（上行数满标 ▼ = 涨势耗尽，下行数满标 ▲ = 跌势耗尽），本质是左侧的趋势耗尽计数器。

V2 系列普遍在此之上叠了同一套「信号治理」：持仓状态机（只在空仓/反向时开仓）、同根 K 线入场与出场互斥、信号冷却期（默认 3 根）、出场后锚点再入场（RL/RS，默认关），以及 ATR 14 的 2 倍止损 / 3 倍止盈参考线（只是画线，不是订单）。所有提醒都是每根 K 线收盘触发一次，信号用已收盘数据确认，历史上不重绘。

## 与原版 TD Sequential 的关键差异（读代码前先看这段）

1. **比较基准差一根**：原版 TD 用 4 根前收盘（`close[4]`），这套代码用 3 根前（`close[3]`），作者注释里自己写明。结果是计数比原版稍早、稍密，与通达信神奇九转对图会对不齐。
2. **「13」不是 TD 的 Countdown**：原版 9 之后有一套独立的倒计数规则（与 2 根前高低点比、不要求连续）。这里的 13 只是把同一个连数从 9 接着数到 13。
3. **缺原版部件**：没有 Setup 完善（perfection）、没有 TDST 线、没有起始 flip 条件。信号密度因此高于原版，作者用确认窗口 / 投影门控 / 指数门控来补过滤。

## 各版本独有逻辑备忘

- **02 确认版**：反转点 = 最近 9 根收盘与其各 4 根前收盘比较后全部同向（下行全破 = 看涨反转点 +1）。`sellSignal = (近 2 根内出现过 high9 且看跌反转点刚交叉) or high13`，买入镜像。
- **03 投影版**：投影带用上一根高周期 K 线（日线图→上周）的开高低收按枢轴类公式算出（`X = (H + 2L + C)/2` 或 `(2H + L + C)/2`，再 `ProjHi = X − L`、`ProjLo = X − H`）。默认模式下投影带会随高周期 K 线进展微调；开 `Non-Repainting HTF Values` 后只用上根已收盘值，带固定。
- **04 MACD Setup**：计数对象是 MACD 线本身（`macdLine > macdLine[3]` 连数）；确认窗口默认 4 根；信号还可叠加 240 分钟 MACD 线斜率过滤（默认关）。
- **05 Sequential MACD**：默认信号语义和其余版本相反——不是数满当根给信号，而是等连珠**断掉**的那根（streak-end）才触发；里程碑 6/9/13 都可作触发点；另有 Legacy 开关回到 V1 的逐根重复信号。
- **06 Highlights**：确认指标 Magic-9 Index = 上涨高点差均值 /（上涨高点差均值 + 下跌低点差均值）× 100，均值周期 13，本质是拿高低点差做的类 RSI。序列在最近 8 根内完成 9/13 且指数 >70 才给 SELL、<30 才给 BUY，信号间有 13 根抑制期；BUY/SELL 标签默认画在触发后 1 根（`labelOffset=1`，原版行为）。注意：代码里指数的四行 plot 已被注释掉，指数本身不单独成图，只用于背景色、仪表盘和信号过滤。

## 已知小毛病（审查记录）

- 01 基础版有一段永不可能触发的信号分支（要求 9 次比较中出现 10 次同向）和一组算完未被使用的变量（死代码）；且无任何可调参数、无提醒。
- 全套配色是「反转语义」：买入标签 B 用红色、卖出标签 S 用绿色，第一次看容易读反。
- 04 中有两个 occurrence helper 定义后未被使用；05 的注释里保留了作者修过的两个历史 bug 说明（锚点窗口计数冻结、锚点重复捕获），V2 已修复。

## 转录说明

源码由 TradingView 公开页面文本转录，逻辑部分经多视图交叉核对；01 号文件开头的装饰性 ASCII 艺术字未收入（不影响功能），个别标签行的颜色参数在页面不同视图间呈现略有出入，以 TradingView 原页为准：
01 https://www.tradingview.com/script/cGj67e4X-blackcat-L3-Magic-9-Sequential-Setup/
02 https://www.tradingview.com/script/JQcD6UFh-blackcat-L3-Magic-9-13-with-Reversal-Points-Confirmation/
03 https://www.tradingview.com/script/xx21J50b-blackcat-L3-Projected-Magic-9-Sequence/
04 https://www.tradingview.com/script/31xvShhe-blackcat-L3-Magic-9-MACD-Setup/
05 https://www.tradingview.com/script/tDzRDIeU-blackcat-L3-Magic-9-Sequential-MACD/
06 https://www.tradingview.com/script/h92KjsEI-blackcat-L3-Magic-9-Sequential-within-Highlights/

作者另有一个自研高级版本「神泣三转」，未开源，不在本存档内。
