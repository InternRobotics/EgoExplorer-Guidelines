# 11. 坐标系约定

[返回目录](../../README.md) · [English](../en/11_coordinate_systems.md)

这是全项目唯一的坐标约定，代码实现见 `src/five_tracker_smplx/coordinates.py`。早期曾在三处犯过系统性错误（见 11.9），读写数据前请完整读一遍。

## 11.1 世界坐标系

| 项 | 约定 |
|---|---|
| 来源 | VIVE / SDK 世界坐标系原样记录，**不做原点、yaw 或轴变换**（`coordinate_frame = raw_capture_y_up`；LeRobot 中为 `sdk-world-as-recorded`） |
| 手性 / 单位 | 右手系，米 |
| 竖直方向 | **+Y 向上**，重力方向 `(0, −1, 0)` |
| 原点 | VIVE 场地地图里指定的参考位置（不是地面，也不是受试者） |
| 地面 | 地面高度为 `ground_y_m`（原始坐标里的 Y 值，默认由踝 tracker 低分位估计） |
| 水平朝向 | 标定时用地面胶带标记让起始站姿的 **+X = 受试者左、+Z = 受试者前**。普通任务录制可以从任意 yaw 开始，人体的左右/前后由拟合出的根旋转决定，**不能**把固定世界的 X/Z 当成当帧人体的左/前 |

## 11.2 Tracker 位姿

- **位置**：tracker 原点在世界系中的坐标（米）。
- **四元数顺序**：

  | 文件 | 顺序 |
  |---|---|
  | 原始 `*.tracker.npz` | **w, x, y, z** |
  | 手套 `hand_skeleton`、subject profile、LeRobot 发布 | **x, y, z, w** |

- **旋转含义**：四元数转成矩阵 `R` 后是标准的**主动、局部 → 世界**旋转（列向量），`R` 的**列**是 tracker 自身三个轴在世界中的方向。
- **固定安装**（profile v5，`column: tracker_world = joint_world @ tracker_in_joint`）：
  ```text
  R_tracker_world = R_joint_world · R_tracker_in_joint
  p_tracker_world = p_joint_world + R_joint_world · a_tracker_in_joint      （a 为米制，不再乘 scale）
  ```
  `attachment_position` / `attachment_rotation` 就是 `a` 和 `R_tracker_in_joint`。
- **相对运动**：`ΔR = R_0ᵀ · R_t`。
- **换世界基** `Q`：位置 `Q · p`，旋转 `Q · R`。
- **6D 旋转**（LeRobot `rotation_6d`）：矩阵前两列，顺序 `r00, r10, r20, r01, r11, r21`。

## 11.3 角色、锚点关节与左右

| tracker 角色 | 锚点 SMPL-X 关节 |
|---|---|
| `camera` | 15 head |
| `left_wrist` / `right_wrist` | 20 / 21 |
| `left_ankle` / `right_ankle` | 7 / 8 |

`left_*` 永远是**受试者自己的**左侧，不是画面左侧。正视图不镜像：面对受试者时，他的左侧出现在画面右边。任何代码和 viewer 都不得按画面交换角色来"修正"显示。

注意 `tracker_roles` 的顺序：原始 npz 和 LeRobot 是 `camera, left_ankle, left_wrist, right_ankle, right_wrist`（字母序），unified_30hz 和 profile 是 `camera, left_wrist, right_wrist, left_ankle, right_ankle`。**按名字取，不要按位置取。**

## 11.4 SMPL-X

- 模型：中性 SMPL-X，`num_betas = 10`，`use_pca = False`（手部 45 维轴角）。
- 未施加根旋转时的人体轴：**+X = 人体左，+Y = 上，+Z = 人体前**。
- `global_orient` 是相对世界的根旋转，`body_pose` 是 21 个关节的局部轴角（弧度），`left/right_hand_pose` 是 MANO 15 关节的局部轴角。
- **unified_30hz（以及 mini release 的 `smplx_episode.npz`）的平移约定**：
  ```text
  V_world = scale · SMPLX(betas, global_orient, body_pose, hands, transl = 0).vertices + transl
  ```
  tracker 位置与 `V_world` 在同一世界系、同一单位，tracker 不乘 `scale`。
