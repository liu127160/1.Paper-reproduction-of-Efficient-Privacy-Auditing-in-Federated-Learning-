# Efficient Privacy Auditing in Federated Learning：复现报告

> 论文：**Efficient Privacy Auditing in Federated Learning**（USENIX Security 2024）  
> 官方代码：[changhongyan123/privacy_auditing_in_FL](https://github.com/changhongyan123/privacy_auditing_in_FL)  
> 论文页面：[USENIX Security 2024](https://www.usenix.org/conference/usenixsecurity24/presentation/chang)  
>
> 复现者：刘明鑫

---

## 方法及步骤

本论文研究的是：联邦学习虽然不直接共享原始数据，但模型在训练过程中可能对训练样本表现出与非成员不同的变化规律，从而泄露成员身份。

论文提出的主要方法是 **FTA（Free Training Attack，免额外训练攻击）**。其基本过程如下：

1. 在每个通信轮次，获得当前的 global 或 local 模型快照；
2. 将 `train`、`test` 和非成员参考集 `rest` 中的样本输入该模型；
3. 对每个样本记录 loss、confidence、rescaled logit 等逐轮性能信号；
4. 对同一个样本跨轮次的信号进行线性拟合，得到变化斜率；
5. 使用 `rest` 构成已知非成员参考分布，将审计分数转换为相对 p 值；
6. 扫描多个 p 值阈值，计算 ROC、AUC 及不同低 FPR 条件下的 TPR，以量化隐私泄露风险。

论文还提出了 KTA（Knowledge Transfer Reference Attack，知识迁移参考攻击）作为扩展方法：

- 对目标样本 A，训练多份参考模型；
- 一部分参考模型的训练集包含 A，称为 IN 模型；
- 另一部分参考模型的训练集不包含 A，称为 OUT 模型；
- 由此获得样本 A 的 IN / OUT 两类斜率分布；
- 再判断 A 在目标模型中的斜率更符合哪一类分布，从而推断其是否更可能为成员。

> 本仓库复现的是官方代码中的 FTA 及其基线方法；KTA 用于理解论文的扩展思路，未在本次代码实验中完整实现。

### 理论依据

成员样本在训练中被模型直接使用，因此模型通常会更快地学习它们。相较于未参与训练的非成员样本，成员样本的 loss 往往下降更快，confidence 往往上升更快。

因此，若成员与非成员的跨轮轨迹斜率存在明显差异，攻击者就可能利用这种差异推断样本是否参与过训练。成员和非成员越容易区分，隐私泄露风险越高。

### 应用场景

该方法适用于需要持续观察隐私风险的联邦学习场景，例如医疗、金融、移动终端等数据不能直接集中共享的任务。

它的作用不是直接保护数据，而是帮助研究者或系统管理员回答：

- 训练过程是否泄露了成员身份；
- 哪个通信轮次的泄露风险更明显；
- global 模型与 local 模型的风险是否不同；
- 哪一种审计方法在低误报条件下更有效。

---

## 1. 复现工作概述

本项目在 WSL2 环境下复现论文提出的联邦学习训练期隐私审计流程。目标是完成从**数据划分、联邦训练、逐轮轨迹保存、成员推断审计到结果可视化**的完整链路。

本次已完成的工作：

- 原始 `environment.yml` 在 Linux 系统上导出，且混入多个无关项目的依赖，存在版本冲突。因此本次复现使用适用于 WSL2 的精简环境文件 `environment-wsl2-minimal.yml`；
- 在 CIFAR-10 数据集上，采用 ResNet56 模型和 FedAvg 联邦学习算法，设置 4 个客户端，完成 100 轮联邦训练；
- 将 `fl.test_batch_size` 设置为 64。默认值 1000 在单张 4 GB GPU、4 个客户端并行时会带来较大的资源压力，训练可能停滞；
- 保存客户端 0 在每轮 global / local 模型上的逐样本预测轨迹，包含 `train`、`test`、`rest` 样本；
- 分别审计 global 与 local 模型，并比较 11 种审计分数；
- 复现论文风格的轨迹图、斜率分布图、local/global 结果表，以及 AUC / 低 FPR TPR 随轮数变化图；
- 修改输出目录命名规则，防止不同轮数的实验结果互相覆盖；
- 增加图表和结果汇总脚本。

本复现实验的主要配置如下：

| 项目 | 设置 |
|---|---|
| 数据集 | CIFAR-10 |
| 数据划分 | IIDK，train 0.3，test 0.3，rest 0.4 |
| 联邦算法 | FedAvg |
| 模型 | ResNet56 |
| 客户端数 | 4 |
| 通信轮数 | 100 |
| 本地 epoch | 1 |
| 训练 batch size | 32 |
| 测试 batch size | 64 |
| 优化器 | Adam |
| 学习率 / 权重衰减 | 0.001 / 0.0001 |
| 随机种子 | 0 |
| 审计对象 | client 0 的 global 和 local 模型 |

---

## 2. 审计数据与审计分数

在每个通信轮次，项目会让目标模型对客户端 0 的三类样本进行逐样本预测：

- `train`：该客户端真正参与训练的样本，记为成员；
- `test`：未参与该客户端训练的样本，记为非成员；
- `rest`：额外的非成员参考集，用于构造非成员参考分布。

对每个样本，保存其跨轮变化的 `loss`、正确类别 `confidence`、`rescaled logit` 与预测正确性等信息。

论文提出的 FTA 分数在代码中对应：

- `our_loss`：loss 跨通信轮的线性趋势斜率；
- `our_conf`：正确类别置信度跨通信轮的线性趋势斜率；
- `our_rescaled`：rescaled logit 跨通信轮的线性趋势斜率。

成员样本通常被模型学习得更快，因此其轨迹斜率与非成员存在分布差异；差异越容易区分，成员推断风险越高。

---

## 3. 复现结果与分析

### 3.1 置信度轨迹与斜率分布

论文中：

![输入图片说明](figure1.png)

本次复现：

![Figure 1-style result](figures/figure1_local_confidence_slope_hist.png)

图中以 client 0 的 local 模型为例：

- 左图可看出成员样本的平均 confidence 整体高于非成员；
- 中图可看出成员样本的 confidence 变化更快；
- 在第 30 轮时，成员的单样本 confidence 斜率分布整体更靠右；
- 两类样本仍存在重叠，但已经形成可利用的成员区分信号。

### 3.2 Local 模型审计结果

论文中：

![Table 2](table2.png)

本次复现：

![Table 2-style local result](figures/table_2-style_local_cifar10.png)

在本次单次运行、client 0、100 轮的设置下：

- 在 FPR 限制严格的条件下，FTA 的 TPR 优于大多数基线方法；
- `our_conf` 在低误报成员推断场景中表现较明显。

这说明 FTA 的轨迹斜率在本地模型审计中能够提供有效成员信号。

### 3.3 Global 模型审计结果

论文中：

![Table 3](table3.png)

本次复现：

![Table 3-style global result](figures/table_3-style_global_cifar10.png)

在 global 模型上，`our_conf` 在低误报区间表现最强：

- `TPR @ 0.1% FPR = 7.2000%`；
- `TPR @ 0.5% FPR = 10.9556%`；
- `TPR @ 1% FPR = 13.9111%`。

该趋势与论文的结论一致：使用训练过程中的多轮轨迹，尤其是 confidence 的变化趋势，可以比仅观察最终模型或简单平均分数更有效地审计隐私风险。

### 3.4 随训练推进的风险变化

![AUC versus rounds](figures/auc_vs_rounds_global_local.png)

![TPR at low FPR versus rounds](figures/tpr_at_fpr_0_5_vs_rounds_global_local.png)

这两张图分别展示：

- 整体可分性，即 AUC，如何随通信轮数变化；
- 严格低误报条件下的实际攻击能力，即 `TPR @ 0.5% FPR`，如何随通信轮数变化。

在本次 CIFAR-10、4 客户端、client 0、seed 0 的复现实验中，FTA 的 confidence slope（`our_conf`）在 global 和 local 模型上均表现出最显著的低误报成员推断能力。

但 FTA 的 loss 与 rescaled-logit 变体并非在所有设置下都优于其他方法，这一点与论文表格中的现象一致。

---

## 4. 快速启动指南

### 4.1 环境

建议在 WSL2 / Linux 下运行，并确认以下组件可用：

- Python 3.9；
- PyTorch CUDA；
- OpenMPI；
- mpi4py；
- NumPy；
- Pandas；
- scikit-learn；
- Hydra；
- Matplotlib。

```bash
conda activate auditing_in_fl
