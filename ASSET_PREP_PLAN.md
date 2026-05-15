# 宠物陪伴素材准备计划

## 1. 目标

素材要做成“角色包”，不要做成“猫咪专用资源”。

猫、狗、鸟、卡通 IP 都必须使用同一套动作 ID、同一套 manifest 结构。程序只认行为语义，不认物种。

## 2. 角色包结构

```text
cdn/pets/{character_id}/
├── standard/
│   ├── manifest.json
│   ├── idle_breathe_01/
│   ├── sleep_loop_01/
│   ├── touch_affection_01/
│   └── ...
└── low/
    ├── manifest.json
    ├── idle_breathe_01/
    ├── sleep_loop_01/
    ├── touch_affection_01/
    └── ...
```

包内不放完整素材。素材放 CDN/云存储，按需下载和缓存。

## 3. 动作命名

命名描述行为，不描述物种。

格式：

```text
{state_or_action}_{detail}_{variant}
```

示例：

```text
idle_breathe_01
sleep_loop_01
touch_affection_01
play_object_01
wake_up_01
```

禁止：

```text
cat_rub_01
dog_tail_01
bird_flap_01
```

## 4. 播放类型

只保留两类：

| 类型 | 说明 |
|---|---|
| `loop` | 可长期循环的状态动作 |
| `transition` | 播放一次后进入目标动作 |

触摸、玩耍、入睡、醒来都是行为语义，不是播放类型。

## 5. MVP 动作清单

| 动作 ID | 类型 | 行为含义 | 猫 | 狗 | 鸟/IP |
|---|---|---|---|---|---|
| `idle_breathe_01` | `loop` | 默认待机 | 坐着呼吸 | 站坐呼吸 | 站立/漂浮呼吸 |
| `sleep_loop_01` | `loop` | 睡眠循环 | 睡觉呼吸 | 睡觉呼吸 | 收翅/闭眼轻动 |
| `touch_affection_01` | `transition` | 亲密反馈 | 蹭蹭 | 摇尾贴近 | 开心靠近 |
| `touch_annoyed_01` | `transition` | 不满反馈 | 甩尾躲开 | 后退/低哼 | 飞开/皱眉 |
| `play_object_01` | `transition` | 玩具互动 | 玩毛线球 | 玩球/骨头 | 啄玩具 |
| `sleep_enter_01` | `transition` | 进入睡眠 | 趴下闭眼 | 趴下 | 收翅入睡 |
| `wake_up_01` | `transition` | 醒来 | 睁眼抬头 | 抬头 | 睁眼展开 |
| `idle_blink_01` | `transition` | 微动作 | 眨眼 | 眨眼 | 眨眼/眼睛闪动 |

首版就做这 8 个。后续再补：

```text
idle_look_01
idle_groom_01
idle_stretch_01
touch_happy_01
play_object_02
```

## 6. 连续性规范

角色不能瞬移、变大、变小。

同一个角色包必须统一：

| 项目 | 要求 |
|---|---|
| 画布尺寸 | 同一档位完全一致，例如 `540x720` |
| 角色比例 | 同一角色包内大小稳定 |
| 角色中心线 | 主体中心在统一 x 坐标附近 |
| 地面线/锚点 | 站立、坐下、趴下必须对齐 |
| 光照风格 | 不同动作亮度和边缘一致 |
| 安全边距 | 动作不能出画布 |

每个动作必须记录锚点：

```json
{
  "anchor": {
    "type": "ground_center",
    "x": 270,
    "y": 650
  }
}
```

常用锚点：

| 锚点 | 适用对象 |
|---|---|
| `ground_center` | 猫、狗、站立 IP |
| `body_center` | 鸟、漂浮 IP |
| `seat_center` | 坐姿角色 |
| `head_center` | 头像型 IP |

## 7. 首尾帧要求

| 动作 | 来源 | 目标 |
|---|---|---|
| `touch_affection_01` | `idle_breathe_01` | `idle_breathe_01` |
| `touch_annoyed_01` | `idle_breathe_01` | `idle_breathe_01` |
| `play_object_01` | `idle_breathe_01` | `idle_breathe_01` |
| `sleep_enter_01` | `idle_breathe_01` | `sleep_loop_01` |
| `wake_up_01` | `sleep_loop_01` | `idle_breathe_01` |
| `idle_blink_01` | `idle_breathe_01` | `idle_breathe_01` |

要求：

- `loop` 首尾帧要能自然循环。
- `transition` 第一帧接近来源动作。
- `transition` 最后一帧接近目标动作。
- 锚点偏移尽量控制在 3-8 像素内。

## 8. 源素材要求

