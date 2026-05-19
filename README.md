graph TD
    Input["<b>输入：双目视频录制 (INPUT: STEREO RECORDING)</b><br>left_raw.mp4 · right_raw.mp4 · camera_intrinsics.json<br>camera_traj_left.npz (SLAM, 预计算)"]

    StageA["<b>阶段 A</b><br>图像去畸变 + 内参 JSON"]
    StageB["<b>阶段 B</b><br>相机轨迹复制 / 后备方案"]

    Input --> StageA
    Input --> StageB

    DataA["ego_raw_video/<br>ego_undistorted_video/<br>*_video_info.json"]
    DataB["camera_traj.npz<br>(N×4×4 cam_c2w, intrinsic, depths)"]

    StageA --> DataA
    StageB --> DataB

    subgraph StageC [阶段 C — 手部流水线 Hand Pipeline]
        direction TB
        C1["<b>C.1 左眼单目</b><br>left_undistorted.mp4 + MoGe 深度预测<br>→ scratch/mono_left/hands.npz"]
        C2["<b>C.2 右眼单目</b><br>right_undistorted.mp4 + MoGe 深度预测<br>→ scratch/mono_right/hands.npz"]

        C3["<b>C.3 双目立体三角化</b><br>双视角 YOLO + MediaPipe<br>DLT → 真实尺度 3D 关节"]
        C3Out["scratch/triangulation/skeleton.npz"]

        C4["<b>C.4 混合融合 (Hybrid Merge)</b><br>三角化（主干）<br>+ 偏差校正的单目后备方案<br>+ 间隙插值"]
        C4Out["scratch/hybrid_v2/skeleton.npz"]

        C5["<b>C.5 One-Euro 平滑滤波</b><br>逐关节逐轴自适应滤波<br>+ Y轴向上规范转换 (Y-up)"]
        C5Out["hands.npz<br>skeleton.npz<br>(交付物根目录)"]

        C1 --> C3
        C2 --> C3
        C3 --> C3Out
        C3Out --> C4
        C4 --> C4Out
        C4Out --> C5
        C5 --> C5Out
    end

    DataA --> C1
    DataA --> C2
    DataB --> StageC

    DataFinal["hands.npz, skeleton.npz (来自阶段 C)<br>camera_traj.npz (来自阶段 B)<br>left_raw.mp4 (来自阶段 A)"]
    StageC --> DataFinal

    StageD["<b>阶段 D — 可视化 (Visualization)</b><br>visualization.mp4 (原始左眼视频叠加 2D 骨骼)<br>3d_view_multi.mp4 (matplotlib 4面板 3D 动画)<br>3d_view.html (plotly 交互式, 4面板, 带滑动条)"]
    DataFinal --> StageD

    StageE["<b>阶段 E — 数据标注 (Annotations)</b><br>annotations.json (标注模板，由人工标注员填写)"]
    StageD --> StageE
