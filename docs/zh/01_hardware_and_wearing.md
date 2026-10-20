# 1. 硬件组成与穿戴

[返回目录](../index.md) · [English](../en/01_hardware_and_wearing.md)

![五个 tracker、头部相机与两种手套的穿戴方式](../assets/img/rig-fig.jpg)

*A：五个 tracker 的位置（头、双手手背、双踝外侧）；B：头部 GoPro 与 tracker 刚性相连；C：踝部 tracker；D：MANUS 手套（上）与 Wuji 手套（下）。*

## 1.1 采购清单

> **待补充**：手套的购买链接由硬件负责同事提供后补全。

| 部件 | 型号 / 规格 | 数量 | 购买链接 | 备注 |
|---|---|---|---|---|
| 身体 tracker | VIVE Ultimate Tracker（含无线接收器） | 5 | [VIVE Ultimate Tracker（淘宝）](https://e.tb.cn/h.j2riSa4h9gOXIbP?tk=bTaNTF8KLgN) | inside-out 定位，**不需要基站** |
| 头部相机 | GoPro HERO13 Black | 1 | [GoPro HERO13 Black（淘宝）](https://e.tb.cn/h.jX74wO5kS7nAy2D?tk=eQhiTF8vq0V) | |
| 相机媒体模组 | GoPro Media Mod（适用 HERO13/12/11） | 1 | [GoPro 媒体模组（淘宝）](https://e.tb.cn/h.jWg7Rd1eJVpQFGl?tk=sRXrTF8xGCs) | 提供 micro-HDMI 输出 |
| HDMI 采集卡 | 绿联 4K60Hz USB UVC 采集卡 | 1 | [HDMI 采集卡（京东）](https://3.cn/37Yq-tog?jkl=@Q3DcI0h3Iu7W@) | Linux 下为 `/dev/videoN` |
| micro-HDMI → HDMI 线 | Micro HDMI 转 HDMI 线 | 1 | [micro-HDMI 线（京东）](https://3.cn/37YriR-s?jkl=@XB71C3CgoM8X@) | |
| 手套（二选一） | MANUS Metagloves Pro | 1 副 | — | 只有手指关节，无触觉 |
| | Wuji 手套 | 1 副 | — | 手指关节 + 每手 24×31 触觉；通过以太网连接 |
| USB 网口拓展坞 | 绿联 USB3.0 千兆网口扩展坞 | 2 | [USB 网口拓展坞（京东）](https://3.cn/37-YsP1x?jkl=@VER0R1iYlA3G@) | 每只 Wuji 手套一个网口 |
| 头环、绑带 | VIVE tracker 绑带；GoPro 头带 | 若干 | [tracker 绑带（淘宝）](https://e.tb.cn/h.jX7EVqK0xb7nHvB?tk=vOtgTFj8BVK)<br>[GoPro 头带（淘宝）](https://e.tb.cn/h.jc684WNxd6oG6gN?tk=BdM7TFj9A15) | 头环 1、手背绑带 2、踝部绑带 2 |
| Linux 采集机（Wuji 方案） | Ubuntu 22.04+ | 1 | — | 可以是便携设备 |
| Windows 主机 | 运行 SteamVR | 1 | — | Wuji 方案只负责 tracker；MANUS 方案负责全部采集 |

## 1.2 3D 打印件

> **待补充**：STL 文件将放在 [egoexplorer-toolkit](https://github.com/InternRobotics/egoexplorer-toolkit) 仓库的 `hardware/3d_print/` 目录，并附打印参数。

| 打印件 | 用途 | 文件 | 数量 |
|---|---|---|---|
| 头部转接件 | 把 GoPro 与头部 tracker **刚性**连在一起，再固定到头环上 | gopro-tracker.stl | 1 |
| 手背底座 | tracker 固定在手套手背 | wuji-contact.stl | 2 |

## 1.3 五个 tracker 的角色

角色名是固定的，采集端、管线和发布格式全部用同一套名字，**任何环节都不允许按"画面左右"交换**：

| 采集端角色（SteamVR role） | 管线内部角色 | 佩戴位置 | 对应 SMPL-X 关节 |
|---|---|---|---|
| `camera` | `head` | 头部，与 GoPro 刚性相连 | 15（head） |
| `left_wrist` | `left_hand` | 受试者自己的左手手背 | 20（left_wrist） |
| `right_wrist` | `right_hand` | 受试者自己的右手手背 | 21（right_wrist） |
| `left_ankle` | `left_foot` | 受试者自己的左踝外侧 | 7（left_ankle） |
| `right_ankle` | `right_foot` | 受试者自己的右踝外侧 | 8（right_ankle） |

"左"永远指受试者自己的左侧。面对受试者时，他的左手在你的右边，这是正常透视，不是标反了。

## 1.4 为什么头部相机要和头部 tracker 刚性相连

头部 tracker 直接测出相机的 6-DoF 位姿，相机位姿因此是**测量值**，不需要事后用视觉 SLAM 去估。前提是两者之间绝对不能有相对运动：转接件要拧紧，头环不能松。如果录制中途相机被碰歪，那一段的相机外参就不再可信。

## 1.5 穿戴步骤

建议两人配合，一人穿戴、一人检查。

1. **量体**（新受试者或新一轮标定前做，单位 cm，每项量两次取平均）：身高、臂展（中指尖到中指尖）、左右肩峰间距、颈根到会阴高度、会阴到地面高度、左右上臂长、左右前臂长、左右手长（腕横纹到中指尖）、掌宽、左右踝 tracker 中心到地面高度。左右差异明显时分开记。标定时真正用到的是**身高和臂展**（以米输入），其余备查。
2. **戴手套，确认左右。** Wuji 手套按序列号区分左右，并在采集页面里核对左右手显示正确。
3. **装手背 tracker**：底座贴在手套手背中央，绑带勒紧到手腕活动时 tracker 不晃。**左手必须戴 SteamVR 里角色为 `left_wrist` 的那一个**，右手同理。
4. **装踝部 tracker**：脚踝外侧、踝骨上方，绑带勒紧，走路和下蹲时不转、不滑。注意左右角色。
5. **戴头环 + 头部 tracker + GoPro**：头环调到不晃动，相机朝向自然平视方向。
6. **拍两张安装照片**（正面、侧面），记录每个 tracker 的位置和朝向，方便之后排查。
7. **静置自检**：受试者自然站立几秒，在采集页面上确认五个 tracker 都在线、没有跳帧，两只手套都有数据，视频有画面（见第 4.4 节的截图）。

## 1.6 穿戴规则（直接影响数据能不能用）

- **标定和任务录制必须是同一次穿戴。** tracker 被拆下重装、绑带滑动、换受试者之后，都必须重新采一段标定（第 5 章）。坐标修正补偿不了拆装后未知的安装旋转。
- **不要在任务之间调整 tracker。** 安装外参（tracker 相对骨骼的平移和旋转）在整次穿戴里只解一份。
- **左右一定要对。** Tracker需要按照SteamVR中的设置进行配对，否则会影响SMPLX的解算。
- **绑带松了比没绑更糟**：tracker 慢慢滑动会表现为"漂移"，每一帧看都合理，优化器会悄悄把误差吸收进身体姿态里。
- **Wuji 触觉**会随使用缓慢衰减，个别线路可能时断时续。建议每段录制开头在wuji studio中重新完成触觉标定（做约 15 秒的触觉标定动作，便于逐 session 估计健康系数）。
