# NSW1 RRP 概率预测与表后储能运营研究

本项目研究如何利用 AEMO 五分钟预调度（P5MIN）及其他决策时可见信息，滚动预测未来 1 小时的 NSW1 区域参考价格（Regional Reference Price, RRP）概率分布，并评估这些预测能否改善新南威尔士州小型表后储能的风险调整增量净收益。

项目是可复现的离线研究环境，不是生产控制、市场报价或真实结算系统。

## 第一阶段

| 项目 | 当前约定 |
|---|---|
| 区域 | NSW1 |
| 决策与数据粒度 | 5 分钟 |
| 预测范围 | 未来 60 分钟 |
| 输出步数 | 12 个 horizon |
| 第一优先级 | 概率预测 |
| 最小概率输出 | P10 / P50 / P90 |
| 官方预测基线 | AEMO P5MIN |
| 历史 forecast 入口 | NEMSEER |
| 历史实际与市场特征入口 | NEMOSIS |
| 基准储能 | 10 kW / 20 kWh 可用能量 |
| 结算 | NSW1 RRP 加显式费用调整 |
| 费用 | 低、中、高参数化情景 |
| 主比较 | 概率指标对比 P5MIN 校准基线；点预测与运营价值对比原始 P5MIN |

P5MIN 每 5 分钟运行一次并给出下一小时、共 12 个五分钟周期的调度与价格结果，因此与第一阶段任务直接对应。

## 研究闭环

    AEMO 数据
      -> forecast-vintage 与可见时间对齐
      -> 未来 12 步 NSW1 概率预测
      -> 风险感知的储能滚动决策
      -> 参数化费用与退化核算
      -> 相对基线的样本外增量净收益

P5MIN 将同时作为官方基线、核心特征、残差校准基础和集成成员。项目不会只比较 MAE 或 RMSE；模型必须同时接受概率校准、负价/高价事件和储能运营价值检验。

## 文档导航

- [项目工作契约](./AGENTS.md)
- [研究范围](./docs/RESEARCH_SCOPE.md)
- [数据契约](./docs/DATA_CONTRACT.md)
- [概率预测与评价协议](./docs/FORECAST_PROTOCOL.md)
- [储能与经济回测协议](./docs/BESS_BACKTEST_PROTOCOL.md)
- [实验记录模板](./docs/EXPERIMENT_TEMPLATE.md)

## 数据工具

- [NEMSEER](https://github.com/UNSW-CEEM/NEMSEER)：获取和整理 AEMO 历史 P5MIN 等 forecast vintages。
- [NEMOSIS](https://github.com/UNSW-CEEM/NEMOSIS)：获取 AEMO 历史 DISPATCHPRICE 及其他市场数据。

截至 2026-08-18，当前兼容基线为 Python 3.11、[nemseer 1.0.7](https://pypi.org/project/nemseer/) 和 [nemosis 3.8.1](https://pypi.org/project/nemosis/)。两个包是数据访问适配层；AEMO 原始文件、数据模型和现行规则仍是事实来源。

## 本地环境

项目使用 [uv](https://docs.astral.sh/uv/) 管理 Python 解释器、虚拟环境和锁定依赖：

```bash
uv sync --locked
uv run python -c "import nemseer, nemosis"
```

`.python-version` 将项目解释器固定为 Python 3.11；`uv.lock` 固定完整传递依赖。更新 NEMSEER、NEMOSIS 或其数据栈依赖前，必须先执行数据与回测兼容性测试。

## 初始里程碑

1. 审计 NEMSEER 与 NEMOSIS 的共同可用日期、字段、版本和缺失情况。
2. 构建不可覆盖的 NSW1 P5MIN vintage 与实际 RRP 数据集。
3. 复现原始 P5MIN、持久性和季节性基线。
4. 首先实现 P5MIN 残差的概率校准模型。
5. 扩展不同特征链路、概率模型和集成模型。
6. 建立滚动储能与参数化结算回测。
7. 比较预测指标改善是否转化为样本外净价值。

## 当前状态

初始化研究文档与 uv 环境配置已建立，NEMSEER 1.0.7 和 NEMOSIS 3.8.1 已锁定。数据目录和研究代码尚未开始实现；实现时必须以这些契约为准，并通过数据审计后再冻结具体历史区间和费用数值。

## 重要边界

- 研究目标和结算区域均为 NSW1。
- 其他区域只能作为决策时可见的辅助特征。
- RRP 不会直接、无条件地等同于用户最终电价。
- 合成资产和费用情景用于比较算法，不代表真实用户收益。
- 完美预知只表示机会空间上界，不是可部署结果。
