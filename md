你正在一台新的 Ubuntu GPU 主机上工作。不要只给计划，依次实际执行定位、审计、实现、测试和 Full 冻结流程。

目标：
为 GridCons 论文补充唯一必要的 equal-exposure 对照实验：
GRIDCONS_EQEXP_C0 / DualGrid-Sup。

用户明确要求：
- 跳过 A0/500-update pilot；
- 只做必要的 2-update CUDA 冒烟；
- 冒烟通过后冻结 Full bundle；
- 必须等待用户返回精确批准口令后，才启动 Full；
- 不修改论文、PDF或现有实验结果；
- 不开展新数据集、超参数搜索或其他消融。

一、不可违反的边界

1. 这是新主机，不得假设项目位于 ~/Yanjingjia 或任何旧主机路径。
2. 不得从 / 开始无边界扫描。
3. 只允许在以下位置进行只读、有深度限制的定位：
   - 当前目录及其父目录；
   - $HOME 下最多 4 层；
   - /workspace、/data、/mnt/data、/scratch、/opt/project（存在时）。
4. 不得删除、移动或覆盖现有源码、数据、环境、checkpoint 和结果。
5. 不得修改原始数据。
6. 不得使用 sudo，不得全局升级 pip/conda/PyTorch/CUDA。
7. 不得下载数据或源码；如果关键内容缺失，停止并报告。
8. 所有新增代码和结果必须写入原项目旁边的新目录或独立 git worktree：
   GridCons_equal_exposure_control_20260721
9. held-out evaluation split 在 Full 获批并完成所有训练前不得读取指标、调参或用于 checkpoint 选择。
10. 保留 NOT_RUN、UNKNOWN 和 BLOCKED，不得把缺失证据写成 PASS。

二、定位当前有效源码

使用 rg/find 进行有界搜索，重点查找：

- grid_stable_mamba_probe/run_paired_full.py
- grid_stable_mamba_probe/model.py
- CONTROL_PROTOCOL_PRETRAINING.json
- CONTROL_RESULTS.json
- Prompt2_Task04_Control_Matrix
- confirm_manifest.json
- evaluation.json
- metrics.csv
- dynamic_delta_consistency
- gridcons_mamba
- unet3d_consistency

也检查 ZIP/TAR 源码包，但不要仅凭文件名认定其有效。

如果发现文件名完全为：

GridCons-Mamba_MMM_20260719.zip

可核验其历史 SHA-256：

57771e44be3c577d5902589f899440fa77a25728e01cc702bdc1dbb5afc55c00

这个哈希只能证明它是历史包，不能自动证明它是当前 canonical source。

canonical source 必须能够追溯当前 Task04 六臂矩阵：

- dynamic_delta
- dynamic_delta_consistency
- static_delta
- gridcons_mamba
- unet3d
- unet3d_consistency

检查训练入口、配置、日志、checkpoint、case-level metrics 和 split manifest 是否互相对应。

若出现多个候选源码且无法通过配置、结果文件、时间、git 状态和哈希唯一确定，立即停止：

SOURCE_STATUS=BLOCKED_SOURCE_AMBIGUOUS
FULL_NOT_RUN

并生成 DISCOVERY_REPORT.md，列出每个候选的绝对路径和证据。不得自行猜选。

三、定位并验证 Task04 数据

有界搜索：

- Task04_Hippocampus.tar
- dataset.json
- imagesTr/
- labelsTr/
- *.nii.gz

若存在原始压缩包，Task04 官方归档的预期 SHA-256 为：

282d808a3e84e5a52f090d9dd4c0b0057b94a6bd51ad41569aef5ff303287771

若只有解压数据，则生成只读数据 manifest，记录：

- 数据根目录绝对路径；
- dataset.json；
- image/label case ID；
- 缺失配对；
- 重复 case；
- 文件数量；
- split manifest 哈希。

必须验证原实验的确定性划分：

- train：80 cases
- validation：20 cases
- development-screen：20 cases
- held-out evaluation：40 cases
- unused：100 cases

确认 patient/case 不重叠。将 held-out case IDs 和 split 文件哈希写入：

OUTER_TEST_LOCK.json

若数据身份、配对或 split 无法确认，停止：

DATA_STATUS=BLOCKED
FULL_NOT_RUN

四、新机环境审计

记录到 ENVIRONMENT_REPORT.md：

