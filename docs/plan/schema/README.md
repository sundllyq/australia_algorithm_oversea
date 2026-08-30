# 第一阶段字段字典与版本命名

本文档承接主技术方案中的长字段列表，避免正文被 schema 细节打断。字段实现可以按存储技术调整类型和分区方式，但不得改变主键语义、时间可见性、单位和血缘要求。

## 1. 字段分组原则

| 分组 | 作用 | 典型字段 |
|---|---|---|
| 业务主键 | 唯一定位一条事实或产物 | `decision_id`、`snapshot_id`、`forecast_distribution_id`、`scenario_batch_id` |
| 时间与信息集 | 防止未来信息泄漏 | `decision_cutoff_aest`、`model_issue_time_aest`、`target_interval_end_aest`、`source_available_at_aest` |
| 数值事实 | 价格、概率、功率、电量和现金流 | `rrp_aud_per_mwh`、`quantile_level`、`charge_kwh`、`net_value_aud` |
| 配置与方法版本 | 连接模型、校准、路径、优化和费用配置 | `forecast_cdf_family_version`、`scenario_engine_version`、`fee_scenario_id` |
| 血缘与质量 | 可复算、可审计和降级归因 | `source_manifest_hash`、`raw_object_id`、`quality_flags`、`fallback_reason` |

正文只列核心字段组；完整持久化结构按下列字段字典实现。

## 2. 版本命名与兼容别名

| 概念 | 主字段 | 含义 | 兼容/相关字段 |
|---|---|---|---|
| CDF 方法族 | `forecast_cdf_family_version` | 目标、锚点、特征 schema、模型策略、分位数网格、单调化、校准、事件融合、插值和尾部规则 | 旧 `forecast_cdf_version` 仅在完成映射后作为别名 |
| 完整预测快照 | `forecast_snapshot_id` | 某次生效的完整预测流水线，引用全部组件实例、拟合窗口和有效期 | 不等于输入数据快照 |
| 模型组件实例 | `model_snapshot_id` | 本次拟合出的模型组件包 | `model_id` 仅保留为兼容别名或展示字段 |
| 校准组件实例 | `calibration_snapshot_id` | 本次拟合出的校准器实例 | `calibration_id` 仅保留为兼容别名或展示字段 |
| 单轮输入快照 | `data_snapshot_id` / `feature_snapshot_id` | 当前决策截点可见的数据和特征快照 | 不表示模型权重或校准器 |
| CDF 构造子版本 | `distribution_method_version` | family 内部的 CDF 构造、插值、尾部等子版本 | 不单独定义兼容策略，引用 family 清单 |
| PIT 规则 | `pit_method_version` | PIT 左极限、点质量随机化和哈希映射规则 | 与 CDF family 分开管理 |
| 场景引擎 | `scenario_engine_version` | J0/J1/J2/J3、抽样、重排、回退、压缩和权重规则 | 与当前 CDF family、模板 family 都需记录 |

兼容原则：主实现优先使用主字段；别名字段只用于旧记录映射或对外展示，不得在同一 artifact 中承载不同语义。

## 3. 数据与 snapshot 层

### 3.1 Raw manifest

```text
raw_object_id
source_name
dataset_name
source_uri_or_query
query_parameters
nominal_source_run_at
source_last_changed_at
local_first_seen_at
ingested_at_utc
sha256
byte_size
parser_version
nemseer_version
nemosis_version
storage_uri
parse_status
quality_flags
```

### 3.2 Snapshot metadata

```text
snapshot_id
decision_cutoff_aest
model_issue_time_aest
first_target_interval_end_aest
horizon_steps
availability_policy_id
feature_version
source_manifest_hash
completeness_ratio
max_staleness_seconds
quality_flags
created_at_utc
```

### 3.3 P5MIN vintage table

```text
region_id
run_datetime_aest
source_available_at_aest
target_interval_end_aest
horizon_step
forecast_rrp_aud_per_mwh
forecast_demand_mw
source_last_changed_at
raw_object_id
availability_quality
```

### 3.4 Actual price table

```text
region_id
target_interval_end_aest
intervention
rrp_aud_per_mwh
rop_aud_per_mwh
apc_flag
market_suspended_flag
price_version
source_last_changed_at
raw_object_id
```

### 3.5 Model sample table

```text
sample_id
snapshot_id
model_issue_time_aest
decision_cutoff_aest
target_interval_end_aest
horizon_step
region_id
anchor_rrp_aud_per_mwh
anchor_run_datetime_aest
anchor_age_seconds
actual_rrp_aud_per_mwh
residual_aud_per_mwh
target_price_version
feature_version
availability_quality
```

## 4. 预测与模板层

### 4.1 Forecast distribution long table

