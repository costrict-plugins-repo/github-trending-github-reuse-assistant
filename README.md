# GitHub Reuse Assistant

ML/DL GitHub 项目复用助手 - 从 clone 到训练的全流程引导插件。

## 功能概述

提供结构化的 6 阶段工作流，帮助快速复用任意 ML/DL GitHub 项目：

1. **代码获取** - Git clone 并了解项目概况
2. **深度调研** - 通过 DeepWiki、论文、Issues 深入理解项目，生成 `PROJECT_NOTES.md`
3. **环境准备** - 自动检查本机环境，生成 Conda 搭建命令，含 OOM 预估
4. **数据集准备** - 格式检查、自动转换、可视化验证
5. **配置文件** - 找最相关配置，复制修改关键参数
6. **训练测试** - 冒烟测试 → 正式训练，内置错误自动诊断

## 插件组件

| 组件 | 类型 | 功能 |
|------|------|------|
| `ml-project-reuse` | Skill | 6 阶段方法论、调研笔记规范、错误诊断表 |
| `/reuse` | Command | 用户入口，接收 GitHub 链接，启动全流程 |
| `repo-researcher` | Agent | 深度调研：DeepWiki + 论文 + Issues + 代码结构 → PROJECT_NOTES.md |
| `env-checker` | Agent | 环境检测 + 兼容性分析 + OOM 预估 + 安装命令生成 |
| `data-inspector` | Agent | 数据格式检查 + 转换脚本生成 + 可视化输出 |

## 使用方法

### 快速开始

```
/reuse https://github.com/open-mmlab/mmdetection
```

插件会自动引导你完成 6 个阶段，每阶段给出可直接执行的命令。

### 自动触发

当你在对话中提到以下关键词时，相关 Skill 会自动加载：
- "复用一个GitHub项目"、"跑通一个开源项目"
- "clone并运行训练"、"搭建训练环境"
- "准备数据集"、"配置训练参数"

## 安装

### 本地测试

```bash
claude --plugin-dir /path/to/github-reuse-assistant
```

### 项目级安装

将插件目录复制到项目的 `.claude-plugin/` 下。

## 生成的产物

插件在工作过程中会生成以下文件：

| 文件 | 位置 | 用途 |
|------|------|------|
| `PROJECT_NOTES.md` | 项目根目录 | 调研笔记，贯穿全流程的参考文档 |
| `data_check/` | 项目根目录 | 数据验证结果目录 |
| `data_check/visualize/` | 数据检查目录 | 标注可视化图像 |
| `data_check/class_distribution.png` | 数据检查目录 | 类别分布统计图 |
| `data_check/anomaly_report.md` | 数据检查目录 | 数据异常报告 |
| `data_check/convert_dataset.py` | 数据检查目录 | 格式转换脚本（如需要） |

## 支持的项目类型

- 目标检测（YOLO、Faster R-CNN、DETR 等）
- 语义分割（DeepLab、SegFormer 等）
- 实例分割（Mask R-CNN、SOLOv2 等）
- 图像分类（ResNet、ViT、Swin Transformer 等）
- 点云处理（PointNet、PointPillars 等）
- 其他 PyTorch / TensorFlow 训练项目

## 设计原则

- **先跑通、再定制**：优先选择最稳妥的方案
- **不确定就问**：关键参数不猜测，先向用户确认
- **分阶段推进**：每阶段完成后总结确认，再进入下一步
- **输出可执行**：所有命令可直接复制执行
- **多平台兼容**：支持 Linux / macOS / WSL2
