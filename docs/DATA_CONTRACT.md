# 数据契约

## 目的

本契约定义第一阶段 NSW1 五分钟 RRP 概率预测所需的数据源、时间语义、标准字段、可见性、血缘和质量门槛。其首要目标是保证历史回测能够重建“某个决策时点真正可以知道什么”，而不是只拼接事后最完整的数据。

## 数据源与权威边界

### AEMO

AEMO 原始文件、MMS 数据模型和现行规则是市场事实来源。第三方包解析后的结果必须能追溯到相应 AEMO 表、月份、文件或查询。

### NEMSEER

第一阶段使用 NEMSEER 获取和整理历史 P5MIN forecast vintages：

- forecast type：P5MIN；
- 核心表：REGIONSOLUTION；
- 核心区域：NSW1；
- 核心内容：每个 run 对未来区间的区域价格与相关区域结果；
- 关键时间：RUN_DATETIME、INTERVAL_DATETIME、LASTCHANGED 或对应版本字段。

NEMSEER 能按 run time 和 forecasted time 查询历史预测，但它是适配器而不是权威数据定义。每次使用前必须通过实际小样本检查所选月份的表版本、字段和重复键。

### NEMOSIS

第一阶段使用 NEMOSIS 获取实际市场数据：

- 核心表：DISPATCHPRICE；
- 主标签：NSW1、非干预情形的 RRP；
- 候选特征：实际价格、区域需求、机组调度、可再生能源、联络线和约束等决策时可见或可重建字段。

任何候选表加入模型前都必须单独建立发布时间和可见性契约。能被 NEMOSIS 下载不等于它在历史决策时点已经可用。

## 依赖基线

截至 2026-08-18，兼容基线为：

| 组件 | 版本 |
|---|---|
| Python | 3.11 |
| nemseer | 1.0.7 |
| nemosis | 3.8.1 |

版本必须进入依赖锁文件和每次数据 manifest。升级需先运行小样本契约测试，不得让包升级静默改变字段、时间解析、缓存或缺失行为。

