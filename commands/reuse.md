---
name: reuse
description: 复用 ML/DL GitHub 项目 - 从 clone 到训练的全流程引导
argument-hint: "<GitHub 链接>"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Agent
  - WebFetch
  - WebSearch
  - AskUserQuestion
  - TaskCreate
  - TaskUpdate
  - TaskList
  - mcp__plugin_github_github__*
---

# ML/DL 项目复用全流程

接收用户提供的 GitHub 仓库链接，按 6 个阶段引导完成项目复用，最终跑通训练和测试。

## 启动流程

解析用户提供的参数：

- 如果参数是完整 GitHub 链接（含 `github.com`），提取 owner 和 repo 名称
- 如果参数是项目名称，先搜索 GitHub 确认仓库，列出候选让用户选择
- 如果没有参数，询问用户提供 GitHub 链接

确认目标仓库后，输出：
1. 仓库名称与用途简述（1-3 句话）
2. 准备开始 6 阶段流程

## 执行阶段

按以下顺序严格执行，每阶段完成后给出总结，确认后再进入下一步。

### 阶段 1：Git Clone

1. 输出 `git clone` 命令（含推荐的本地目录路径）
2. 执行克隆
3. 列出项目顶层目录结构

### 阶段 2 + 3：深度调研 + 环境检查（并行）

同时启动两个 Agent：

1. **启动 `repo-researcher` Agent**：
   - 传入仓库的 owner、repo 名称、本地项目路径
   - Agent 将输出 `PROJECT_NOTES.md` 到项目根目录

2. **启动 `env-checker` Agent**：
   - 传入本地项目路径（用于读取 requirements.txt / setup.py 等）
   - Agent 将输出环境兼容性报告

两个 Agent 完成后：
- 展示 `PROJECT_NOTES.md` 的摘要（项目概述 + 论文要点 + 快速上手路线）
- 展示环境检查结果摘要
- 给出 Conda 环境创建和依赖安装的完整命令序列
- 询问用户确认后执行环境搭建

### 阶段 4：数据集准备

1. 向用户提问确认：
   - 数据集存放路径（训练集/验证集/测试集）
   - 数据集标签格式（COCO / YOLO / VOC / 其他）
   - 类别列表和数量
   - 目录结构是否已整理完成

2. **启动 `data-inspector` Agent**：
   - 传入数据集路径、标签格式、类别信息
   - 传入 `PROJECT_NOTES.md` 中的数据格式要求
   - Agent 将输出到 `data_check/` 目录

3. 展示可视化结果和异常报告
4. 如有格式转换需求，确认并执行转换脚本

### 阶段 5：配置文件处理

1. 回顾 `PROJECT_NOTES.md`（特别是代码结构和数据格式章节）
2. 浏览 `configs/` 目录，找出最相关的配置
3. 说明选择理由
4. 复制并重命名配置文件
5. 列出需修改的参数清单（区分「必须确认」和「可用默认值」）
6. 逐项修改，先给冒烟测试版本
7. 提醒更新 `PROJECT_NOTES.md` 修改日志

### 阶段 6：训练测试

**阶段 A - 冒烟测试**：
1. 给出冒烟测试命令（少量 epoch，batch_size=2）
2. 执行并监控输出
3. 如失败：参考错误诊断表定位问题，修复后重试
4. 如成功：展示日志摘要，确认一切正常

**阶段 B - 自动 Batch Size 搜索**（目标：显存占用 ~95%）：
1. 通过 `nvidia-smi` 获取 GPU 总显存
2. 以冒烟测试 batch_size 为下界，执行二分搜索：
   - 每轮修改配置中的 batch_size，启动训练跑 5-10 个 iteration
   - 并行用 `nvidia-smi` 采样峰值显存占用
   - OOM → 上界下调；利用率 < 90% → 下界上调
   - 目标：显存占用在 90%~95% 之间
3. 找到最优 batch_size 后，按线性缩放规则调整学习率：
   - `lr_new = lr_base * batch_new / batch_base`
4. 向用户展示搜索结果表格，确认后更新配置文件
5. 如果项目支持 AMP，分别给出 FP32 和 AMP 两个推荐值

搜索细节参见 `references/batch-size-finder.md`。

**阶段 C - 正式训练**：
1. 使用阶段 B 确定的 batch_size 和学习率
2. 给出完整训练命令
3. 说明输出目录：日志、checkpoint、评估结果的保存位置
4. 给出监控训练进度的方法
5. 给出测试/验证命令

## 每步输出格式

每个阶段的回复遵循以下结构：

```
## 当前阶段：[阶段名]

### 已确认的信息
- ...

### 需要你提供的信息（如有）
- ...

### 执行命令 / 操作步骤
（可复制执行的命令）

### 预期结果
- ...

### 常见报错与排查
- ...

### 下一步
- ...
```
