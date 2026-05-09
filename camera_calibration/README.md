# Camera Calibration

本目录提供针对 **FHHL 实验室**所用 USB 摄像头的棋盘格标定工具和已有标定参数。  
标定结果以 `calib_<name>.json` 格式保存，可直接通过 `calib_file` 参数传入 lerobot 的采集、推理命令，实现**内联去畸变**（每帧写入/读取前自动处理，无需额外后处理步骤）。

---

## 目录结构

```
camera_calibration/
├── calibrate_camera.py      # 棋盘格标定工具（交互式采图 + 计算标定参数）
├── camera_viewer.py         # 多摄像头浏览器预览（确认摄像头 index）
├── verify_undistort.py      # 实时左右对比验证去畸变效果
└── calib_params/            # 标定结果存放目录
    ├── calib_follower_1.json    # follower_1 摄像头（/dev/video4，index=4）
    ├── calib_follower_2.json    # follower_2 摄像头（/dev/video0，index=0）
    ├── calib_top.json           # top 摄像头（/dev/video2，index=2）
    └── res.md                   # 标定运行日志（原始终端输出）
```

---

## 已有标定参数（FHHL 0509）

以下标定参数于 **2025年5月9日** 在 FHHL 实验室完成，均使用 9×6 棋盘格，采集 25 张图像。

| 文件 | 对应摄像头 | `/dev/videoX` | `index` | 重投影误差 | 畸变系数 |
|------|-----------|--------------|---------|-----------|----------|
| `calib_follower_1.json` | front（正面） | `/dev/video4` | `4` | **0.867 px** ✅ | `[-0.3399, 0.1574, -0.0004, -0.0002, -0.0418]` |
| `calib_top.json`        | top（俯视）  | `/dev/video2` | `2` | **0.681 px** ✅ | `[0.0535, -0.1525, -0.0006, 0.0, 0.0985]` |
| `calib_follower_2.json` | 备用 front   | `/dev/video0` | `0` | **0.831 px** ✅ | `[-0.3379, 0.1549, 0.0001, -0.0007, -0.0406]` |

> 重投影误差 < 1.0 px 为优秀（Excellent），上述三个标定结果均满足。  
> ⚠️ 标定参数与**具体摄像头硬件**绑定，更换摄像头或调整焦距后需重新标定。  
> ⚠️ USB 重插后 `/dev/videoX` 编号可能变化，但标定 JSON 仍有效，只需修改命令中的 `index_or_path`。

---

## 如何在 lerobot 命令中使用标定参数

通过在 `--robot.cameras` 的每路摄像头配置中加入 `calib_file` 字段，lerobot 会在每帧采集/推理前自动执行去畸变。

### 路径说明

`calib_file` 支持**相对路径**（相对于运行命令时的工作目录）或**绝对路径**。

**推荐做法**：在 `lerobot/` 根目录下运行命令，路径写为：

```
camera_calibration/calib_params/calib_<name>.json
```

或使用绝对路径（更稳健）：

```
/path/to/lerobot/camera_calibration/calib_params/calib_<name>.json
```

---

### 示例 1：数据采集（`lerobot-record`）— 双摄像头 + 去畸变

在采集时加入 `calib_file`，生成的数据集图像**已是去畸变后的结果**，可直接用于训练，无需批量后处理：

```bash
# 在 lerobot/ 目录下运行
lerobot-record \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=follower_1 \
    --robot.cameras="{
        front: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30,
                calib_file: camera_calibration/calib_params/calib_follower_1.json},
        top:   {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30,
                calib_file: camera_calibration/calib_params/calib_top.json}
    }" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=leader_3 \
    --display_data=true \
    --dataset.repo_id=local/my_task \
    --dataset.root=$(pwd)/data/my_task \
    --dataset.num_episodes=40 \
    --dataset.single_task="My task description" \
    --dataset.streaming_encoding=false \
    --dataset.push_to_hub=false
```

### 示例 2：推理（`lerobot-rollout`）— 仅 front 摄像头

训练时用了去畸变数据，推理时**必须同样启用 `calib_file`**，保证图像分布一致：

```bash
lerobot-rollout \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=follower_1 \
    --display_data=true \
    --robot.cameras="{
        front: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30,
                calib_file: camera_calibration/calib_params/calib_follower_1.json}
    }" \
    --strategy.type=base \
    --policy.path=/path/to/outputs/my_task/checkpoints/010000/pretrained_model
```

