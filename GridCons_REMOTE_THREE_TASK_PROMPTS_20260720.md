# GridCons-Mamba 远程机三段任务 Prompt

使用方式：按顺序分别发送给远程机上的 Codex。远程用户和原目录保持不变：`/home/yanjingjia`。三段任务禁止合并；每段只写自己的目录和报告。

---

## Prompt 1：原机复现入口审计（不训练）

你在原始远程用户 `yanjingjia` 下工作。执行且只执行任务 `T0_ORIGINAL_MACHINE_AUDIT`。

### 固定路径

- 主项目：`/home/yanjingjia/grid_stable_mamba_probe`
- baseline：`/home/yanjingjia/metric_mamba_gate`
- 数据：`/home/yanjingjia/data/Task04_Hippocampus`
- Python：`/home/yanjingjia/miniforge3/envs/metric-mamba/bin/python`
- 新交付目录：`/home/yanjingjia/gridcons_followup_20260720/00_original_audit`

### 动作

1. 确认四个固定路径存在、可读；记录主机、GPU、驱动、CUDA、Python、PyTorch及磁盘余量。
2. 在 `/home/yanjingjia/grid_stable_mamba_probe` 执行原 `SHA256SUMS` 校验，并单独校验数据归档哈希。
3. 运行：
   ```bash
   PYTHONDONTWRITEBYTECODE=1 /home/yanjingjia/miniforge3/envs/metric-mamba/bin/python -m unittest -v test_probe.py test_paired_full.py
   ```
4. 只读核验已有 `paired_full` 证据：40个confirm病例、5个seed、3种方法、3600行唯一metric、无failure、checkpoint/source/protocol哈希一致。
5. 不修改冻结源码、输出、manifest、checkpoint或原ZIP；不运行训练、开发集筛选或confirm评价。

### 输出

- `ORIGINAL_MACHINE_AUDIT.md`
- `ORIGINAL_MACHINE_AUDIT.json`
- 最终状态只能是 `PASS_ORIGINAL_MACHINE_AUDIT` 或 `BLOCKED_ORIGINAL_MACHINE_AUDIT`。

只有全部检查通过才输出 `PASS_ORIGINAL_MACHINE_AUDIT`。失败时保留命令、退出码和原始错误后停止，不修源码，不继续Prompt 2。

---

## Prompt 2：Task04独立对照矩阵（补标准3D骨干，也补缺失的2×2 Mamba对照）

前置条件：`/home/yanjingjia/gridcons_followup_20260720/00_original_audit/ORIGINAL_MACHINE_AUDIT.json` 的状态必须为 `PASS_ORIGINAL_MACHINE_AUDIT`。执行任务 `T1_TASK04_CONTROL_MATRIX`。

### 目的

当前冻结实验只有：

- dynamic-delta，无consistency；
- static-delta，无consistency；
- static-delta + paired-grid consistency（GridCons-Mamba）。

它不能区分“static delta与consistency存在交互”还是“任何骨干加consistency都会提升”。本任务新增三个独立arm：

1. `dynamic_delta_consistency`：dynamic-delta + 完全相同的paired-grid consistency；
2. `unet3d`：标准plain 3D U-Net，只用R1监督；
3. `unet3d_consistency`：同一个3D U-Net + 完全相同的paired-grid consistency。

### 隔离边界

- 原项目、原结果、原checkpoint保持只读。
- 所有代码、配置和输出仅写入：`/home/yanjingjia/gridcons_followup_20260720/01_task04_control_matrix`。
- 可以复用原项目的数据加载、split、alignment、loss、metric和bootstrap实现，但必须记录来源文件SHA-256。
- 不得改split、R1/R2/R4构造、训练更新数、验证频率、优化器、consistency权重/热身、checkpoint选择规则或seed。
- 不得查看原40-case confirm结果来选择新arm的结构或超参数；3D U-Net结构在训练前冻结。

