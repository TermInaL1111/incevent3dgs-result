# IncEventGS 复现结果

> 数据集: [SharePoint](https://westlakeu-my.sharepoint.com/personal/cvgl_westlake_edu_cn/_layouts/15/onedrive.aspx?ga=1&id=%2Fpersonal%2Fcvgl%5Fwestlake%5Fedu%5Fcn%2FDocuments%2FIncEventGS)
> 模型: IncEventGS (CVPR 2025) — Pose-Free Event-based 3D Gaussian Splatting
> GPU: NVIDIA RTX 4090 24GB

---

## 场景一: Replica 数据集 — office0 (合成事件)

### 实验配置

| 项目 | 值 |
|------|-----|
| 数据集 | **Replica (合成事件)** |
| 场景 | **office0** |
| 训练帧 | frame 10→60 (25帧, interval=2) |
| 评估帧 | frame 21→119 (49帧, interval=2) |
| 图片分辨率 | 768×480 |
| 数据 | 999 images + 7.3GB events (.npy) + traj.txt + poses_ts.txt |
| 训练时长 | 2h32min |
| 深度模型 | prs-eth/marigold-lcm-v1-0 (HuggingFace, 14GB) |

### NVS 性能

| 指标 | 值 | 论文 |
|------|-----|------|
| **PSNR** ↑ | **25.02 dB** | 25.64 dB |
| **SSIM** ↑ | **0.489** (raw) / 0.822 (corr) | 0.54 |
| **LPIPS** ↓ | **0.312** (raw) / 0.130 (corr) | 0.30 |

> comppsnr.py 对渲染图做 tone-mapping 校正后再算 PSNR, 校正后 25.02 dB 与论文 25.64 dB 仅差 0.6 dB。

### 姿态精度

| 指标 | 值 |
|------|-----|
| **ATE RMSE** | **0.076 cm** (论文: 0.045 cm) |
| ATE Mean | 0.072 cm |
| ATE Median | 0.068 cm |
| ATE Std | 0.024 cm |

---

## 场景二: TUM-VIE 数据集 — mocap-desk (真实事件相机)

### 实验配置

| 项目 | 值 |
|------|-----|
| 数据集 | **TUM-VIE (真实事件相机)** |
| 场景 | **mocap-desk** |
| 训练帧 | frame 90→130 (40帧, interval=1) |
| 评估帧 | frame 100→138 (38帧) |
| 相机 | 鱼眼校正 (event cam: 240×180, RGB cam: 1024×1024) |
| 数据 | 752 images + 2.4GB events (HDF5/EventSlicer) + mocap_data.txt |
| 训练时长 | ~3h00min |
| 深度模型 | prs-eth/marigold-lcm-v1-0 (HuggingFace, 14GB) |

### 定性结果

> 论文原文: "不存在与事件相机对齐的真实图像 (GT), 因此我们从 RGB 相机中选取最近的图像并裁剪至与渲染图像相同大小, 以进行视觉对比。"
>
> TUM-VIE 真实事件数据集**没有对齐的 GT 图像**。event cam (240×180) 和 RGB cam (1024×1024) 是**不同的物理传感器**, 存在视差、分辨率和光照响应差异。论文只做定性视觉比较, **不报告 PSNR/SSIM/LPIPS**。
>
> 渲染图像见 `results/mocap-desk/initialization_vis/` 和 `results/mocap-desk/BA_vis/`。

### 姿态精度

| 指标 | 值 |
|------|-----|
| **ATE RMSE** | **0.129 cm** (论文: 0.231 cm) |
| ATE Mean | 0.118 cm |
| ATE Median | 0.110 cm |
| ATE Std | 0.051 cm |

---

## 两场景对比

| 指标 | office0 (Replica) | mocap-desk (TUM-VIE) |
|------|:---:|:---:|
| 数据类型 | 合成事件 | **真实事件相机** |
| 训练帧数 | 25 | 40 |
| 训练时长 | 2h32m | ~3h00m |
| 最终 GS | 471K | 20K |
| **PSNR** ↑ | 25.02 dB | N/A (定性) |
| **SSIM** ↑ | 0.489 | N/A (定性) |
| **ATE RMSE** ↓ | 0.076 cm (论文 0.045) | 0.129 cm (论文 0.231) |

### 关键观察

1. **姿态精度两个场景都极好 (< 2mm)**：EGM 约束对姿态优化的有效性在合成和真实数据上都得到了验证
2. **真实数据集只做定性评估**: 论文不报告真实数据的 NVS 指标, 因 event cam 与 RGB cam GT 不对齐
3. **Gaussian 数量差异大**：office0 密度高 (471K)，mocap-desk 经过 prune 后只有 20K，说明真实数据上模型更保守
4. **NVS 并非本方法的主要优势**：事件相机天然不感知颜色，渲染灰度图与 RGB GT 比较有系统偏差

---

## 完整指标汇总

```
╔════════════════════════════════════════════════════════╗
║           IncEventGS 复现结果 (RTX 4090)              ║
╠════════════════════════════════════════════════════════╣
║                    │  office0     │  mocap-desk       ║
║                    │  (合成事件)  │  (真实事件相机)    ║
╠════════════════════════════════════════════════════════╣
║  训练时长          │  2h32m       │  ~3h00m           ║
║  最终 GS           │  471K        │  20K              ║
╠════════════════════════════════════════════════════════╣
║  PSNR   ↑         │  25.02 dB    │  N/A (定性)       ║
║  SSIM   ↑         │  0.489       │  N/A              ║
║  LPIPS  ↓         │  0.312       │  N/A              ║
╠════════════════════════════════════════════════════════╣
║  ATE RMSE  ↓      │  0.076 cm    │  0.129 cm         ║
║  (论文)            │  (0.045 cm)  │  (0.231 cm)       ║
║  ATE Median ↓     │  0.068 cm    │  0.110 cm         ║
╚════════════════════════════════════════════════════════╝
```

## 输出文件

| 文件 | 内容 |
|------|------|
| `/data/training_loss.png` | office0 Loss 曲线 |
| `/data/training_loss_tumvie.png` | mocap-desk Loss 曲线 |
| `/data/training_replica_office0.log` | office0 完整训练日志 |
| `/data/training_tumvie_mocap-desk.log` | mocap-desk 完整训练日志 |
| `output/final/office0/demo/` | office0 全部输出 (2053 files) |
| `output/final/tumvie/mocap-desk/demo/` | mocap-desk 全部输出 (2430 files) |
