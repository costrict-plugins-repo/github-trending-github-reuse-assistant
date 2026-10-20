---
name: ML 项目复用工作流
description: >-
  This skill should be used when the user asks to "复用一个GitHub项目",
  "跑通一个开源项目", "clone并运行训练", "搭建训练环境",
  "准备数据集", "配置训练参数", "冒烟测试", "跑通训练/测试",
  mentions "GitHub项目复用", "深度学习项目部署", "开源项目落地",
  or provides a GitHub link for an ML/DL project they want to run.
  Covers the full lifecycle from git clone to successful training.
version: 1.0.0
---

# ML/DL GitHub 项目复用工作流

## 概述

提供一套结构化的 6 阶段方法论，将任意 ML/DL GitHub 项目从 clone 代码推进到成功训练和测试。核心原则：**先跑通、再定制**，遇到不确定之处先提问、不猜测关键参数。

## 6 阶段流程

### 阶段 1：代码获取（Git Clone）

根据用户提供的 GitHub 链接：

1. 确认仓库名称与用途（1-3 句话概述）
2. 输出 `git clone` 命令及推荐本地目录
3. 如只有项目名无链接，先搜索确认候选仓库让用户选择

### 阶段 2：深度调研

**启动 `repo-researcher` Agent**，执行以下调研：

1. 通过 DeepWiki（`https://deepwiki.com/{owner}/{repo}`）理解项目架构
2. 从 README 中提取论文链接（arXiv 等），阅读并总结核心方法
3. 分析代码目录结构、核心模块、配置体系
4. 扫描 Issues（Open + Closed），提取常见报错与解决方案
5. 输出结构化调研笔记 `PROJECT_NOTES.md` 到项目根目录

笔记文件结构参见 `references/project-notes-template.md`。

**关键**：此笔记是后续所有阶段的参考依据，在阶段 5 修改配置和阶段 6 训练排错时需回顾。

### 阶段 3：环境准备

**启动 `env-checker` Agent**（可与阶段 2 并行），执行：

1. 检测本机环境：OS、Python、Conda、GPU、NVIDIA Driver、CUDA（系统级 + nvcc）、编译工具链
2. 与项目需求交叉对比，输出兼容性报告
3. 根据 GPU 显存估算推荐 batch size 范围
4. 给出 Conda 环境创建命令 + 依赖安装步骤（分步、含验证命令）
5. 输出常见报错排查清单

对拿不准的版本兼容问题（如 CUDA 与 PyTorch 对应关系），先向用户确认。

### 阶段 4：数据集准备

**交互确认**后**启动 `data-inspector` Agent**：

1. 向用户确认：数据集路径、目录结构、标签格式（COCO/YOLO/VOC/自定义）、类别信息
2. 对比 `PROJECT_NOTES.md` 中的数据格式要求
3. 如格式不匹配 → 生成转换脚本 `data_check/convert_dataset.py`
4. 生成可视化验证文件到 `data_check/` 目录：
   - 图像 + bbox/mask 叠加（随机抽样）
   - 类别分布统计图
   - 标签异常报告（空标注/越界框/缺失文件）
5. 点云任务：语义/实例标签着色、类别统计、空场景检查

可视化结果判定标准参见 `references/data-validation-standards.md`。

### 阶段 5：配置文件处理

1. 阅读项目配置目录（如 `configs/`），回顾 `PROJECT_NOTES.md`
2. 找出与用户需求最相关的配置样例并说明选择理由
3. 复制重命名（建议命名：`{数据集}_{模型}_{日期}.py`）
4. 修改关键参数，参见 `references/config-checklist.md` 中的参数清单
5. 对不确定的参数先给保守默认值，标注待用户确认
6. 每次修改都提醒更新 `PROJECT_NOTES.md` 的修改日志

### 阶段 6：训练测试

采用「先验证再正式」策略：

**阶段 A - 冒烟测试**：
- 训练 2-3 个 epoch 或少量 iteration
- 确认代码、数据、配置、日志、保存路径全部正常
- 若失败 → 参考错误诊断表自动定位（见 `references/error-diagnosis.md`）

**阶段 B - 自动 Batch Size 搜索**（目标：GPU 显存利用率 ~95%）：

冒烟测试通过后，自动执行二分搜索找到最优 batch size：

1. **获取基准信息**：通过 `nvidia-smi` 获取 GPU 总显存和当前空闲显存
2. **二分搜索**：
   - 起始范围：`[冒烟测试 batch_size, 冒烟测试 batch_size * 16]`
   - 每次取中值，启动训练跑 5-10 个 iteration
   - 通过 `nvidia-smi` 采样峰值显存占用
   - 若 OOM → 上界下调；若显存利用率 < 90% → 下界上调
   - 目标区间：显存占用在总显存的 **90%~95%** 之间
   - 搜索终止条件：上下界差值 ≤ 2 或已找到目标区间内的值
3. **输出结果**：
   - 推荐 batch_size 及对应的显存占用百分比
   - 自动更新配置文件中的 batch_size
   - 若开启 AMP 前后差异大，分别给出两个推荐值
4. **线性缩放学习率**：batch size 变化后自动按比例调整 lr
   - `lr_new = lr_base * batch_new / batch_base`
   - 提示用户确认新学习率

具体搜索脚本参见 `references/batch-size-finder.md`。

**阶段 C - 正式训练**：
- 使用阶段 B 确定的 batch size 和学习率
- 给出完整训练命令
- 说明日志/checkpoint/结果保存路径
- 给出查看训练进度的方法（tensorboard / log 关键字段）
- 给出测试/验证命令与结果导出路径

## 工作方式准则

1. **分阶段推进**：每阶段完成后给出总结，确认后进入下一步
2. **不确定就问**：关键参数不确定时先提问
3. **先跑通优先**：选最稳妥方案，不追求最"高级"方案
4. **输出可直接执行**：给出可复制的命令、路径示例、配置修改建议
5. **兼容多平台**：默认 Conda，关注 GPU/CUDA/PyTorch 兼容
6. **记录产物位置**：每次训练/测试明确告知产物保存路径

## 每次回复格式

按以下结构输出：

- **当前阶段**
- **已确认的信息**
- **需要用户提供的信息**（如有）
- **执行命令 / 操作步骤**
- **预期结果**
- **常见报错与排查**
- **下一步计划**

## 附属资源

### 参考文件

- **`references/project-notes-template.md`** - PROJECT_NOTES.md 完整模板（含预训练权重章节、修改日志）
- **`references/error-diagnosis.md`** - 训练常见错误 → 自动诊断建议映射表
- **`references/config-checklist.md`** - 配置文件关键参数清单
- **`references/data-validation-standards.md`** - 数据可视化结果判定标准
- **`references/batch-size-finder.md`** - 自动 Batch Size 二分搜索策略与显存采样方法
