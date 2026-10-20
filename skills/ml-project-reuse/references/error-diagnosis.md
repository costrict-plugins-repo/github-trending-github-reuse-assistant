# 训练常见错误诊断表

冒烟测试和正式训练阶段遇到报错时，参考此表快速定位和修复。

## GPU / 显存相关

### CUDA out of memory
```
RuntimeError: CUDA out of memory. Tried to allocate XXX MiB
```
**诊断步骤**：
1. 降低 `batch_size`（每次减半，直到不 OOM）
2. 开启混合精度训练（AMP）：通常节省 30-50% 显存
3. 减小输入分辨率（如 1024→512）
4. 减少 `num_workers`（每个 worker 也占显存）
5. 使用 `gradient_accumulation`（等效大 batch）
6. 检查是否有内存泄漏（`torch.cuda.empty_cache()`）

### CUDA version mismatch
```
RuntimeError: The NVIDIA driver on your system is too old
```
**诊断步骤**：
1. 运行 `nvidia-smi` 查看 Driver 版本
2. 运行 `nvcc -V` 查看 CUDA Toolkit 版本
3. 核对 PyTorch 官网的版本对应表
4. 如果 Driver 版本低，需要升级 NVIDIA 驱动
5. 如果 CUDA 版本不匹配，重新安装对应版本的 PyTorch

## 模型相关

### size mismatch / key mismatch
```
RuntimeError: Error(s) in loading state_dict: size mismatch for xxx
```
**诊断步骤**：
1. 检查 `num_classes` 是否与数据集类别数一致
2. 检查预训练权重是否与当前模型架构匹配
3. 尝试 `strict=False` 加载权重（跳过不匹配的 key）
4. 打印 `model.state_dict().keys()` 与权重文件 keys 对比
5. 如有 head 不匹配，检查是否需要删除分类头后 fine-tune

### loss = nan / inf
```
Training loss suddenly becomes nan or inf
```
**诊断步骤**：
1. 降低初始学习率（除以 10）
2. 检查数据标注是否有异常值（坐标越界、空标注）
3. 开启 gradient clipping（如 `max_norm=1.0`）
4. 检查数据归一化是否正确（是否除以 255）
5. 检查是否有除零操作（某些 loss 函数在特殊情况下）
6. 关闭 AMP 排查是否为精度问题

### loss 不下降
```
Training loss stays at initial value or decreases very slowly
```
**诊断步骤**：
1. 检查学习率是否过小
2. 检查数据是否正确加载（可视化一个 batch）
3. 检查标签与图像是否对应（可视化确认）
4. 检查模型是否正确初始化（权重加载是否成功）
5. 尝试在极小数据集（10 张图）上过拟合测试

## 数据加载相关

### FileNotFoundError
```
FileNotFoundError: [Errno 2] No such file or directory: 'xxx'
```
**诊断步骤**：
1. 检查配置文件中的 `data_root` 路径是否正确
2. 检查图像/标签文件是否实际存在
3. 检查路径分隔符（Windows: `\` vs Linux: `/`）
4. 检查是否有相对路径与绝对路径混用
5. 在 WSL 中检查 Windows 路径映射（`/mnt/c/...`）

### DataLoader worker killed / 卡住
```
RuntimeError: DataLoader worker (pid xxx) is killed
或训练无输出、进度条不动
```
**诊断步骤**：
1. 降低 `num_workers`（设为 0 排查是否为多进程问题）
2. 检查系统内存是否足够（`free -h`）
3. 检查共享内存大小（`df -h /dev/shm`）
4. 检查数据文件是否损坏（逐个读取测试）
5. 在 Docker 中需增加 `--shm-size`

### 类别不匹配
```
IndexError: index xxx is out of bounds for dimension 0 with size yyy
```
**诊断步骤**：
1. 检查 `num_classes` 配置是否包含背景类
2. 检查标签 ID 是否从 0 开始（有些数据集从 1 开始）
3. 核对类别映射表（config 中 classes 列表 vs 数据标注）
4. 生成类别分布统计，确认无超出范围的类别 ID

## 依赖相关

### ModuleNotFoundError
```
ModuleNotFoundError: No module named 'xxx'
```
**诊断步骤**：
1. 确认 Conda 环境已激活
2. `pip install xxx`（或 `pip install -e .` 安装项目本身）
3. 检查包名与 import 名是否不同（如 `pip install opencv-python` → `import cv2`）
4. 检查是否需要编译安装（如 mmcv、detectron2）

### 编译错误
```
error: command 'gcc' failed / nvcc fatal error
```
**诊断步骤**：
1. 检查 gcc/g++ 版本（某些库要求特定版本）
2. 检查 CUDA_HOME 环境变量是否正确设置
3. 检查 ninja 是否安装（`pip install ninja`）
4. 尝试 `pip install` 预编译包而非源码编译
5. 检查 PyTorch 与 CUDA Toolkit 版本是否对应

## 多卡训练相关

### NCCL 错误
```
RuntimeError: NCCL error / ProcessGroup not initialized
```
**诊断步骤**：
1. 确认 NCCL 版本与 PyTorch 兼容
2. 检查 `MASTER_ADDR` 和 `MASTER_PORT` 设置
3. 检查防火墙是否阻止了端口通信
4. 尝试设置 `NCCL_DEBUG=INFO` 获取详细日志
5. 单卡先跑通再尝试多卡