- hostname、操作系统、时间；
- nvidia-smi；
- GPU 型号、UUID、显存；
- driver 和 CUDA 版本；
- 当前 GPU 进程；
- Python 路径和版本；
- torch 版本；
- torch.version.cuda；
- torch.cuda.is_available()；
- nibabel、numpy 等实际版本；
- bf16/fp16 支持；
- 当前 conda/venv；
- 项目盘、数据盘和结果盘剩余空间；
- git commit、branch、dirty status；
-确定性设置和环境变量。

优先复用源码声明的已有环境。缺少依赖时，不得全局安装或随意升级；先检查项目中的 requirements、environment.yml、venv、conda env 和运行日志。无法安全恢复环境则停止：

ENV_STATUS=BLOCKED
FULL_NOT_RUN

五、建立隔离工作目录

在 canonical source 的同级目录创建：

GridCons_equal_exposure_control_20260721

如果 canonical source 是干净 git 仓库，优先创建独立 worktree 和分支：

codex/gridcons-equal-exposure-control

如果不是 git 仓库，则复制最小必要源码；不得复制数据、旧 checkpoint 或大结果目录。数据只通过只读绝对路径引用。

记录：

- canonical source 绝对路径；
- 工作目录绝对路径；
- source commit；
- source dirty patch；
- 新增/修改文件；
- 全部文件 SHA-256。

六、实现唯一的新对照

候选 ID：

GRIDCONS_EQEXP_C0

三个新 arm：

- dynamic_delta_dualgrid_sup
- static_delta_dualgrid_sup
- unet3d_dualgrid_sup

必须复用现有 GridCons 的：

- R1/R2 构造；
- physical-grid mapping；
- crop/augmentation；
- backbone；
- optimizer；
- batch size；
- learning-rate schedule；
- precision；
- checkpoint selection；
- update 数；
- evaluation code。

只替换训练目标。冻结目标为：

L_control(t)
  = L_seg(pred_R1, label_R1)
  + lambda(t) * L_seg(pred_R2, label_R2)

lambda(t) = 0.1 * min(1, t / 500)

其中：

- label_R2 必须由现有 grid/resampling 工具从同一原始标签确定性生成；
- 标签插值必须为 nearest-neighbor；
- 不得使用 GridCons prediction-consistency loss；
- 不得把 R1 prediction 作为 teacher；
- 不得改变模型结构；
- 不得增加推理分支；
- 不得调节 lambda、warm-up 或其他超参数。

这个对照检验的是：

“GridCons 的改善是否可以被简单的额外 R2 暴露/双网格监督解释。”

若源码中现有冻结协议不是 2500 updates，或者 lambda/warm-up 与上述论文协议冲突，不得自行选择；停止并报告 PROTOCOL_CONFLICT。

七、最小检查

实现后必须执行最小单元检查：

1. R2 label shape 与 R2 prediction 一致；
2. R2 label 只含合法类别；
3. nearest-neighbor 重采样不产生新类别；
4. R2 supervised term 对模型参数产生非零有限梯度；
5. 总 loss、各分量和梯度均为有限值；
6. lambda(0)=0；
7. lambda(250)=0.05；
8. lambda(500)=0.1；
9. lambda(2500)=0.1；
10. 三个新 arm 的非目标配置与对应 GridCons arm 一致；
11. 数据 loader 在 smoke 阶段不能访问 held-out case；
12. OUTER_TEST_LOCK.json 哈希保持不变。

失败则停止：

IMPLEMENTATION_STATUS=BLOCKED
FULL_NOT_RUN

八、CUDA 冒烟，不运行 A0

只运行以下 3 个 smoke job：

- dynamic_delta_dualgrid_sup，seed 17，2 updates；
- static_delta_dualgrid_sup，seed 17，2 updates；
- unet3d_dualgrid_sup，seed 17，2 updates。

要求：

- 只使用 train/validation 数据；
- 不运行 held-out evaluation；
- 验证 forward、backward、optimizer step、checkpoint 写入；
- 记录显存峰值、耗时和 GPU UUID；
- loss 和 gradient 必须有限；
- 不允许修改 batch size、crop size或模型以迁就 OOM。

若 OOM 或环境错误，停止并报告真实原因；不得静默降低实验规格。

成功后写：

SMOKE_STATUS=PASS
A0_STATUS=NOT_RUN_USER_AUTHORIZED_SKIP
OUTER_TEST_NOT_TOUCHED=true

九、确定 Full 实际运行范围

论文现有五个 seeds 固定为：

17, 29, 41, 53, 67

每个新 arm 固定运行 2500 updates，不进行调参。

