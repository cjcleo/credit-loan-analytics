# 信贷业务数据分析

基于信贷用户数据，从**用户转化**与**风险**两个维度进行分析，支持精细化获客与风控决策。

---

## Problem

- **场景**：信贷平台需在控制风险的前提下提升用户转化与放款规模，平衡获客与违约。
- **核心问题**：
  - 放款结构是否健康？哪类客户笔数/金额/违约率异常？
  - 哪些因素最驱动违约（利率、DTI、历史逾期、破产记录等）？
  - 能否用模型预测违约并指导审批/定价？哪些特征最重要？
  - 如何对客户分群，实现差异化运营与风控？

---

## Data

### 数据来源

- **数据集**：[Lending Club Loan Data (Kaggle)](https://www.kaggle.com/datasets/adarshsng/lending-club-loan-data-csv)
- **背景**：Lending Club 是美国 P2P 借贷平台，撮合投资者与借款人：投资者出资，借款人还款（本金+利息）回流投资者；借款人通常获得较低利率，投资者获得收益。
- **内容**：2007–2015 年通过 Lending Club 发放的**全量贷款**记录，包含当前贷款状态（Current、Late、Fully Paid 等）、最近还款信息；特征包括信用评分、征信查询次数、地址（邮编、州）、collections（是否曾逾期并由团队催收）等。原始数据约 **89 万条观测（代码中仅使用前20万条）、75 个变量**。
- **本项目**：使用上述数据集的子集（如 `loan_sample_100m.csv`，约 20 万行），经清洗后得到 `loan_clean.csv` 用于分析。

### 清洗与产出

| 环节 | 说明 |
|------|------|
| **清洗** | `1_data_prepare.ipynb`：保留已放款样本，列筛选（`COLS_TO_KEEP`），`loan_status` → `is_default`，`int_rate`/`term` 转数值，缺失与异常处理 |
| **产出** | `data/loan_clean.csv`（约 16.8 万笔、50+ 列），供描述分析、建模与聚类 |

**核心指标口径**：违约 = `loan_status` ∈ {Charged Off, Default, Late, In Grace Period} → `is_default=1`；违约率 = 违约笔数 / 已放款笔数。

---

## Feature

| 用途 | 特征 |
|------|------|
| **放款/违约描述**（2_data_summary） | 维度：`grade`, `purpose`, `verification_status`, `addr_state`；分箱：`int_rate`→int_rate_bin, `dti`→dti_bin, `revol_util`→revol_util_bin；离散：`delinq_2yrs`, `pub_rec_bankruptcies`/`pub_rec` |
| **违约预测**（3_default_prediction） | 数值：`int_rate`, `dti`, `loan_amnt`, `funded_amnt`, `installment`, `delinq_2yrs`, `mths_since_last_delinq`, `revol_util`, `open_acc`, `total_acc`, `inq_last_6mths`, `annual_inc`, `term_months`, `revol_bal`；分类 One-Hot：`grade`, `purpose`, `home_ownership`（约 34 维） |
| **客户分群**（4_cluster_profile） | 偿债：`annual_inc`, `dti`, `installment_inc_ratio`；信用历史：`delinq_2yrs`, `num_tl_90g_dpd_24m`, `pub_rec_bankruptcies`, `inq_last_6mths`；使用与规模：`revol_util`, `loan_amnt`, `int_rate_num`, `term_months`；稳定性：`emp_length_num`, `home_ownership_num`（13 维，标准化后 K-Means） |

---

## Model

| 模块 | 模型 | 说明 |
|------|------|------|
| 放款与违约描述 | — | 按维度/分箱聚合：笔数、放款额、违约率（`2_data_summary.ipynb`） |
| 违约预测 | **Logistic Regression** | 可解释性好，标准化特征，`class_weight='balanced'` |
| 违约预测 | **Random Forest** | 原始特征，`n_estimators=100`, `max_depth=10`，用于特征重要性 |
| 违约预测 | **XGBoost / LightGBM** | 梯度提升树，`scale_pos_weight` 处理不平衡，可选对比 KS/AUC |
| 客户分群 | **K-Means**（k=4） | 标准化后的 13 维特征，产出 4 类客群标签 |

---

## Result

- **描述分析**：按 grade 违约率单调上升（A 约 0.4% → G 约 7%）；按 int_rate / DTI / revol_util / delinq_2yrs / 破产记录分箱均呈现“越高越易违约”。
- **违约预测**（样本违约率约 1.18%，8:2 划分；不平衡用 class_weight / scale_pos_weight）：
  - 评估：Accuracy / Precision / Recall / F1 / AUC-ROC /  KS/ Lift/PR 曲线与 AP
  - **四模型对比**：**LR** 召回最高（约 68%）、AUC 0.72、KS 0.325，达 KS>0.3，更适合“少漏违约”；**RF** 准确率约 82%、AUC 0.69、KS 0.311；**LGB** 召回约 46%、AUC 0.66、KS 0.24；**XGB** 准确率最高（约 89%）但召回最低（约 19%）、AUC/KS 相对较弱。更怕漏掉违约可优先选 LR，更怕误拒好人可提高阈值或选更保守模型。
  - 特征重要性（RF 等）：Top 为 `int_rate`, `revol_bal`, `dti`, `installment`, `revol_util`, `annual_inc`, `grade_D`/`grade_E` 等，notebook 中附**业务解读**。
- **分群画像**：四类——低风险优质、高负债高风险、高收入高额度、潜在违约；各簇规模、违约率、平均净收益可对比，支撑差异化策略。

---

## Business insight

- **放款与风险结构**：Grade 与违约率强相关，可按 purpose / 地区 / 核验状态识别高违约或低转化细分；高 DTI、高 revol_util、历史逾期/破产越多，违约率越高。
- **模型与风控**：在极度不平衡下应看召回率、AUC 与 KS，不能只看准确率；四模型对比下 LR 在召回与 KS 上均优，更利于“多拦潜在违约”；RF/XGB/LGB 特征重要性可指导准入与定价（利率、DTI、revol_util、grade 等）。
- **策略建议**：对高转化且违约可控的 grade/用途/地区加大投放；依据特征重要性优化准入与定价，对高利率高违约群体收缩额度或加强贷后；按聚类分群做差异化——优质客群提额与复借激励，高风险客群收紧或拒绝，潜在违约客群加强触达与还款提醒。

---

## 仓库结构

```
├── README.md
├── src/
│   ├── 1_data_prepare.ipynb   # 数据清洗 → loan_clean.csv
│   ├── 2_data_summary.ipynb   # 放款总结 + 违约率多维分析
│   ├── 3_default_prediction.ipynb  # 违约预测 LR + RF + XGB/LGB，KS/Lift/PR，特征解读
│   └── 4_cluster_profile.ipynb    # K-Means 客户分群
├── data/
└── docs/
    └── dataset.md             # 字段说明
```

运行顺序：1 → 2 → 3 → 4（2/3/4 依赖 `data/loan_clean.csv`）。