版本来源：[nemseer PyPI](https://pypi.org/project/nemseer/)、[nemosis PyPI](https://pypi.org/project/nemosis/)。

## 时间标准

### 市场时间

所有 AEMO 市场时间使用固定 AEST，即 UTC+10，不切换夏令时。

### 站点时间

合成或未来真实站点可使用 Australia/Sydney 表示当地日历、负荷、光伏和零售费率。进入市场层时必须显式转换为 AEST，并保留原始本地时间和 UTC offset。

### 必须区分的时间

| 标准字段 | 含义 |
|---|---|
| source_run_time_aest | AEMO ahead process 的名义 run time |
| source_lastchanged_aest | AEMO 表中的 LASTCHANGED 或同类字段 |
| available_at_aest | 本研究用于 as-of 判断的可见时间 |
| model_issue_time_aest | 自研模型生成预测的决策时点 |
| target_interval_end_aest | 目标五分钟区间结束时点 |
| ingested_at_utc | 本项目下载或处理数据的真实 UTC 时间 |
| price_revision_at_aest | 若可获得，价格修订或确认时间 |

source_run_time_aest 与 available_at_aest 不得默认相等。available_at_aest 的构造规则必须随数据集版本保存。

LASTCHANGED 只能在完成数据审计并说明局限后作为可见时间代理。它可能反映处理或记录变更，并不自动等同于外部平台接收时间或 AEMO 发布 SLA。

### 区间语义

- 市场目标采用 interval-ending。
- 时间范围采用半开区间 start <= t < end，除非数据源 API 明确采用其他规则。
- 五分钟时长统一为 5/60 小时。
- model_issue_time_aest 到 target_interval_end_aest 的差定义为 horizon_minutes。
- 第一阶段合法 horizon_minutes 为 5、10、...、60。
- horizon_step = horizon_minutes / 5。

任何无法被 5 分钟整除的对齐结果都必须失败，不得自动舍入。

## 价格标签与版本

第一阶段标准标签字段：

| 字段 | 含义 |
|---|---|
| region_id | 固定 NSW1 |
| target_interval_end_aest | 实际价格对应的五分钟区间结束 |
| target_rrp_aud_per_mwh | NSW1 RRP，单位 A$/MWh |
| intervention | 默认 0 |
| price_version | 价格来源与版本口径 |
| target_source_file | 原始或归档来源身份 |

通过 NEMOSIS 月度历史归档获得的价格默认命名为 aemo_archive_dispatch_rrp。除非已经保存并验证相应 interval 发布文件和发布时间，不得将月度归档标签称为 as-published。

如果后续同时维护首次发布、调整、确认或最终结算价格，应作为不同 price_version 保存和评价，不得覆盖同一字段。模型训练标签与经济回测结算标签必须在实验中分别声明。

极端价格、负价和经调整价格必须保留原始值。主标签不得裁剪、置零或 winsorize。

## P5MIN forecast-vintage 标准表

标准长表每行表示某个来源 run 对某个未来目标的预测：

| 字段 | 要求 |
|---|---|
| region_id | NSW1 |
| source_run_time_aest | 必填 |
| source_lastchanged_aest | 原表存在时必填 |
| available_at_aest | 必填，规则有版本 |
| target_interval_end_aest | 必填 |
| horizon_minutes | 正整数且为 5 的倍数 |
| horizon_step | 正整数 |
| p5min_rrp_aud_per_mwh | P5MIN 预测 RRP |
| intervention | 默认过滤为 0 |
| source_table | P5MIN/REGIONSOLUTION |
| source_schema_version | 尽可能记录 |
| source_file | 必填或通过 manifest 可解析 |
| source_row_id | 能定位原始记录的稳定标识 |

候选唯一键：

    region_id
    source_run_time_aest
    target_interval_end_aest
    intervention
    source_schema_version

若实际表存在额外键，适配器必须保留。任何重复键都应先审计，不得任意保留第一行或最后一行。

## 模型样本标准表

每个 model_issue_time 和 target 的建模样本至少包含：

| 字段 | 说明 |
|---|---|
| model_issue_time_aest | 模型决策时点 |
| target_interval_end_aest | 目标区间结束 |
| horizon_minutes | 5 至 60 |
| horizon_step | 1 至 12 |
| region_id | NSW1 |
| target_rrp_aud_per_mwh | 训练或评价标签 |
| target_price_version | 标签版本 |
| latest_p5min_available_at_aest | 选中 P5MIN 的可见时间 |
| latest_p5min_run_time_aest | 选中 P5MIN 的来源 run |
| p5min_rrp_aud_per_mwh | 官方基础预测 |
| feature_snapshot_id | 特征快照标识 |
| data_snapshot_id | 数据版本标识 |

样本构建必须保留未选中的历史 vintages；latest 只表示在该 model_issue_time 下按明确规则选中的最新可用预测。

## As-of 连接规则

任一特征 x 进入 model_issue_time = t 的样本必须满足：

    available_at_aest(x) <= t

如果可见时间未知：

1. 优先查找 AEMO 数据模型、文件元数据或官方发布说明；
2. 必要时采用明确、保守的延迟假设；
3. 将延迟假设写入配置并做敏感性；
4. 无法合理界定时从可部署特征集排除。

禁止：

- 使用目标区间之后才发布的数据；
- 将月度最终归档值当成历史时点已知特征；
- 按 target time 连接同一时刻的实际价格；
- 使用全数据统计量进行填补、缩放或异常检测；
- 用测试期信息选择高价阈值、特征或数据清洗规则。

## 数据分层

建议的逻辑分层如下，实际目录在代码初始化时建立：

### raw

- NEMSEER、NEMOSIS 下载缓存；
- 可获得的 AEMO ZIP、CSV 或原始文件身份；
- 只追加，不覆盖；
- 文件哈希和下载日志。

### standardized

- 字段名、类型、区域、单位和 AEST 时间统一后的 Parquet；
- 保留来源键和原始字段；
- 按数据集、年份和月份分区。

### curated

- forecast-vintage 长表；
- 实际 RRP 标签；
- 按可见时间构建的候选特征；
- 质量报告和覆盖矩阵。

### features

- 具体 feature set 的不可变快照；
- 只使用训练期拟合的转换；
- 保存特征定义版本。

### artifacts

- 预测、模型、回测、指标、图和 manifest；
- 已有实验禁止覆盖。

## 原始数据与 manifest

每次数据构建至少记录：

- dataset_id 和 schema_version；
- 查询起止时间及边界语义；
- region、table、forecast_type 和过滤条件；
- Python、NEMSEER、NEMOSIS 和关键依赖版本；
- 原始来源 URL 或文件名；
- 下载时间；
- 文件 SHA-256；
- 原始、过滤和输出行数；
- 最小与最大 run、available 和 target time；
- 重复、缺失、异常和排除计数；
- available_at 构造规则版本；
- 生成代码的 Git commit；
- 产物文件和哈希。

若第三方包未保留精确原始 ZIP，manifest 必须至少保留其下载路由、AEMO 文件身份和包缓存文件哈希；在进入正式结论前评估是否需要独立保留 AEMO 原始文件。

## 缺失值

### 不允许填补

- 实际 RRP 标签；
- 当前样本所需的 P5MIN 基础预测；
- issue、available 或 target time；
- region、intervention、price version；
- 储能 SOC、功率或费用配置。

缺失上述字段时，该样本不得进入主评价，并必须报告缺失原因和比例。

### 可研究性填补

候选辅助特征可在以下条件下填补：

- 只用当时可见的历史数据；
- 方法在训练期拟合；
- 同时增加缺失指示；
- 与不填补或删除特征做消融；
- 在实验记录中报告。

## 数据质量门

进入建模前必须通过：

- region_id 全部为 NSW1；
- intervention 过滤符合实验契约；
- target 和 run 时间为合法五分钟边界；
- 第一阶段 horizon 完整或显式报告缺口；
- forecast-vintage 键无未解释重复；
- 每月 P5MIN run、target 和实际标签覆盖率报告；
- P5MIN 与实际 RRP 单位一致；
- 价格缺失和非有限值为零或有明确排除说明；
- 极端值保留且可追溯；
- available_at 不晚于其进入的 model_issue_time；
- 测试期未参与数据转换拟合；
- manifest 覆盖全部产物。

## 历史切分候选

数据审计后按共同可用月份确定最终边界：

- 起点不早于 2021-10-01；
- 使用所有质量合格的完整历史月份；
- 最新完整 12 个月冻结为最终滚动测试候选；
- 测试之前设置时间连续验证期；
- 更早数据作为训练期；
- 模型或数据链路不得根据最终测试结果反复选择。

如有效历史不足，应缩减模型复杂度、扩大不确定性并明确报告，不得通过随机切分制造更多测试样本。

## 数据契约变更

字段、价格版本、可见时间规则、数据包版本或切分边界变化都属于契约变更。变更必须：

1. 提升 schema 或 dataset version；
2. 重新生成质量报告；
3. 使旧实验继续可复算；
4. 说明哪些结果因此失效；
5. 不覆盖旧数据和 manifest。
