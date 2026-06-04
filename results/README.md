# IncEventGS 复现结果

## 目录结构

每个场景 (office0 / mocap-desk):

```
<scene>/
├── initialization_vis/     # 初始化可视化 (每100步)
│   ├── iter_0_vis.jpg     # 渲染图 + 深度图
│   ├── iter_500_vis.jpg
│   ├── iter_1000_vis.jpg
│   ├── ...
│   └── iter_4500_vis.jpg
├── BA_vis/                 # BA 诊断面板 (关键帧)
│   ├── BA_fXXX_0000_img.jpg  # 5列事件对比: start_img | end_img | gt_events | syn_events | diff
│   └── BA_fXXX_pose.png      # 估计 vs GT 姿态轨迹
├── pose_viz/               # 初始化/Tracking/BA 姿态估计 vs GT
│   ├── init_fXXX_pose.png
│   └── tracking_fXXX_pose.png
├── global_BA/              # 全局 BA 最终结果
│   ├── est_kitti.txt       # 估计轨迹 (KITTI format)
│   ├── gt_kitti.txt        # GT 轨迹
│   └── BA_fXXX_pose.png    # 全局姿态轨迹
├── eval/                   # NVS 评估 (仅 office0)
│   ├── psnr_est.txt
│   ├── ssim_est.txt
│   └── lpips_est.txt
├── training_loss.png       # Loss 曲线
└── training_*.log          # 完整训练日志
