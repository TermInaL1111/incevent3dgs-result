# IncEventGS 复现结果

- **场景一 — office0 (Replica, 合成事件)**: `office0/`
- **场景二 — mocap-desk (TUM-VIE, 真实事件相机)**: `mocap-desk/`

每个场景包含:
- `initialization_vis/` — 初始化过程可视化 (every 500 steps)
- `eval/` — NVS 评估 (PSNR/SSIM/LPIPS)
- `global_BA/` — 全局 BA 姿态估计 (KITTI format, 可直接 evo 评估)
- `training_loss.png` — Loss 曲线
- `training_*.log` — 完整训练日志

详见项目根目录 [RESULTS.md](../RESULTS.md)
