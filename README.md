# Databento MBO Sample

AAPL、MSFT 在 Nasdaq 和 NYSE Arca 的股票现货历史数据，日期为 **2026-09-15、2026-09-16**。
数据来源为 Databento 标准化行情，价格未复权。每个场所具有独立订单簿，数据覆盖相应场所的活动。

## 下载

**[下载完整数据包：Databento_MBO_Sample_v1.zip](https://github.com/Kuangyang980/Databento_MBO_Sample/releases/latest/download/Databento_MBO_Sample_v1.zip)**（约 347 MB）。

数据文件位于 [Releases](https://github.com/Kuangyang980/Databento_MBO_Sample/releases) 的 ZIP 附件中，解压后按下述目录排列。
仓库首页同时提供数据说明、`files.csv` 文件清单和 `SHA256SUMS.txt` 校验清单。

## 文件与范围

| 目录 | 数据 | 格式 |
| --- | --- | --- |
| `data/mbo/` | 订单级消息，L3，8 份文件 | `.dbn.zst` |
| `data/ohlcv-1m/` | Databento 成品分钟线，8 组，每组两种格式 | `.dbn.zst`、`.csv` |
| `data/ohlcv-1h/` | 从 MBO 成交事件聚合的小时线，8 份文件 | `.csv` |

目录依次按数据类型、场所、UTC 日期排列，文件名包含股票代码。
`XNAS.ITCH` 表示 Nasdaq TotalView-ITCH，`ARCX.PILLAR` 表示 NYSE Arca Integrated。
`files.csv` 列出全部文件、记录数、字节数、数据来源及 SHA-256；`SHA256SUMS.txt` 可用于完整性校验。

| 场所数据集 | 股票 | UTC 日期 | MBO 消息数 | 分钟线数 |
| --- | --- | --- | ---: | ---: |
| ARCX.PILLAR | AAPL | 2026-09-15 | 2,003,776 | 844 |
| ARCX.PILLAR | MSFT | 2026-09-15 | 968,589 | 854 |
| ARCX.PILLAR | AAPL | 2026-09-16 | 1,896,286 | 828 |
| ARCX.PILLAR | MSFT | 2026-09-16 | 1,109,933 | 850 |
| XNAS.ITCH | AAPL | 2026-09-15 | 5,211,804 | 836 |
| XNAS.ITCH | MSFT | 2026-09-15 | 2,182,962 | 697 |
| XNAS.ITCH | AAPL | 2026-09-16 | 5,212,217 | 774 |
| XNAS.ITCH | MSFT | 2026-09-16 | 2,521,343 | 720 |

共 21,106,910 条 MBO 消息、6,403 根成品分钟线；同一组分钟线的 DBN 和 CSV 内容对应相同记录。

## 时间与价格

日期按完整 UTC 日划分，窗口为 `[当日 00:00, 次日 00:00)`。
例如 2026-09-15 文件覆盖纽约 9 月 14 日 20:00 至 9 月 15 日 20:00，
对应北京时间 9 月 15 日 08:00 至 9 月 16 日 08:00。
当日纽约 09:30 开盘对应 UTC 13:30、北京时间 21:30。

DBN 时间戳使用 UTC Unix 纳秒整数。DBN 价格使用定点整数，**美元价格 = 整数价格 / 1,000,000,000**；
未定义价格为 `9223372036854775807`，未定义时间为 `18446744073709551615`。
本仓库 CSV 的价格已转为美元；时间字符串带有时区，分钟线为 UTC，小时线为纽约时间。

`.dbn.zst` 是 Zstandard 压缩的 Databento Binary Encoding，需用兼容 DBN 的读取工具解码。

## MBO 字段

| 字段 | 解释 |
| --- | --- |
| `ts_recv` | Databento 接收到消息的 UTC 纳秒时间 |
| `ts_event` | 交易所事件时间，具体取时位置依场所定义 |
| `rtype` | 消息结构类型；MBO 为 160 |
| `publisher_id` | 数据发布源编号，关联数据集和场所 |
| `instrument_id` | 证券数字编号，需结合日期及映射解释 |
| `action` | 事件类型，见下表 |
| `side` | A 卖方、B 买方、N 未指定；含义随 action 变化 |
| `price` | 定点价格，单位为 1e-9 美元 |
| `size` | 股数；在新增、撤减和成交消息中有不同含义 |
| `channel_id` | Databento 分配的数据频道编号 |
| `order_id` | 场所提供的委托编号 |
| `flags` | 消息标记位，可同时包含多个标记 |
| `ts_in_delta` | 接收时间减去源端发送时间，单位纳秒；源端发送时间可由 ts_recv 减此值取得 |
| `sequence` | 源端消息序号，需要结合场所和频道解释 |
| `symbol` | 股票代码；解码时根据 DBN 元数据映射附加，属于展示列 |

| action | 含义 |
| --- | --- |
| `A` | 新增委托 |
| `M` | 修改委托价格或数量 |
| `C` | 全部撤单或部分减量 |
| `R` | 清空该证券的存量委托 |
| `T` | 成交事件，描述主动成交一侧 |
| `F` | 被动挂单的成交通知 |
| `N` | 无盘口更新动作，可携带其他标记 |

`side` 对 T 表示主动方，对 F 表示被成交的挂单方，对 A/M/C 表示挂单所在买卖侧。
Nasdaq 的 T 消息 `order_id` 为 0，成交方向也可能为 N。
一次 Nasdaq 执行可能被标准化为 T、F、C 多条消息；统计成交时使用 T，盘口扣量按更新消息语义处理。

常见 flags：128 表示该事件最后一条消息；32 表示回放/快照；8 表示接收时间可能不准确；
4 表示频道出现无法恢复的缺口。标记可叠加，应按位检查。
委托编号、消息序号与事件时间均不应单独视为跨文件的唯一键。

## K 线字段与聚合口径

| 字段 | 解释 |
| --- | --- |
| `ts_event` | 成品分钟线的区间起点，依据成交 ts_recv 分桶 |
| `bar_start` | 本地小时线的区间起点 |
| `open` / `high` / `low` / `close` | 首笔、最高、最低、末笔成交价 |
| `volume` | 区间内成交股数 |
| `trade_count` | 小时线参与聚合的 T 消息条数 |
| `vwap` | 小时线按成交股数加权的均价 |
| `symbol` / `dataset` | 股票代码 / 场所数据集；dataset 列在小时线中提供 |
| `rtype` / `publisher_id` / `instrument_id` | 成品分钟线的结构和来源标识；rtype 为 33 |

分钟和小时区间均左闭右开，以整分钟或整小时起算，包含文件内盘前、常规及盘后成交；
无成交的桶不补价格。本地聚合使用有效 T 消息，以 ts_recv 分桶，同一时间戳内保留原始顺序。
小时线的 09:00 表示纽约 `[09:00,10:00)`，而非从 09:30 开盘起算。

8 组 MBO 本地聚合分钟线已与另行下载的 Databento 成品分钟线逐根核对：
**6,403 根 OHLCV 全部一致，无缺失分钟。**

## 数据来源

- [Databento MBO 定义](https://databento.com/docs/schemas-and-data-formats/mbo)
- [Databento OHLCV 定义](https://databento.com/docs/schemas-and-data-formats/ohlcv)
- [通用字段、动作和标记](https://databento.com/docs/standards-and-conventions/common-fields-enums-types)
- [Nasdaq 标准化说明](https://databento.com/docs/venues-and-datasets/xnas-itch)
- [NYSE Arca 说明](https://databento.com/docs/venues-and-datasets/arcx-pillar)

数据的使用及再分发遵循 Databento 与相应数据发布方的适用条款。