### 标准3D U-Net定义

实现无注意力、无Mamba、无Transformer的plain 3D U-Net：四层encoder-decoder、每层两个`3×3×3 Conv + normalization + activation`、skip connection、`2×`下采样/上采样、最终`1×1×1`分类头。基础通道固定为16；除输出类别数外不做任务特化。训练前记录参数量和结构文本。不要引入新颖模块。

### 冻结协议

- 数据和split：原Task04的80 train / 20 validation / 20 development / 40 confirm，病例ID必须逐项复用。
- seed：`17, 29, 41, 53, 67`。
- 训练：2500 updates；每250 updates验证；AdamW `lr=3e-4, weight_decay=1e-5`；bfloat16。
- checkpoint：只按R1 validation前景mean Dice选择。
- consistency：只用无标签R2预测；stop-gradient R1 teacher；foreground soft-Dice；权重线性热身至0.1；R2标签绝不进入训练或选模。
- 评价：完全复用R1 Dice、R2 consistency、R2 mapped Dice、surface Dice@2mm、R4 consistency及hierarchical paired bootstrap。

### 执行顺序

1. 冻结 `CONTROL_PROTOCOL_PRETRAINING.json`、三个arm的代码哈希、split哈希、环境和命令。
2. 写最小测试：UNet输出shape/finite；consistency为0时退化为原监督训练；R2 label未进入train graph；同一病例物理对齐检查。
3. 先在20-case development集运行三个arm。预先冻结release rule：相应consistency arm相对其无consistency reference的mean R2 consistency增益>0，且mean R1 Dice差值不低于`-0.01`。
4. 只有代码、数据、checkpoint和development gate全部通过，才一次性解封40-case confirm。
5. confirm后不得调参、改模型、换seed或重跑挑结果；工程失败可以同配置重跑并完整记录。

### 必报比较

- `dynamic_delta_consistency - dynamic_delta`
- `gridcons_mamba - static_delta`
- `unet3d_consistency - unet3d`
- Mamba difference-in-differences：
  `[(gridcons_mamba-static_delta) - (dynamic_delta_consistency-dynamic_delta)]`
- 每个比较报告5 seed、40病例的paired mean、hierarchical 95% CI、R1 guardrail。
- 报告参数量、单病例batch-1 latency、peak allocated memory；硬件和计时协议必须相同。

### 解释规则（训练前冻结）

- 若Mamba difference-in-differences的95% CI下界不大于0：禁止声称static delta是consistency增益的必要来源。
- 若`unet3d_consistency-unet3d`通过：结论是consistency具有跨骨干效应；不得把全部增益归因于Mamba。
- 若U-Net control失败而GridCons通过：只能写“观察到架构相关交互”，不能写“Mamba普遍优于U-Net”。
- 负结果照实保留，禁止换指标救结果。

### 输出

写入任务目录：源码、测试、冻结协议、训练日志、checkpoint、逐病例metrics、统计脚本、`CONTROL_RESULTS.json`、`CONTROL_REPORT.md`、`SHA256SUMS`。

最终状态只能是：

- `PASS_CONTROL_EVIDENCE_COMPLETE`
- `FAIL_CONTROL_SCIENTIFIC`
- `BLOCKED_CONTROL_ENGINEERING`

三者都代表任务已经给出真实结论；只有缺文件、泄漏、协议漂移或未完成运行才可写`BLOCKED`。

---

## Prompt 3：独立数据集／真实异质spacing验证（与Prompt 2完全分开）

前置条件：Prompt 2已生成完整、可审计的终态文件。无论Prompt 2科学结果正负，本任务不得修改Prompt 2。执行任务 `T2_EXTERNAL_REAL_SPACING_VALIDATION`。

### 目标

用一个独立公开3D分割数据集检验GridCons机制是否离开Task04后仍成立；若原始NIfTI header确有多种spacing，再同时检验真实异质spacing。优先候选为官方Medical Segmentation Decathlon `Task09_Spleen`，但必须先通过数据门，不能凭名称宣称异质spacing。

