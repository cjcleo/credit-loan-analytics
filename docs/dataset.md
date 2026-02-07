# 数据集说明

## 一、清洗后表 `loan_clean.csv` 的列来源

**数据流**：原始表 `loan_sample.csv` (80万行只截取20万行) → `loan_sample_100m.csv`，约 20 万行、145 列）→ `1_data_prepare.ipynb` → **`data/loan_clean.csv`**。

### 1.1 保留的原始列

仅保留以下列表中**在原始表中存在**的列，其余删除。

| 类别 | 字段 |
|------|------|
| **基础/结果** | `loan_status`, `issue_d`, `loan_amnt`, `funded_amnt`, `funded_amnt_inv` |
| **期限与定价** | `term`, `int_rate`, `installment` |
| **信用与用途** | `grade`, `sub_grade`, `verification_status`, `purpose`, `addr_state`, `zip_code`, `title` |
| **收入与房产** | `dti`, `annual_inc`, `emp_length`, `emp_title`, `home_ownership` |
| **历史逾期与不良** | `delinq_2yrs`, `mths_since_last_delinq`, `pub_rec`, `revol_bal`, `revol_util` |
| **账户与查询** | `open_acc`, `total_acc`, `inq_last_6mths`, `inq_last_12m`, `earliest_cr_line` |
| **催收与核销** | `collections_12_mths_ex_med`, `chargeoff_within_12_mths` |
| **还款与余额** | `total_pymnt`, `total_rec_int`, `total_rec_prncp`, `last_pymnt_d`, `last_pymnt_amnt`, `out_prncp`, `out_prncp_inv` |
| **授信与使用** | `avg_cur_bal`, `bc_util`, `all_util`, `tot_cur_bal`, `total_rev_hi_lim` |
| **其他风险** | `acc_now_delinq`, `num_accts_ever_120_pd`, `num_tl_90g_dpd_24m`, `pub_rec_bankruptcies` |
| **申请类型** | `application_type` |

### 1.2 清洗阶段生成的衍生列

| 衍生列 | 来源 | 说明 |
|--------|------|------|
| `is_default` | `1_data_prepare.ipynb` | `loan_status` ∈ {Charged Off, Default, Late (31-120/16-30 days), In Grace Period} → 1，否则 0 |
| `term_months` | `1_data_prepare.ipynb` | 从 `term` 字符串提取数字（如 "36 months" → 36） |
| `issue_date` | `1_data_prepare.ipynb` | 由 `issue_d` 解析为日期（如 "Dec-2015"） |
| `int_rate` | `1_data_prepare.ipynb` | 若为带 "%" 字符串，则转为数值 |

缺失处理：如 `mths_since_last_delinq` 缺失填 180 等，见 notebook。最终导出前会做缺失/异常处理，保证下游可用的行与列。

---

## 二、字段说明

| 字段 | 说明 | 类型 |
|-----|------|------|
| `loan_status` | 贷款状态；定义好坏客户，衍生 `is_default`（Charged Off、Default、Late、In Grace Period 等为 1） | 基础与结果 |
| `issue_d` | 发标/放款日期（如 "Dec-2015"）；时间范围、趋势，清洗中可衍生 `issue_date` | 基础与结果 |
| `funded_amnt` | 实际放款金额；转化与放款结构、笔均、违约率分母 | 基础与结果 |
| `loan_amnt` | 申请金额；额度分析、净收益计算（与 total_pymnt）、预测与聚类特征 | 基础与结果 |
| `delinq_2yrs` | 过去 2 年逾期次数 | 历史违约与不良 |
| `mths_since_last_delinq` | 距离上次逾期月数 | 历史违约与不良 |
| `pub_rec` | 公共记录数（含破产等）；无 `pub_rec_bankruptcies` 时代替 | 历史违约与不良 |
| `pub_rec_bankruptcies` | 破产记录数 | 历史违约与不良 |
| `collections_12_mths_ex_med` | 催收次数（12 个月） | 历史违约与不良 |
| `chargeoff_within_12_mths` | 12 个月内核销 | 历史违约与不良 |
| `grade` | 信用等级（A–G） | 信用评分 |
| `sub_grade` | 子等级（如 A1–A5） | 信用评分 |
| `dti` | 负债收入比 | 杠杆与偿债能力 |
| `revol_util` | 信用卡使用率（可能带 %）；代码中先转数值 | 杠杆与偿债能力 |
| `revol_bal` | 循环余额 | 杠杆与偿债能力 |
| `bc_util`, `all_util`, `avg_cur_bal`, `tot_cur_bal`, `total_rev_hi_lim` | 银行卡/总授信使用率、平均余额等，供扩展分析 | 杠杆与偿债能力 |
| `int_rate` | 利率（可能带 %）；清洗转数值，可衍生 `int_rate_num` | 收益与定价 |
| `term` | 期限（如 "36 months"）；清洗衍生 `term_months` | 收益与定价 |
| `installment` | 每期还款额；可衍生 `installment_inc_ratio`（与 annual_inc） | 收益与定价 |
| `total_rec_int`, `total_pymnt`, `total_rec_prncp` | 已收利息、已还总额、已还本金；净收益用 total_pymnt − loan_amnt | 收益与定价 |
| `last_pymnt_d`, `last_pymnt_amnt`, `out_prncp`, `out_prncp_inv` | 最后还款日/额、未还本金等 | 收益与定价 |
| `annual_inc` | 年收入 | 用户画像与分群 |
| `emp_length` | 工作年限（如 "10+ years"）；可衍生 `emp_length_num` | 用户画像与分群 |
| `home_ownership` | 房产状态；可衍生 `home_ownership_num`（MORTGAGE=2, OWN=1, 其他=0） | 用户画像与分群 |
| `addr_state`, `purpose` | 所在州、借款用途 | 用户画像与分群 |
| `inq_last_6mths`, `inq_last_12m` | 近 6/12 个月征信查询次数 | 用户画像与分群 |
| `open_acc`, `total_acc` | 当前在贷账户数、历史账户总数 | 用户画像与分群 |
| `earliest_cr_line` | 最早信用账户开通时间 | 用户画像与分群 |
| `emp_title`, `application_type` | 职位、申请类型 | 用户画像与分群 |
| `zip_code`, `title` | 邮编、标题 | 用户画像与分群 |
| `acc_now_delinq` | 当前逾期账户数 | 其他风险与行为 |
| `num_accts_ever_120_pd` | 曾逾期 120 天以上的账户数 | 其他风险与行为 |
| `num_tl_90g_dpd_24m` | 24 个月内 90+ 天逾期账户数 | 其他风险与行为 |

