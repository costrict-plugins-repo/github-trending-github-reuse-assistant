# 配置文件关键参数清单

修改配置文件时，按此清单逐项检查和修改。标注为「必须确认」的参数不能使用默认值，需向用户确认。

## 参数总表

| 类别 | 参数名（常见写法） | 作用 | 推荐默认值 | 是否必须确认 |
|------|-------------------|------|-----------|-------------|
| **数据** | `data_root` / `data_path` | 数据集根目录 | - | 必须确认 |
| **数据** | `num_classes` / `num_class` | 类别数 | - | 必须确认 |
| **数据** | `classes` / `class_names` | 类别名称列表 | - | 必须确认 |
| **数据** | `train_ann_file` | 训练集标注文件路径 | - | 必须确认 |
| **数据** | `val_ann_file` | 验证集标注文件路径 | - | 必须确认 |
| **数据** | `img_scale` / `input_size` | 输入图像尺寸 | 原项目默认值 | 建议确认 |
| **训练** | `batch_size` / `samples_per_gpu` | 每 GPU 批量大小 | 根据显存估算 | 建议确认 |
| **训练** | `num_workers` / `workers_per_gpu` | 数据加载进程数 | 4 | 可用默认 |
| **训练** | `max_epochs` / `total_epochs` | 总训练轮数 | 冒烟:3, 正式:原项目值 | 可用默认 |
| **训练** | `lr` / `base_lr` | 初始学习率 | 原项目默认值 | 建议确认 |
| **训练** | `optimizer` | 优化器类型 | 原项目默认值 | 可用默认 |
| **训练** | `lr_scheduler` / `lr_config` | 学习率调度策略 | 原项目默认值 | 可用默认 |
| **训练** | `weight_decay` | 权重衰减 | 原项目默认值 | 可用默认 |
| **模型** | `pretrained` / `load_from` | 预训练权重路径 | - | 必须确认 |
| **模型** | `resume_from` / `resume` | 断点续训路径 | None | 可用默认 |
| **输出** | `work_dir` / `output_dir` | 输出目录 | `./work_dirs/{实验名}` | 建议确认 |
| **输出** | `log_interval` | 日志打印间隔 | 50 iterations | 可用默认 |
| **输出** | `checkpoint_interval` / `save_interval` | 模型保存间隔 | 每 epoch | 可用默认 |
| **输出** | `eval_interval` | 评估间隔 | 每 epoch | 可用默认 |
| **其他** | `seed` / `random_seed` | 随机种子 | 42 | 可用默认 |
| **其他** | `fp16` / `amp` / `mixed_precision` | 混合精度训练 | True（建议开启） | 建议确认 |
| **其他** | `gpu_ids` / `device` | GPU 设备指定 | [0] | 建议确认 |
| **其他** | `dist_training` / `launcher` | 分布式训练配置 | none（单卡） | 根据需要 |

## 修改策略

### 冒烟测试配置（先跑通）

以最保守的配置快速验证：

```python
# 冒烟测试推荐修改
max_epochs = 3              # 只训练 3 个 epoch
batch_size = 2              # 最小 batch size
num_workers = 2             # 最少 workers
eval_interval = 1           # 每个 epoch 都评估
log_interval = 10           # 频繁打印日志
checkpoint_interval = 1     # 每个 epoch 保存
```

### 正式训练配置

冒烟测试通过后，先执行自动 Batch Size 搜索（阶段 6B），再设置正式参数：

```python
# 使用自动搜索确定的最优值
batch_size = {阶段 6B 搜索结果}
max_epochs = {原项目值}
num_workers = 4             # 或更多
eval_interval = {原项目值}
log_interval = 50
```

### 学习率线性缩放规则

**batch_size 变化时必须同步调整学习率**：

```python
lr_new = lr_base * (batch_new / batch_base)
```

示例：
- 原始配置：`lr=0.01, batch_size=16`
- 搜索结果：`batch_size=48`
- 新学习率：`lr=0.01 * 48/16 = 0.03`

注意事项：
- DETR 系列模型通常使用较小固定 lr（如 1e-4），不建议线性缩放
- 使用 AdamW 优化器的模型，缩放比例可能需要用平方根：`lr_new = lr_base * sqrt(batch_new / batch_base)`
- 如不确定，优先保持原始 lr，让自动调度器处理

## 配置文件命名规范

推荐命名格式：`{数据集}_{模型}_{日期}_{备注}.py`

示例：
- `coco_faster_rcnn_20260304_smoke.py`（冒烟测试）
- `voc_yolov8_20260304_full.py`（正式训练）
- `custom_mask_rcnn_20260304_finetune.py`（微调）

## 常见配置框架对照

不同框架的配置写法不同，以下是常见框架的关键配置位置：

| 框架 | 配置方式 | 数据路径 | 类别数 | batch size |
|------|---------|---------|--------|-----------|
| MMDetection | Python config | `data_root` | `num_classes` | `samples_per_gpu` |
| Detectron2 | YAML config | `DATASETS.TRAIN` | `MODEL.ROI_HEADS.NUM_CLASSES` | `SOLVER.IMS_PER_BATCH` |
| YOLO (ultralytics) | YAML config | `path` in data.yaml | `nc` | `batch` in CLI |
| 自定义项目 | argparse / yaml | 各项目不同 | 各项目不同 | 各项目不同 |
