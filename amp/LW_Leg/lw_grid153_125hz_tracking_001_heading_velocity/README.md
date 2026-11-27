# LW 双足 AMP 专家库 · 航向系基座速度版

`lw_grid153_125hz_tracking_001_heading_velocity` 含 152 条完整的起步、巡航、制动、停稳轨迹和 1 条静站轨迹，共 109,353 帧。数据来自上一版通过验收的 200 Hz OCS2 求解记录，每 0.02 s 直接抽取一帧。步态周期为 0.80 s（1.25 Hz）。[上一版 22 维专家库](../lw_grid153_125hz_tracking_001/)

## 专家帧

训练文件位于 `motion_amp_expert/motion_*.txt`。每个文件是 JSON，`Frames` 中每行保存一个完整采样时刻的 28 个数。以下使用 Python 切片下标。

| 切片 | 数据 | 坐标系、顺序及单位 |
| --- | --- | --- |
| `[0:3]` | 基座线速度 | 航向系 `vx, vy, vz`，m/s，取 `base_link` 连杆原点 |
| `[3:6]` | 基座角速度 | 航向系 `wx, wy, wz`，rad/s，机身相对世界的物理角速度 |
| `[6:14]` | 8 个关节角相对标称零位的偏移 | rad，髋、股、胫、踝，每种均为先右后左 |
| `[14:22]` | 8 个关节速度 | rad/s，与关节角同序 |
| `[22:25]` | 左足连杆原点相对基座的位置 | 完整机身系 xyz，m |
| `[25:28]` | 右足连杆原点相对基座的位置 | 完整机身系 xyz，m |

航向系 x 轴沿机身前向在水平面上的投影，y 轴水平向左，z 轴与世界系竖直向上对齐。设 ψ 为该帧实际航向角，两个向量均使用 `v_H = Rz(-ψ) v_W`、`ω_H = Rz(-ψ) ω_W`。例如，机身朝世界 +y 运动且航向为 +90° 时，该运动记为航向系正向线速度。机身有俯仰和横滚时，`wz` 表示物理角速度在世界竖直轴上的投影。两足的位置使用包含机身横滚和俯仰的坐标系。

每个文件通过 `RootVelocityFrame = "heading"`、`RootVelocityPoint = "base_link"` 和 `FrameLayout` 标识布局。文件头的 `Command = [vx, vy, wz]` 是目标速度，帧中的速度取自该时刻实际求解状态，包括起停和左右换重。各速度点、帧数、动作阶段、`MotionWeight` 及后 22 维数值与上一版逐文件对应。

## 训练端接入

在 `robot_lab` 根目录运行以下命令可拉取和校验专家库，假定 `LW_datasets` 与训练仓库同级。

```bash
git -C ../LW_datasets pull --ff-only origin master
amp_release="$(realpath ../LW_datasets/amp/LW_Leg/lw_grid153_125hz_tracking_001_heading_velocity)"
(cd "$amp_release" && sha256sum --quiet --check SHA256SUMS)
```

训练时将 `agent.amp_motion_files` 指向 `$amp_release/motion_amp_expert/motion_*.txt`。使用新版时，专家加载器和在线 AMP 观测都要采用上表的 28 维布局。在线侧要取根连杆原点的世界线速度和机身的世界角速度，按每一帧自己的航向转换，再按时间从旧到新拼接。10 帧对应 280 维、首尾间隔 0.18 s。旧库对应的加载器采用 22 维输入。

## 覆盖范围和验收

`vx` 从 −0.8 到 +0.8 m/s，每 0.1 m/s 一点。`vy = 0`。`wz` 取 0、±0.10、±0.15、±0.25、±0.35 rad/s。每个非零速度点包含完整动作，采用 `Clamp`。静站轨迹为 41 帧、0.8 s，采用 `Wrap`。[指令和权重](amp_walk_125hz.json)

153 条轨迹均重新通过参考轨迹验收。逐帧用原始广义坐标及广义速度经 Pinocchio 独立计算 `base_link` 速度，与新专家速度旋回世界系的结果比较。线速度和角速度的最大分量误差分别约为 6.66 × 10⁻¹⁶ m/s 和 1.33 × 10⁻¹⁵ rad/s。原有 22 维与上一版逐帧按双精度位模式核对。全库另通过 MuJoCo 正运动学核验。[本版核验结果](release_verification.json) · [逐速度点的巡航跟踪](tracking_by_command.csv) · [步宽、抬脚和机身高度](height_verification.json)
