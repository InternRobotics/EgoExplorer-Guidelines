# 11. 坐标系约定

[返回目录](../index.md) · [English](../en/11_coordinate_systems.md)

这是整个项目统一的坐标约定，代码实现在 `src/five_tracker_smplx/coordinates.py`。读写数据之前请完整读一遍本章，第 11.9 节列出了最容易犯的错误。

## 11.1 世界坐标系

| 项目 | 约定 |
|---|---|
| 来源 | VIVE 定位系统的世界坐标，原样记录，不做平移、旋转或换轴。`unified_30hz` 中记为 `raw_capture_y_up`，LeRobot 中记为 `sdk-world-as-recorded` |
| 手性与单位 | 右手系，米 |
| 竖直方向 | **+Y 向上**，重力方向为 `(0, −1, 0)` |
| 原点 | VIVE 场地地图里设定的参考点，不在地面上，也不在人身上 |
| 地面 | 高度为 `ground_y_m`（原始坐标中的 Y 值），默认由脚踝 tracker 的低分位估计 |
| 水平朝向 | 标定时用地面胶带让起始站姿满足 **+X = 佩戴者的左方，+Z = 佩戴者的前方**。任务录制可以从任意朝向开始，人体的左和前由解出的根节点旋转决定，不能直接把世界坐标的 X、Z 当成人体当前的左和前 |

## 11.2 tracker 位姿

**位置**：tracker 原点在世界坐标系中的位置，单位米。

**四元数顺序**：统一为 **w, x, y, z**。

**旋转的含义**：四元数转成矩阵 `R` 后，是从 tracker 局部坐标到世界坐标的旋转（作用于列向量），`R` 的**各列**是 tracker 自身三个坐标轴在世界中的方向。

**安装关系**（标定文件格式 v5，`column: tracker_world = joint_world @ tracker_in_joint`）：

```text
R_tracker_world = R_joint_world · R_tracker_in_joint
p_tracker_world = p_joint_world + R_joint_world · a_tracker_in_joint      （a 以米为单位，不再乘 scale）
```

导出中的 `attachment_position`、`attachment_rotation` 就是 `a` 和 `R_tracker_in_joint`。

其它常用关系：

- 相对运动：`ΔR = R_0ᵀ · R_t`；
- 换一个世界坐标基 `Q`：位置变为 `Q · p`，旋转变为 `Q · R`；
- 6D 旋转（LeRobot 中的 `rotation_6d`）：取矩阵前两列，顺序为 `r00, r10, r20, r01, r11, r21`。

## 11.3 角色、关节与左右

| tracker 角色 | 对应的 SMPL-X 关节 |
|---|---|
| `camera` | 15，head |
| `left_wrist`、`right_wrist` | 20、21 |
| `left_ankle`、`right_ankle` | 7、8 |

`left_*` 永远是佩戴者自己的左侧。正视图不做镜像：面对佩戴者时，他的左侧出现在画面右边。任何代码和可视化都不能按画面左右调换角色。

!!! warning "角色的排列顺序在不同文件里不同"
    原始 npz 和 LeRobot 按字母排列：`camera, left_ankle, left_wrist, right_ankle, right_wrist`；`unified_30hz` 和标定文件是 `camera, left_wrist, right_wrist, left_ankle, right_ankle`。**按名字取，不要按位置取。**

## 11.4 SMPL-X

- 模型：中性 SMPL-X，`num_betas = 10`，`use_pca = False`（手部为 45 维轴角）。
- 根节点没有旋转时，人体坐标轴为 **+X 指向人体左侧、+Y 向上、+Z 指向人体前方**。
- `global_orient` 是根节点相对世界的旋转，`body_pose` 是 21 个关节的局部轴角（弧度），`left_hand_pose`、`right_hand_pose` 是 MANO 15 个关节的局部轴角。

**`unified_30hz` 和 `smplx_episode.npz` 的平移约定**：

```text
V_world = scale · SMPLX(betas, global_orient, body_pose, hands, transl = 0).vertices + transl
```

tracker 的位置和 `V_world` 在同一个世界坐标系里、单位相同，tracker 不乘 `scale`。

**世界坐标下的骨盆位置** `u_t`（论文和完整发布版的平移字段）。SMPL-X 的根节点绕骨盆关节旋转，所以骨盆位置与姿态无关：

```text
u_t = transl_t + scale · J_pelvis(betas)          （J_pelvis：零姿态、零平移时的骨盆关节位置）
```

## 11.5 手套与手

- `unified_30hz` 里的 `glove_left_points`、`glove_right_points` 是**手腕局部坐标系**下的关节点。MANUS 用的是它自己的手腕局部化公式 `R_wristᵀ (p − p_wrist)`，与 tracker 的安装关系是两套约定，不要混用。
- Wuji 的骨架本来就是手腕局部坐标，根节点四元数始终为单位四元数，不含手的整体朝向。手在世界中的朝向完全来自手腕 tracker 和标定好的手腕安装参数。
- SMPL-X 的手就是 MANO，关节一一对应：左手为关节 25–39，右手为 40–54，顺序为食指、中指、小指、无名指、拇指。
- **左手是右手的镜像。** 凡是用叉积得到的方向（弯曲轴、掌心法向），左右手的符号相反，要分别从静止姿态推导。取 `fwd` 为手腕指向掌指关节中心、`lat` 为小指指向食指时，掌心方向在左手为 `+cross(fwd, lat)`，在右手为 `−cross(fwd, lat)`。

## 11.6 相机

- GoPro 与头部 tracker 刚性相连，相机位姿等于头部 tracker 位姿乘以一个固定的相机–tracker 外参。这个外参**目前还没有单独标定**，发布数据里记为 `not_available`。
- 发布视频由原始 1920×1080 裁掉顶部 110 像素得到，为 1920×970，相机内参为 `fx = 1012.0, fy = 1016.0, cx = 960.0, cy = 430.0`（像素）。

## 11.7 显示时转为 Z 轴向上

可视化需要 Z 向上、地面在 0 时：

```text
[x', y', z'] = [x, −z, y − ground_y_m]       对应旋转矩阵 [[1,0,0],[0,0,−1],[0,1,0]]，行列式为 +1
```

只交换 Y 和 Z、不加负号的变换是镜像（行列式为 −1），会把人体左右翻过来。

## 11.8 时间

| 数据 | 时间约定 |
|---|---|
| 原始文件 | 微秒，主机系统时钟 `t_host_us` |
| `smplx_profile_driven.npz` 的 `timestamps` | 从第一帧开始计时（从 0 开始）；绝对时间 = 相对时间 + `tracker_t_host[0]` |
| `unified_30hz` | 主机绝对时间（秒），栅格为 1/30 秒的整数倍 |
| LeRobot | episode 内的秒数，等于 mp4 的显示时间；`metadata.json` 中的 `start_timestamp_host_us` 可换算回绝对时间 |

## 11.9 常见错误

1. **把四元数当成另一种顺序读取。** 顺序读错后，相对旋转的方向会反过来，例如"向左看"被读成向右转。按 w, x, y, z 读取，相对运动用 `ΔR = R_0ᵀ · R_t`。
2. **用头部 tracker 的绝对朝向当人体朝向。** 头部 tracker 的朝向里含有头带的安装旋转，直接用会让初始朝向偏差很大，优化器只能让左右肢体交叉去够 tracker。初始朝向要先扣除安装旋转，或者用五个 tracker 的相对位置估计。
3. **显示时只交换 Y 和 Z。** 这是镜像，会把人体左右翻过来，要按第 11.7 节的旋转来做。
