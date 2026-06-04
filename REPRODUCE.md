# IncEventGS 复现指南

## 环境信息

| 项目 | 详情 |
|------|------|
| GPU | NVIDIA RTX 4090 24GB |
| CUDA | 12.4 |
| Driver | 550.107.02 |
| Python | 3.10 (miniconda) |
| OS | Ubuntu (Docker) |

## 1. 克隆代码

```bash
git clone git@github.com:wu-cvgl/IncEventGS.git
cd IncEventGS
```

## 2. 安装依赖

```bash
pip install -r requirements.txt -r requirements_extra.txt

# gsplat (CUDA 加速 3DGS 渲染)
pip install gsplat

# tiny-cuda-nn (NeRF encoding, 需从源码编译)
pip install git+https://github.com/NVlabs/tiny-cuda-nn.git#subdirectory=bindings/torch

# ninja (CUDA kernel JIT 编译)
pip install ninja
```

## 3. 下载权重和数据集

### 3.1 深度估计模型 (Marigold)
 从 HuggingFace 下载 LCM 版本 :

```bash
python -c "
from diffusers import DiffusionPipeline
pipe = DiffusionPipeline.from_pretrained(
    'prs-eth/marigold-lcm-v1-0',
    custom_pipeline='marigold_depth_estimation',
    torch_dtype=torch.float16,
    variant='fp16',
)
"
# 缓存路径: ~/.cache/huggingface/models--prs-eth--marigold-lcm-v1-0/
```

使用作者提供的权重:

```bash
tar -xf depthEst_pretrained_model.tar  # Bingxin/Marigold, 4.9GB
```

### 3.2 数据集

