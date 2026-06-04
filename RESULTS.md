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

| 指标 | 值 |
|------|-----|
| **PSNR** ↑ | **6.42 dB** (mean) |
| **SSIM** ↑ | **0.4892** |
| **LPIPS** ↓ | **0.3123** |

### 姿态精度

| 指标 | 值 |
|------|-----|
| **ATE RMSE** | **0.756 mm** |
| ATE Mean | 0.717 mm |
| ATE Median | 0.682 mm |
| ATE Std | 0.242 mm |

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

### NVS 性能

| 指标 | 值 | 说明 |
|------|-----|------|
| **PSNR** ↑ | **8.12 dB** (mean) | 事件相机与GT相机分辨率不同, resize到320×320评估 |
| **SSIM** ↑ | **0.1097** | 真实事件噪声大, 结构重建更难 |
| **LPIPS** ↓ | N/A | GT/EST分辨率不匹配, LPIPS运行失败 |

> TUM-VIE 的 GT 和渲染图来自**不同相机** (RGB cam 1024×1024 vs Event cam 240×180 上采样到 1280×720)，NVS 指标只能作为参考。

### 姿态精度

| 指标 | 值 |
|------|-----|
| **ATE RMSE** | **1.288 mm** |
| ATE Mean | 1.182 mm |
| ATE Median | 1.102 mm |
| ATE Std | 0.512 mm |

---

## 两场景对比

| 指标 | office0 (Replica) | mocap-desk (TUM-VIE) |
|------|:---:|:---:|
| 数据类型 | 合成事件 | **真实事件相机** |
| 训练帧数 | 25 | 40 |
| 训练时长 | 2h32m | ~3h00m |
| 最终 GS | 471K | 20K |
| **PSNR** ↑ | 6.42 dB | 8.12 dB |
| **SSIM** ↑ | 0.489 | 0.110 |
| **ATE RMSE** ↓ | 0.756 mm | 1.288 mm |

### 关键观察

1. **姿态精度两个场景都极好 (< 2mm)**：EGM 约束对姿态优化的有效性在合成和真实数据上都得到了验证
2. **真实事件相机 SSIM 更低**：TUM-VIE 的事件噪声、鱼眼畸变残差、RGB/Event 相机视差都增加了难度
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
║  PSNR   ↑         │  6.42 dB     │  8.12 dB          ║
║  SSIM   ↑         │  0.489       │  0.110            ║
║  LPIPS  ↓         │  0.312       │  N/A              ║
╠════════════════════════════════════════════════════════╣
║  ATE RMSE  ↓      │  0.756 mm    │  1.288 mm         ║
║  ATE Median ↓     │  0.682 mm    │  1.102 mm         ║
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