| 项目 | 建议 |
|---|---|
| 背景 | 纯绿，光照均匀 |
| 格式 | MP4 / MOV |
| 源帧率 | 24fps 或 30fps |
| 运行帧率 | 12-15fps |
| 源分辨率 | 至少 720x960 |
| 运行尺寸 | 标准 `480x640`，高清 `540x720` |
| 镜头 | 固定，不缩放 |

优先绿幕，不建议黑底。黑底容易伤暗色细节和半透明边缘。

## 9. 处理流程

先抽帧试抠：

```bash
ffmpeg -ss 00:00:02 -i input.mp4 -frames:v 1 test_raw.png

ffmpeg -i test_raw.png \
  -vf "chromakey=0x00ff00:0.18:0.08,format=rgba" \
  test_alpha.png
```

批量导出透明 PNG：

```bash
mkdir -p frames/touch_affection_01

ffmpeg -i source/touch_affection_01.mp4 \
  -vf "fps=15,scale=540:-1,chromakey=0x00ff00:0.18:0.08,format=rgba" \
  frames/touch_affection_01/frame_%04d.png
```

压缩：

```bash
oxipng -o 4 frames/touch_affection_01/*.png
```

可选 WebP：

```bash
mkdir -p webp/touch_affection_01

ffmpeg -i frames/touch_affection_01/frame_%04d.png \
  -c:v libwebp \
  -lossless 0 \
  -q:v 80 \
  webp/touch_affection_01/frame_%04d.webp
```

WebP 上线前必须真机验证，并保留 PNG 兜底。

## 10. 帧序列组织

每个动作一个目录，帧文件按顺序命名。

```text
touch_affection_01/
├── frame_0001.webp
├── frame_0002.webp
├── frame_0003.webp
└── frame_0060.webp
```

建议：

- 不使用 spritesheet。
- 每个动作独立目录。
- 帧号固定 4 位或 5 位。
- 运行时只加载当前动作帧。
- 下一动作只做少量预加载。

## 11. Manifest 示例

```json
{
  "version": "pet-v1",
  "characterId": "cat_default",
  "quality": "standard",
  "canvas": {
    "width": 540,
    "height": 720
  },
  "actions": {
    "touch_affection_01": {
      "type": "transition",
      "from": ["idle_breathe_01"],
      "to": "idle_breathe_01",
      "fps": 15,
      "frames": 60,
      "format": "webp",
      "frameWidth": 540,
      "frameHeight": 720,
      "anchor": {
        "type": "ground_center",
        "x": 270,
        "y": 650
      },
      "files": [
        "touch_affection_01/frame_0001.webp",
        "touch_affection_01/frame_0002.webp",
        "touch_affection_01/frame_0003.webp",
        "touch_affection_01/frame_0004.webp"
      ],
      "audio": [
        {
          "id": "voice_affection_01",
          "src": "audio/voice_affection_01.mp3",
          "startMs": 600,
          "volume": 0.45,
          "probability": 0.4
        }
      ]
    }
  }
}
```

## 12. 声音规范

声音也放进角色包。

| 音频 ID | 含义 |
|---|---|
| `voice_affection_01` | 亲密声音 |
| `voice_annoyed_01` | 不满声音 |
| `voice_wake_01` | 醒来声音 |
| `amb_sleep_breath_01` | 睡眠呼吸 |
| `sfx_play_object_01` | 玩具声音 |
| `sfx_step_01` | 脚步/移动 |

原则：

- 不要每次都响。
- 音量偏低。
- 0.3-2 秒为主。
- 必须有静音开关。
- 跟动作时间点同步，例如第 600ms 轻叫。

## 13. 质量检查

| 检查项 | 标准 |
|---|---|
| 命名 | 使用通用动作 ID |
| 类型 | 每个动作标注 `loop` 或 `transition` |
| 锚点 | 切换前后不跳 |
| 尺寸 | 同一角色包内比例稳定 |
| 首尾帧 | 能接来源和目标动作 |
| 边缘 | 无明显绿边 |
| 体积 | 不一次性加载所有帧 |
| 真机 | iOS 和 Android 都通过 |

## 14. 内存预算

```text
单帧解码内存 = 宽 x 高 x 4 字节
```

| 尺寸 | 单帧内存 |
|---:|---:|
| 480x640 | 约 1.17 MB |
| 540x720 | 约 1.48 MB |
| 720x960 | 约 2.64 MB |

运行时只保留当前动作和少量预加载资源。

目标：

- 当前动作资源：20-60 MB。
- 预加载资源：10-30 MB。
- 低端机活跃动画资源低于 40 MB。