从 [SharePoint](https://westlakeu-my.sharepoint.com/personal/cvgl_westlake_edu_cn/_layouts/15/onedrive.aspx?ga=1&id=%2Fpersonal%2Fcvgl%5Fwestlake%5Fedu%5Fcn%2FDocuments%2FIncEventGS) 下载:

| 文件 | 大小 | 内容 |
|------|------|------|
| event_replica.tar | 44.7 GB | Replica 数据集 (5个场景) |
| tumvie.tar | 8.4 GB | TUM-VIE 数据集 (4个场景) |
| depthEst_pretrained_model.tar | 4.9 GB | 深度模型 (可选, 用HF版本替代) |

**下载慢, 本地下载后 SCP:**

```bash
scp event_replica.tar tumvie.tar depthEst_pretrained_model.tar root@<SERVER>:/data/
```

### 3.3 解压

```bash
cd /data
tar -xf event_replica.tar    # → /data/event_replica/
tar -xf tumvie.tar            # → /data/tumvie/

# 解压后删除 tar 释放空间 (~58GB)
rm -f /data/event_replica.tar /data/tumvie.tar /data/depthEst_pretrained_model.tar
```

## 4. 数据集格式

```
<scene_dir>/
├── images/                      # RGB 图像
│   ├── frame000100.jpg
│   └── ...
├── event_threshold_0.1/         # 事件数据
│   └── gray_events_data.npy    # (N, 4): [x, y, timestamp, polarity]
├── traj.txt                     # N×16: 4×4 c2w 矩阵展平
└── poses_ts.txt                 # N 个时间戳
```

## 5. 配置

创建本地配置文件，覆盖模型和数据路径:

**Replica office0**: `configs/SimuEvent/replica_office0_local.yaml`:

```yaml
inherit_from: configs/SimuEvent/replica_office0.yaml
depthEst_pretrained_model_path: ~/.cache/huggingface/models--prs-eth--marigold-lcm-v1-0/snapshots/04a73502f7fd8fc5e59947b9df3b2266d71d6849
data:
  datadir: /data/event_replica/office0
  output: output/final/office0
```

**TUM-VIE mocap-desk**: `configs/TUM_VIE/mocap-desk_local.yaml`:

```yaml
inherit_from: configs/TUM_VIE/mocap-desk.yaml
depthEst_pretrained_model_path: ~/.cache/huggingface/models--prs-eth--marigold-lcm-v1-0/snapshots/04a73502f7fd8fc5e59947b9df3b2266d71d6849
data:
  datadir: /data/tumvie
  output: output/final/tumvie/mocap-desk
```

## 6. 运行

```bash
cd /data/IncEventGS

# 创建输出目录
mkdir -p output/final/office0/demo/img_eval/gt

# 设置 CUDA 架构 (RTX 3090=8.6, RTX 4090=8.9, A100=8.0)
export TORCH_CUDA_ARCH_LIST="8.9"
export CUDA_VISIBLE_DEVICES=0

# 运行 (首次运行需 JIT 编译 ~2分钟)
python main.py --config configs/SimuEvent/replica_office0_local.yaml
```

## 7. 训练流程

IncEventGS 训练分三个阶段:

### 阶段1: 深度估计初始化 (4500 迭代, ~8分钟)

```
随机初始化 100K Gaussians → 用 Marigold 估计深度 → GS 优化 (SfM-style)
```

**Loss 曲线**:

| 指标 | 初始 → 最终 | 降幅 |
|------|-------------|------|
| Total Loss | 0.0537 → 0.0102 | ↓81% |
| Event Loss | 0.0201 → 0.0011 | ↓95% |
| SSIM Loss | 0.0335 → 0.0092 | ↓73% |
| Gaussian 数 | 100,000 → 184,251 | — |

### 阶段2: SfM 初始化 (深度图引导)

用阶段1估计的深度图做 SfM 初始化，精化 Gaussian 位置。

### 阶段3: 增量 Tracking + Bundle Adjustment

```
滑动窗口 Tracking → 局部 BA → 全局 BA → 重复
```

- 窗口大小: 20 frames
- Tracking 每 2 次后做 1 次 BA
- 最终全局 BA

## 8. 输出结构

```
output/final/office0/demo/
├── initialization/          # 初始化可视化
│   ├── iter_0_vis.jpg      # 迭代过程渲染
│   ├── iter_500_vis.jpg
│   ├── iter_1000_vis.jpg
│   ├── iter_1500_vis.jpg
│   ├── ...
│   └── pose_file_*.txt     # 估计/GT 姿态
├── tracking/                # Tracking 结果
├── BA/                      # Bundle Adjustment
├── global_BA/               # 全局 BA (最终结果)
├── img_eval/                # 评估图像
│   ├── gt/                  # Ground Truth
│   └── est/                 # 估计渲染
└── depth_data.npy           # 估计的深度图
```

## 9. 评估

### NVS (Novel View Synthesis)
```bash
cd metrics/event_nerf
bash auto_single.sh [PATH_TO_IMG_EVAL] est gt
```

### 姿态精度
```bash
pip install evo
cd output/final/office0/demo/global_BA
evo_ape kitti est_poses.txt gt_poses.txt --align -as
```

## 10. 已知问题

| 问题 | 状态 | 解决方案 |
|------|------|----------|
| SharePoint 下载慢 (~90KB/s, 需6天) | ⚠️ | 本地下载后 SCP |
| Tar 截断 (传输不完整) | ⚠️ | 重新 SCP 或接受部分场景 |
| CUDA JIT 编译慢 (首次 ~2min) | ✅ 正常 | 设置 TORCH_CUDA_ARCH_LIST |
| 磁盘空间 (需 ~100GB) | ⚠️ | 解压后删 tar |
| Python stdout 缓冲 (log为空) | ✅ 已解决 | 用 python -u |
| 深度模型兼容性 | ✅ | HF LCM 版本可用 |

### 成功运行的场景

| 场景 | 图片 | 事件数据 | 状态 |
|------|------|----------|------|
| office0 | 999 | 7.3 GB | ✅ 完整运行 |
| room2 | 998 | — | 可运行 |
| mocap-desk | 752 | 2.4 GB | 可运行 |
| mocap-desk2 | 429 | — | 可运行 |
| mocap-1d-trans | 732 | — | 可运行 |
| mocap-6dof | 390 | — | 可运行 |