优先验证已有六臂结果能否满足以下全部条件：

- canonical code 可追溯；
- 数据和 split 哈希一致；
- seeds 一致；
- optimizer、updates、augmentation、precision 和 checkpoint rule 有记录；
- checkpoint 可读；
- case-seed-level raw metrics 完整；
- evaluation 实现一致；
- 没有只剩聚合表、缺少原始结果的情况。

如果全部满足，Full 只运行新增的：

3 backbones × 5 seeds = 15 jobs。

如果某个 backbone 的旧 baseline/GridCons provenance 不完整，则 Full 对该 backbone 重跑公平三臂：

baseline + GridCons + DualGrid-Sup

不得把来源不可验证的旧结果与新结果直接比较。

把最终 job 数量、逐 job 命令、预计 GPU 时间和磁盘需求写入 FULL_PLAN.md。单 GPU 串行执行；不得让多个训练进程争抢同一 GPU。

十、冻结批准包并硬停止

创建 approval_bundle/，至少包含：

- DISCOVERY_REPORT.md
- ENVIRONMENT_REPORT.md
- DATA_MANIFEST.json
- OUTER_TEST_LOCK.json
- SOURCE_MANIFEST.json
- implementation diff
- unit-check logs
- smoke logs
- FULL_PLAN.md
- exact commands
- baseline/provenance decision
- data/split/config/code hashes
- A0_STATUS=NOT_RUN_USER_AUTHORIZED_SKIP

生成规范化 SHA-256 manifest。

此时不得启动任何 2500-update Full job。

最终只输出：

1. PROJECT_PATH=<绝对路径>
2. SURVIVE=0 INCONCLUSIVE=1 KILL=0 MODIFY_RETESTED=0
3. SELECTED=GRIDCONS_EQEXP_C0 USER_DIRECT_FULL
4. DISCOVERY_REPORT、FULL_PLAN、approval_bundle 的绝对路径
5. APPROVE_FULL:GRIDCONS_EQEXP_C0:<bundle_sha256>
6. FULL_NOT_RUN

等待用户逐字返回第 5 项批准口令。

十一、收到精确批准口令后

只有收到完全匹配的：

APPROVE_FULL:GRIDCONS_EQEXP_C0:<bundle_sha256>

才可以：

1. 重新计算 bundle 哈希；
2. 确认源码、数据、split、环境和配置没有漂移；
3. 将状态改为 FULL_RUNNING；
4. 按 FULL_PLAN.md 串行运行全部 Full jobs；
5. 训练期间不得查看 held-out 结果进行调参；
6. 所有训练完成后，才执行一次冻结的 held-out evaluation；
7. 基础设施型失败最多原配置重试一次；
8. 科学结果不好不得换指标、换 seed、改 loss 或补调参。

十二、Full 输出

保存：

- 每个 job 的命令、配置和日志；
- exit code、开始/结束时间、GPU UUID；
- checkpoints；
- case-seed-level raw metrics；
- 每个 seed 的结果；
- 10,000 次 case × seed hierarchical paired bootstrap，bootstrap seed=7301；
- GridCons vs DualGrid-Sup 的 primary endpoint：
  R2 prediction consistency；
- secondary endpoints：
  R2 mapped Dice、R2 surface Dice@2mm、R1 native Dice guardrail、R4 absolute consistency；
- 均值、paired difference 和 95% CI；
- 失败 job 和重试记录。

科学解释必须按以下规则：

- 若 GridCons 相对 DualGrid-Sup 的 R2 consistency paired CI 下界 > 0：
  说明现有结果不能仅由简单双网格监督完全解释。
- 若 CI 跨 0：
  MECHANISM_RESULT=INCONCLUSIVE。
- 若 DualGrid-Sup 与 GridCons 相当或更好：
  MECHANISM_CLAIM=UNSUPPORTED；
  不得隐藏该结果。

最终生成：

- FULL_RESULTS.json
- FULL_RAW_METRICS.csv
- FULL_REPORT.md
- REPRODUCIBILITY_MANIFEST.json
- RESULT_ARCHIVE.tar.gz
- RESULT_ARCHIVE.sha256

最终状态必须明确区分：

ENGINEERING_STATUS=
FULL_EXECUTION_STATUS=
SCIENTIFIC_STATUS=
PAPER_CLAIM_STATUS=
OUTER_TEST_TOUCHED=
FAILED_JOBS=
RERUNS=
UNSUPPORTED_CLAIMS=

禁止修改论文。只报告实验结果和可复现路径。