```text
forecast_distribution_id
model_issue_time_aest
decision_cutoff_aest
target_interval_end_aest
horizon_minutes
horizon_step
region_id
anchor_rrp_aud_per_mwh
anchor_run_datetime_aest
quantile_level
raw_quantile_aud_per_mwh
calibrated_quantile_aud_per_mwh
prob_negative
high_price_threshold_aud_per_mwh
prob_high
forecast_cdf_family_version
forecast_snapshot_id
model_snapshot_id
calibration_snapshot_id
data_snapshot_id
distribution_method_version
quality_flags
generated_at_utc
```

兼容展示字段：`model_id` 可映射到 `model_snapshot_id`，`calibration_id` 可映射到 `calibration_snapshot_id`；新记录不应依赖别名表达主语义。

### 4.2 PIT template table

```text
template_id
template_issue_time_aest
template_decision_cutoff_aest
first_target_interval_end_aest
pit_path[H]
residual_path[H]
outcome_available_at_aest
template_eligible_at_aest
market_day_id
episode_id
forecast_cdf_family_version
forecast_snapshot_id
model_snapshot_id
calibration_snapshot_id
pit_method_version
context_version
template_eligibility_policy_id
episode_policy_id
episode_metadata_available_at_aest
source_manifest_hash
target_price_version
created_at_utc
source_quality
```

多事件归属使用 `(template_id, episode_id)` 关联长表，不强制把多个事件塞进单值字段。`[H]` 表示配置 horizon 长度，第一阶段主任务为 12。

## 5. 场景与优化层

### 5.1 Scenario long table

```text
scenario_batch_id
forecast_distribution_id
scenario_id
scenario_weight
model_issue_time_aest
target_interval_end_aest
horizon_step
region_id
rrp_aud_per_mwh
scenario_method
requested_method
actual_method
fallback_reason
source_template_id
template_library_id
template_library_manifest_hash
template_compatibility_policy_id
template_governance_policy_id
template_eligible_at_aest
selection_tier
selection_distance
distribution_version
calibration_snapshot_id
scenario_engine_version
rng_seed
bundle_type
quality_flags
```

`distribution_version` 应映射到当前 `forecast_cdf_family_version`。J0/J1 无 PIT 模板时模板字段为空；J1 另保留残差块来源键。

### 5.2 Optimization request

```text
decision_id
decision_cutoff_aest
scenario_batch_id
state_observed_at_aest
initial_energy_kwh
asset_config_id
nmi_config_id
site_forecast_id
fee_scenario_id
degradation_config_id
risk_config_id
information_structure_id
terminal_value_id
solver_config_id
```

### 5.3 Optimization response

```text
decision_id
status
first_action_interval_end_aest
charge_kw
discharge_kw
expected_incremental_value_aud
p05_incremental_value_aud
cvar_loss_aud
expected_terminal_value_aud
binding_constraints
solver_runtime_ms
solver_gap
fallback_action
quality_flags
```

## 6. 终端价值、回测和实验 manifest

### 6.1 Terminal value artifact

```text
terminal_value_id
method
asset_config_id
site_scenario_id
fee_scenario_id
training_start/end
validation_start/end
context_schema
energy_grid_kwh
reference_energy_kwh
continuation_horizon_minutes
boundary_condition
discount_parameters
predispatch_availability_policy_id
value_table_uri
source_manifest_hash
code_version
created_at_utc
```

V2 每轮动态曲线另存 `terminal_value_instance_id`，引用当时 Predispatch run 和数据 snapshot。

### 6.2 Backtest detail table

```text
backtest_run_id
strategy_id
decision_id
model_issue_time_aest
executed_interval_end_aest
actual_price_version
actual_rrp_aud_per_mwh
forecast_distribution_id
scenario_batch_id
initial_energy_kwh
final_energy_kwh
charge_kw / discharge_kw
charge_kwh / discharge_kwh
site_load_kwh / pv_kwh
site_forecast_id / site_outcome_trajectory_id
nmi_import_kwh / nmi_export_kwh
avoided_import_value_aud
export_revenue_aud
charge_cost_aud
fee_components_aud
degradation_cost_aud
decision_expected_terminal_value_aud
terminal_adjustment_aud
net_value_aud
solver_status / fallback_status
quality_flags
```

### 6.3 Experiment run manifest

```text
experiment_id
run_id
research_question
git_commit_or_source_hash
python_version
uv_lock_sha256
nemseer_version
nemosis_version
data_manifest_hash
snapshot_policy_id
forecast_snapshot_registry_hash
forecast_cdf_family_version
historical_replay_policy_id
feature_version
model_snapshot_id
calibration_snapshot_id
template_library_id
template_library_manifest_hash
template_eligibility_policy_id
template_compatibility_policy_id
template_governance_policy_id
episode_policy_id
pit_audit_report_uri
test_template_update_policy
scenario_engine_version
information_structure_id
optimizer_version
terminal_value_id
asset_config_id
site_scenario_id
fee_scenario_id
random_seed
train/validation/calibration/test boundaries
artifact_uris
started_at_utc
completed_at_utc
status
known_deviations
```

兼容清单可额外保存 `model_id`、`calibration_id`，但必须能一一映射到 snapshot 主字段。
