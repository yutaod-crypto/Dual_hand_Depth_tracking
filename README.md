# Dual_hand_Depth_tracking
┌─────────────────────────────────────────────────────────────────┐
  │  输入：双目立体录制 (STEREO RECORDING)                          │
  │   left_raw.mp4 · right_raw.mp4 · camera_intrinsics.json         │
  │   camera_traj_left.npz  (SLAM, 预计算)                          │
  └───────────────────────────┬─────────────────────────────────────┘
                              │
              ┌───────────────┴──────────────┐
              │                              │
              ▼                              ▼
  ┌───────────────────────┐    ┌─────────────────────────────┐
  │  阶段 A               │    │  阶段 B                     │
  │  图像去畸变 +         │    │  相机轨迹                   │
  │  内参 JSON            │    │  复制 / 后备方案            │
  └───────────┬───────────┘    └──────────────┬──────────────┘
              │                               │
              │  ego_raw_video/               │  camera_traj.npz
              │  ego_undistorted_video/       │  (N×4×4 cam_c2w,
              │  *_video_info.json            │   intrinsic, depths)
              │                               │
              └───────────────┬───────────────┘
                              │  (均被阶段 C 使用)
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  阶段 C — 手部处理流水线 (Hand Pipeline)                        │
  │                                                                 │
  │  C.1  左眼单目 (Mono LEFT) ─────────────────────────┐           │
  │       left_undistorted.mp4 + MoGe 深度预测          │           │
  │       → scratch/mono_left/hands.npz                 │           │
  │                                                     │           │
  │  C.2  右眼单目 (Mono RIGHT) ────────────────────────┤           │
  │       right_undistorted.mp4 + MoGe 深度预测         │           │
  │       → scratch/mono_right/hands.npz                │           │
  │                                                     ▼           │
  │  C.3  双目立体三角化 ───────────→  scratch/                     │
  │       双视角 YOLO + MediaPipe      triangulation/               │
  │       DLT → 真实尺度 3D 关节       skeleton.npz                 │
  │                                                     │           │
  │                                                     ▼           │
  │  C.4  混合融合 (Hybrid merge) ─────────→  scratch/              │
  │       三角化（主干）                      hybrid_v2/            │
  │       + 偏差校正的单目后备方案            skeleton.npz          │
  │       + 间隙插值                                                │
  │                                                     │           │
  │                                                     ▼           │
  │  C.5  One-Euro 平滑滤波 ──────────────→  hands.npz              │
  │       逐关节、逐轴的自适应滤波           skeleton.npz           │
  │       + Y轴向上 (Y-up) 规范转换          (交付物根目录)         │
  └─────────────────────────────┬───────────────────────────────────┘
                                │  hands.npz  skeleton.npz
                                │  camera_traj.npz  (来自阶段 B)
                                │  left_raw.mp4     (来自阶段 A)
                                ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  阶段 D — 可视化 (Visualization)                                │
  │   visualization.mp4    (原始左眼视频叠加 2D 骨骼)               │
  │   3d_view_multi.mp4    (matplotlib 4面板 3D 动画)               │
  │   3d_view.html         (plotly 交互式, 4面板, 带滑动条)         │
  └─────────────────────────────┬───────────────────────────────────┘
                                │
                                ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  阶段 E — 数据标注 (Annotations)                                │
  │   annotations.json     (标注模板，由人工标注员填写)             │
  └─────────────────────────────────────────────────────────────────┘