- **世界骨盆位置** `u_t`（论文 / 完整发布用的平移字段）：SMPL-X 的根旋转绕骨盆关节进行，所以骨盆位置与姿态无关：
  ```text
  u_t = transl_t + scale · J_pelvis(betas)          （J_pelvis 为零姿态、零平移时的骨盆关节位置）
  ```

## 11.5 手套与手

- unified_30hz 的 `glove_{left,right}_points` 是**腕局部坐标系**下的关节点。MANUS 用的是它自己的腕局部化公式 `R_wristᵀ (p − p_wrist)`；这与 tracker 的安装外参约定是两回事，不能混用。
- Wuji 骨架本身就是腕局部的，根节点四元数恒为单位阵，不带手部全局朝向。手的世界朝向完全来自腕部 tracker 和标定好的腕部变换。
- SMPL-X 的手就是 MANO，逐关节对应（左手关节 25–39，右手 40–54，顺序食指/中指/小指/无名指/拇指）。**左手是右手的镜像**，任何由叉积得到的方向（屈曲轴、掌心法向）在两只手之间都会变号，必须按手分别从静息姿态推导。按 `fwd = 腕 → 掌指关节中心`、`lat = 小指 → 食指` 定义时，掌心方向左手为 `+cross(fwd, lat)`、右手为 `−cross(fwd, lat)`。

## 11.6 相机

- GoPro 通过转接件与头部 tracker 刚性相连，所以相机位姿 = 头部 tracker 位姿 · 固定的相机–tracker 外参。这个外参**目前还没有单独标定**（发布里记为 `not_available`）。
- 发布视频：原始 1920×1080 顶部裁掉 110 px，得到 1920×970；内参 `fx = 1012.0, fy = 1016.0, cx = 960.0, cy = 430.0`（像素）。

## 11.7 显示用的 Z-up 转换

可视化需要 Z 向上、地面在 0 时：

```text
[x_view, y_view, z_view] = [x, −z, y − ground_y_m]       旋转矩阵 [[1,0,0],[0,0,−1],[0,1,0]]，det = +1
```

**只交换 Y/Z、不加负号是反射（det = −1），会把人体镜像**——这正是早期 viewer 出过的错。

## 11.8 时间

- 原始文件：微秒，主机墙钟 `t_host_us`。
- `smplx_profile_driven.npz` 的 `timestamps`：**相对第一帧**（从 0 开始），`绝对 = 相对 + tracker_t_host[0]`。
- unified_30hz：绝对主机墙钟（秒），栅格为 1/30 秒的整数倍。
- LeRobot：episode 内相对秒数，等于 mp4 pts；`metadata.json` 里有 `start_timestamp_host_us` 可换回绝对时间。

## 11.9 历史错误（避免重犯）

1. 曾把记录器四元数当成右作用矩阵，相对姿态写成 `R_t · R_0ᵀ`，导致"向左看"读成负方向。根因是源数据其实是 **wxyz** 而不是 xyzw；2026-09-02 修正后 profile 升级为 v5（列约定），**v4 及更早的 profile 会被拒绝，必须重新标定**。项目 README 和 `calibration_protocol.md` 里描述的"右作用 `R_tracker = A_pre @ R_joint_world`"是 v4 的旧写法，以本章为准。
2. 曾用头部 tracker 的**绝对安装朝向**初始化骨盆 yaw，没消掉头环安装旋转，初始朝向错了约 155°，优化器只好交叉左右肢体去追端点。现在用带标签的端点做水平 Procrustes 估计初始 yaw。
3. 曾在 viewer 里把 `(X, Y, Z)` 直接排成 `(X, Z, Y)`（反射），把人体镜像。
