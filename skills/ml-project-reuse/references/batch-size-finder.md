# 自动 Batch Size 搜索指南

冒烟测试通过后执行，目标是找到使 GPU 显存利用率达到 ~95% 的最大 batch size。

## 搜索策略

使用二分搜索法，在安全范围内逐步逼近最优值。

### 前置条件

- 冒烟测试已通过（确认代码/数据/配置无误）
- 已知当前冒烟测试使用的 batch_size（作为搜索下界）
- GPU 显存总量已知（通过 `nvidia-smi` 获取）

### 搜索流程

```
输入：
  - bs_smoke: 冒烟测试通过的 batch size
  - gpu_mem_total: GPU 总显存 (MB)
  - target_ratio: 目标显存利用率 (默认 0.95)

算法：
  lo = bs_smoke
  hi = bs_smoke * 16  # 初始上界（通常足够大）
  best_bs = bs_smoke
  best_usage = 0

  while hi - lo > 2:
      mid = (lo + hi) // 2
      # 将 mid 对齐到 2 的倍数（部分框架要求）
      mid = mid - (mid % 2) if mid % 2 != 0 else mid

      修改配置中的 batch_size 为 mid
      启动训练，跑 5-10 个 iteration
      同时用 nvidia-smi 采样峰值显存

      if OOM:
          hi = mid - 2
      else:
          usage = peak_mem / gpu_mem_total
          if usage > 0.97:
              # 太接近上限，有 OOM 风险
              hi = mid - 2
          elif usage >= 0.90:
              # 命中目标区间 [90%, 97%]
              best_bs = mid
              best_usage = usage
              if usage < target_ratio:
                  lo = mid + 2  # 还有空间，尝试更大
              else:
                  break  # 已达目标
          else:
              # 利用率 < 90%，还有余量
              lo = mid + 2
              best_bs = mid
              best_usage = usage

  输出：best_bs, best_usage
```

### 显存采样方法

训练开始后，用独立进程每 0.5 秒采样一次显存峰值：

```bash
# 采样脚本（在训练启动后并行运行）
nvidia-smi --query-gpu=memory.used,memory.total --format=csv,noheader,nounits -l 1
```

或在 Python 中：

```python
import subprocess, time, threading

class GpuMemoryMonitor:
    def __init__(self, gpu_id=0, interval=0.5):
        self.gpu_id = gpu_id
        self.interval = interval
        self.peak_mb = 0
        self.total_mb = 0
        self._running = False

    def _sample(self):
        while self._running:
            try:
                out = subprocess.check_output(
                    ['nvidia-smi', '--query-gpu=memory.used,memory.total',
                     '--format=csv,noheader,nounits', f'--id={self.gpu_id}'],
                    text=True
                ).strip()
                used, total = [int(x.strip()) for x in out.split(',')]
                self.peak_mb = max(self.peak_mb, used)
                self.total_mb = total
            except Exception:
                pass
            time.sleep(self.interval)

    def start(self):
        self.peak_mb = 0
        self._running = True
        self._thread = threading.Thread(target=self._sample, daemon=True)
        self._thread.start()

    def stop(self):
        self._running = False
        self._thread.join(timeout=2)
        return self.peak_mb, self.total_mb

    @property
    def usage_ratio(self):
        if self.total_mb == 0:
            return 0
        return self.peak_mb / self.total_mb
```

### 执行模板

在 Claude Code 中，实际执行搜索的步骤：

1. **读取当前配置中的 batch_size 和 lr 作为基准值**
2. **循环执行**（每轮都向用户展示进度）：

```
[搜索进度] 第 N 轮 | 测试 batch_size=XX | 范围 [lo, hi]
  → 修改配置文件 batch_size
  → 启动训练 (5-10 iter) + 显存采样
  → 结果: 峰值显存 XX MB / XX MB (XX%)
  → 判断: [OOM / 太高 / 命中目标 / 有余量]
```

3. **搜索结束后输出**：

```
## Batch Size 搜索结果

| 项目 | 值 |
|------|-----|
| 推荐 batch_size | XX |
| 显存峰值占用 | XX MB / XX MB (XX%) |
| 基准 batch_size | XX (冒烟测试) |
| 搜索轮数 | X 轮 |

### 学习率线性缩放
- 基准: lr={base_lr}, batch_size={base_bs}
- 推荐: lr={new_lr}, batch_size={new_bs}
- 公式: {new_lr} = {base_lr} * {new_bs} / {base_bs}

⚠️ 请确认新学习率是否合理，部分模型（如 DETR 系列）不建议使用线性缩放。
```

## 特殊情况处理

### AMP 开关对比

如果项目支持混合精度训练（AMP），分别搜索 AMP 开/关两个值：

| 模式 | 推荐 batch_size | 显存占用 | 训练速度 |
|------|----------------|---------|---------|
| FP32 | XX | XX% | 基准 |
| AMP (FP16) | XX | XX% | 约快 30-50% |

默认推荐 AMP 模式的 batch_size。

### 多卡训练

多卡训练时，每卡的 batch size 独立搜索：
- `total_batch_size = per_gpu_batch_size * num_gpus`
- 学习率按 `total_batch_size` 缩放
- 搜索时只在单卡上执行，找到最优 per_gpu_batch_size

### 梯度累积替代方案

如果最优 batch_size 仍然很小（如 ≤ 4），建议使用梯度累积：
- `accumulation_steps = target_batch / actual_batch`
- 等效 batch size = `actual_batch * accumulation_steps`
- 学习率按等效 batch size 缩放

## 安全守则

1. **永远不要从大到小搜索**：OOM 可能导致 GPU 进程残留，需要手动清理
2. **每轮测试前清理 GPU 缓存**：`torch.cuda.empty_cache()` 或直接重启训练进程
3. **预留 3-5% 显存缓冲**：实际训练中某些 batch 可能比平均更大（如含大量目标的图像）
4. **OOM 后等待 GPU 释放**：`nvidia-smi` 确认 GPU 进程已退出再继续下一轮