### 示例 3：推理（`lerobot-rollout`）— 双摄像头

```bash
lerobot-rollout \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=follower_1 \
    --display_data=true \
    --robot.cameras="{
        front: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30,
                calib_file: camera_calibration/calib_params/calib_follower_1.json},
        top:   {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30,
                calib_file: camera_calibration/calib_params/calib_top.json}
    }" \
    --strategy.type=base \
    --policy.path=/path/to/outputs/my_task/checkpoints/010000/pretrained_model
```

### 示例 4：遥操作（`lerobot-teleoperate`）— 预览去畸变效果

```bash
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id=follower_1 \
    --robot.cameras="{
        front: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30,
                calib_file: camera_calibration/calib_params/calib_follower_1.json},
        top:   {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30,
                calib_file: camera_calibration/calib_params/calib_top.json}
    }" \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id=leader_3 \
    --display_data=true
```

> 💡 **单行格式**（复制到终端时更方便）：
> ```bash
> --robot.cameras="{ front: {type: opencv, index_or_path: 4, width: 640, height: 480, fps: 30, calib_file: camera_calibration/calib_params/calib_follower_1.json}, top: {type: opencv, index_or_path: 2, width: 640, height: 480, fps: 30, calib_file: camera_calibration/calib_params/calib_top.json}}"
> ```

---

## 工具使用说明

### 1. 确认摄像头 index（`camera_viewer.py`）

USB 重新插拔后 `/dev/videoX` 编号可能变化，用此工具在浏览器中确认：

```bash
cd /path/to/lerobot
python camera_calibration/camera_viewer.py
# 浏览器访问 http://localhost:8899
# 页面显示所有摄像头实时画面，左上角标注 index
# 按 Ctrl+C 退出
```

可选参数：

```
--port   PORT    HTTP 监听端口（默认 8899）
--width  WIDTH   预览宽度（默认 640）
--height HEIGHT  预览高度（默认 480）
--fps    FPS     预览帧率（默认 30）
```

---

### 2. 重新标定摄像头（`calibrate_camera.py`）

> ✅ 当前已有有效标定文件，通常无需重新标定。  
> 仅在**更换摄像头硬件**或**调整焦距**后才需重新执行。

准备一张 9×6 棋盘格纸，运行：

```bash
cd /path/to/lerobot

# 标定 front 摄像头（index=4，命名为 follower_1）
python camera_calibration/calibrate_camera.py --index 4 --name follower_1

# 标定 top 摄像头（index=2，命名为 top）
python camera_calibration/calibrate_camera.py --index 2 --name top
```

操作方式：
- 将棋盘格在摄像头前缓慢移动，覆盖画面各区域（中心、四角、不同倾斜角度）
- 检测到棋盘格角点时按 **Space** 采集（建议 25~50 张）
- 采集完成后按 **Q** 执行标定计算并保存
- 结果文件自动保存到 `camera_calibration/calib_params/calib_<name>.json`

重投影误差参考：

| 误差范围 | 评价 |
|---------|------|
| < 0.5 px | 极佳 |
| 0.5 ~ 1.0 px | 优秀（Excellent） |
| 1.0 ~ 2.0 px | 可接受 |
| > 2.0 px | 建议重新标定 |

---

### 3. 验证去畸变效果（`verify_undistort.py`）

实时显示原始图像（左）和去畸变图像（右）对比：

```bash
cd /path/to/lerobot

# 验证 front 摄像头
python camera_calibration/verify_undistort.py \
    --index 4 \
    --calib camera_calibration/calib_params/calib_follower_1.json

# 验证 top 摄像头
python camera_calibration/verify_undistort.py \
    --index 2 \
    --calib camera_calibration/calib_params/calib_top.json
```

操作：按 **S** 截图保存对比图，按 **Q** 退出。

---

## 重新标定的时机

| 情形 | 是否需要重新标定 |
|------|----------------|
| USB 重新插拔，`/dev/videoX` 编号变化 | ❌ 不需要，只需更新命令中的 `index_or_path` |
| 更换了摄像头硬件（不同镜头） | ✅ 需要 |
| 调整了摄像头焦距 / 变焦 | ✅ 需要 |
| 肉眼可见画面有明显桶形/枕形畸变 | ✅ 建议重新标定 |
| 标定文件丢失或损坏 | ✅ 需要 |