### 数据门（先做，失败就停止）

1. 只使用官方来源；记录URL、许可、归档字节数、SHA-256、病例数、标签定义和引用。
2. 数据根优先使用已有 `/home/yanjingjia/data/Task09_Spleen`；不存在时才从官方来源下载到该目录。禁止删除或覆盖已有数据。
3. 审计每个NIfTI的case ID、image-label一一对应、重复/冲突、shape、affine、orientation和三轴spacing。
4. 输出spacing的unique count、每轴min/median/max/IQR；只有真实header存在至少两种spacing triplet时才允许写`REAL_HETEROGENEOUS_SPACING=true`，否则只作为`INDEPENDENT_DATASET=true`。
5. 至少需要40个有标签独立病例、清晰许可、case-disjoint split可执行且无label泄漏。任一失败输出`BLOCKED_EXTERNAL_DATA_GATE`，不训练。

### 实验协议

- 新目录：`/home/yanjingjia/gridcons_followup_20260720/02_external_spacing`。
- 复用同一机制的三个Mamba arm：`dynamic_delta`、`static_delta`、`gridcons_mamba`；只允许调整输入通道、输出类别数、非标签驱动的crop/padding和强度归一化。
- 使用`17, 29, 41`三个seed。
- 对全部有标签病例预先冻结5-fold outer cross-validation；每个outer-train内再固定validation子集，只用validation R1 Dice选checkpoint。每个病例每个seed只作为outer-test一次。
- 训练预算、优化器、updates和consistency超参数默认与Task04相同；若因体积尺寸必须改batch/crop，只能在任何outer-test暴露前依据GPU dry-run一次性冻结，并对三个arm完全一致。
- 若数据存在真实异质spacing：每例保留原生物理网格作为native view，并由训练fold预先冻结的目标spacing生成paired view；所有对齐使用affine/physical coordinates，不允许仅按array shape插值。训练只使用一个view的标签，另一view label-free。
- 若header spacing不异质：仍可做独立数据集验证，但必须生成与Task04相同定义的受控paired grids，并明确标注`SYNTHETIC_PAIRED_GRID_ON_EXTERNAL_DATA`。
- outer-test解封前冻结代码、fold、预处理、primary metric和统计规则；解封后不调参。

### 主要比较和判定

- 主比较：`gridcons_mamba - dynamic_delta`的paired consistency差值。
- 必须同时满足：mean consistency gain>0；native-grid Dice差值`>= -0.01`。
- 报告跨病例和seed的hierarchical 95% CI；5个outer fold全部列出，不用最大fold代表整体。
- 次比较：`gridcons_mamba - static_delta`；同时报告mapped Dice、surface Dice和严重spacing病例分层结果。
- 若主比较CI下界不大于0，输出`FAIL_EXTERNAL_SCIENTIFIC`，不得把Task04结果写成跨数据集成立。
- 若数据独立但spacing不异质，只能支持“跨数据集”，不能支持“真实异质spacing”。

### 输出

- `DATA_GATE.md/json`
- `SPACING_AUDIT.csv/json`
- `EXTERNAL_PROTOCOL_PRETRAINING.json`
- fold/split清单及哈希
- 全部源码、测试、日志、checkpoint、逐病例metrics
- `EXTERNAL_RESULTS.json`
- `EXTERNAL_REPORT.md`
- `SHA256SUMS`

最终状态只能是：

- `PASS_EXTERNAL_SCIENTIFIC`
- `FAIL_EXTERNAL_SCIENTIFIC`
- `BLOCKED_EXTERNAL_DATA_GATE`
- `BLOCKED_EXTERNAL_ENGINEERING`

禁止把工程可运行、数据独立、spacing真实异质和科学效果通过合并成一个PASS；四项必须分别报告。
