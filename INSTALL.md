# ACT-Plus-Plus 部署指南

本文档记录了 act-plus-plus 项目的完整部署过程。

## 环境信息

- **部署日期**: 2026-01-13
- **操作系统**: Linux 6.14.0-27-generic
- **部署路径**: `/home/xushuai/repos/act-plus-plus`

## 已安装组件版本

| 组件 | 版本 |
|------|------|
| Python | 3.8.10 |
| PyTorch | 2.4.1+cu121 |
| MuJoCo | 2.3.7 |
| dm_control | 1.0.14 |
| robomimic | 0.3.1 |
| diffusers | 0.11.1 |
| huggingface_hub | 0.14.1 |
| OpenCV (headless) | 4.12.0 |

## 部署步骤

### 1. 安装 Miniconda

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O /tmp/miniconda.sh
bash /tmp/miniconda.sh -b -p $HOME/miniconda3
```

接受服务条款：
```bash
source ~/miniconda3/bin/activate
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r
```

### 2. 创建 Conda 环境

```bash
source ~/miniconda3/bin/activate
conda create -n aloha python=3.8.10 -y
conda activate aloha
```

### 3. 克隆仓库

```bash
cd /home/xushuai/repos
git clone https://github.com/MarkFzp/act-plus-plus.git
```

### 4. 安装核心依赖

```bash
conda activate aloha

# PyTorch
pip install torch torchvision

# 其他核心依赖
pip install pyquaternion pyyaml rospkg pexpect
pip install mujoco==2.3.7 dm_control==1.0.14
pip install opencv-python-headless matplotlib
pip install einops packaging h5py ipython
```

### 5. 安装 DETR 子模块

```bash
cd /home/xushuai/repos/act-plus-plus/detr
pip install -e .
```

### 6. 安装 robomimic (Diffusion Policy 支持)

原始仓库的 r2d2 分支已不存在，使用包含 diffusion_policy 的 fork：

```bash
cd /home/xushuai/repos/act-plus-plus
git clone https://github.com/AaronCaoZJ/RoboMimic.git robomimic
cd robomimic
pip install -e . --no-deps
```

安装 robomimic 依赖：
```bash
pip install imageio imageio-ffmpeg psutil tensorboard tensorboardX termcolor
pip install diffusers==0.11.1
pip install huggingface_hub==0.14.1
```

**注意**: huggingface_hub 需要使用 0.14.1 版本以兼容 diffusers 0.11.1

### 7. 验证安装

```bash
source ~/miniconda3/bin/activate
conda activate aloha
cd /home/xushuai/repos/act-plus-plus

python -c "
import torch
print(f'PyTorch: {torch.__version__}, CUDA: {torch.cuda.is_available()}')
import mujoco
print(f'MuJoCo: {mujoco.__version__}')
import robomimic
print(f'robomimic: {robomimic.__version__}')
from robomimic.algo.diffusion_policy import replace_bn_with_gn
print('diffusion_policy: OK')
from policy import ACTPolicy, DiffusionPolicy
print('All policies: OK')
"
```

## 目录结构

```
act-plus-plus/
├── detr/                    # ACT 模型定义
├── robomimic/               # Diffusion Policy 支持 (fork)
├── assets/                  # 仿真资源文件
├── imitate_episodes.py      # 训练和评估 ACT
├── policy.py                # 策略适配器
├── sim_env.py               # MuJoCo + DM_Control 环境
├── ee_sim_env.py            # 末端执行器空间控制环境
├── scripted_policy.py       # 仿真环境脚本策略
├── constants.py             # 共享常量
├── utils.py                 # 工具函数
├── record_sim_episodes.py   # 记录仿真数据
├── visualize_episodes.py    # 可视化数据集
└── INSTALL.md               # 本文档
```

## 使用方法

### 激活环境

```bash
source ~/miniconda3/bin/activate
conda activate aloha
cd /home/xushuai/repos/act-plus-plus
```

### 生成仿真数据

```bash
python3 record_sim_episodes.py \
    --task_name sim_transfer_cube_scripted \
    --dataset_dir <data_save_dir> \
    --num_episodes 50
```

添加 `--onscreen_render` 可查看实时渲染。

### 训练 ACT

```bash
python3 imitate_episodes.py \
    --task_name sim_transfer_cube_scripted \
    --ckpt_dir <ckpt_dir> \
    --policy_class ACT \
    --kl_weight 10 \
    --chunk_size 100 \
    --hidden_dim 512 \
    --batch_size 8 \
    --dim_feedforward 3200 \
    --num_epochs 2000 \
    --lr 1e-5 \
    --seed 0
```

### 评估策略

添加 `--eval` 标志运行评估：

```bash
python3 imitate_episodes.py \
    --task_name sim_transfer_cube_scripted \
    --ckpt_dir <ckpt_dir> \
    --policy_class ACT \
    --eval
```

添加 `--temporal_agg` 启用时序集成。

## 注意事项

1. **图形界面要求**: 仿真环境需要 DISPLAY 环境变量，无图形界面环境下无法运行仿真
2. **robomimic 来源**: 使用 AaronCaoZJ/RoboMimic fork，包含 diffusion_policy 模块
3. **OpenCV**: 使用 headless 版本避免 libGL 依赖问题
4. **EGL 渲染**: 如需 EGL 支持，需安装系统包 `libegl1-mesa-dev`

## 参考链接

- 项目仓库: https://github.com/MarkFzp/act-plus-plus
- robomimic fork: https://github.com/AaronCaoZJ/RoboMimic
- Mobile ALOHA: https://mobile-aloha.github.io/
- ACT 调优指南: https://docs.google.com/document/d/1FVIZfoALXg_ZkYKaYVh-qOlaXveq5CtvJHXkY25eYhs
