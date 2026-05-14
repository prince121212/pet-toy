# Pet Toy - 透明 WebM 视频多浏览器兼容方案

## 问题现象

| 浏览器 | 表现 |
|---|---|
| Edge (桌面) | ✅ 正常显示，透明背景 |
| 安卓微信内置浏览器 | ❌ 无法加载 WebM 视频 |
| 安卓 QQ 浏览器 | ❌ 视频显示灰色默认背景，丢失透明效果 |
| 部分浏览器 | ❌ 右键/长按出现"下载视频""暂停"等原生控件 |

核心诉求：**视频在网页中应该"无痕"，用户感知不到它是视频，如同 3D 渲染动画。**

---

## 根因分析

### 1. WebM + VP9 Codec 兼容性问题

当前视频文件：

```
transparent-1778528599049.webm  → VP9 + Opus, 720×960, 30fps, ALPHA_MODE=1
siri.webm                       → VP9 + Opus, 1280×960, 24fps, ALPHA_MODE=1
```

- **微信安卓内置浏览器** 使用腾讯 X5 WebView 内核，对 VP9 解码支持极差，WebM 容器也缺乏完整支持。
- **QQ 浏览器安卓** 同样使用 X5 内核，对 VP9 的 **Alpha 通道**（`ALPHA_MODE=1`）不支持合成，只能显示默认灰色背景。
- **iOS Safari / WKWebView** 对 VP9 支持受限于硬件解码能力（A12 以上芯片才支持），且 WebM alpha 合成行为不一致。

### 2. Alpha 通道合成不统一

WebM VP9 的 Alpha 通道是 **非标准扩展**。Chrome/Edge 桌面版能正确处理，但移动端 WebView（尤其是国内厂商定制内核）普遍不支持将视频的 Alpha 通道与页面 CSS 背景合成。

### 3. 浏览器原生视频控件暴露

`<video>` 元素在没有以下防护时，浏览器会暴露原生 UI：

- **右键菜单**：显示"保存视频""视频另存为""画中画"等
- **长按菜单**（移动端）：显示"下载视频""在新窗口打开"
- **播放控件**：部分浏览器在视频区域悬停时自动显示进度条、暂停按钮
- **画中画按钮**：iOS/Android 部分浏览器在视频上覆盖 PiP 图标

浏览器将 `<video>` 识别为"视频内容"并提供特殊交互，这与"看起来像 3D 渲染动画"的目标冲突。

---

## 解决方案

### 方案一：Canvas 逐帧渲染（⭐ 推荐）

将视频的每一帧绘制到 `<canvas>` 上，彻底隐藏视频身份。

**原理**：使用 `requestAnimationFrame` 循环将 `<video>` 的当前帧绘制到隐藏的 `<canvas>`，用户看到的是 Canvas 而非 Video 元素，浏览器不会对 Canvas 施加任何视频交互行为。

```html
<!-- 隐藏的视频源，仅供提取帧数据 -->
<video
  src="transparent-1778528599049.webm"
  muted loop playsinline
  style="display:none;"
  id="sourceVideo"
></video>

<!-- 用户看到的画布 -->
<canvas id="petCanvas"></canvas>

<script>
  const video = document.getElementById('sourceVideo');
  const canvas = document.getElementById('petCanvas');
  const ctx = canvas.getContext('2d', { alpha: true });

  canvas.width = 720;
  canvas.height = 960;

  video.addEventListener('play', function draw() {
    if (video.paused || video.ended) return;
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
    requestAnimationFrame(draw);
  });

  video.play();
</script>
```

**Canvas 方案优势**：

| 维度 | Canvas | 原生 `<video>` |
|---|---|---|
| 右键/长按暴露 | ❌ 无视频菜单 | ⚠️ 显示视频选项 |
| 控件覆盖 | ❌ 永远不会出现 | ⚠️ 可能自动出现 |
| 透明背景 | ✅ 天然支持 | ⚠️ 依赖浏览器 Alpha 合成 |
| 感知为视频 | ❌ 不感知 | ✅ 浏览器识别为视频 |
| 兼容性 | ✅ 所有支持 Canvas 的浏览器 | ⚠️ X5/WebView 问题多 |

### 方案二：为 `<video>` 添加防护属性（辅助方案）

如果不想改用 Canvas，至少应添加以下属性减少暴露：

```html
<video
  src="pet.webm"
  autoplay muted loop playsinline
  disablePictureInPicture            <!-- 禁用画中画 -->
  disableRemotePlayback              <!-- 禁用投屏 -->
  controlslist="nodownload nofullscreen noremoteplayback"  <!-- 禁用下载/全屏 -->
  oncontextmenu="return false"       <!-- 禁用右键菜单 -->
  style="pointer-events:none;"       <!-- 禁用所有鼠标/触摸交互 -->
  preload="auto"
></video>
```

### 方案三：多格式 Fallback（兼容微信 X5 内核）

为微信等不支持 WebM 的浏览器提供 MP4 回退。注意 MP4 不支持 Alpha 通道，需预处理为带黑色/纯色背景的版本。

```html
<video autoplay muted loop playsinline disablePictureInPicture
       controlslist="nodownload" oncontextmenu="return false;">
  <source src="pet.webm" type="video/webm; codecs=vp9,opus">
  <source src="pet-fallback.mp4" type="video/mp4">
</video>
```

对于透明背景需求，Canvas + WebM 仍然是唯一可靠路径。

### 方案四：转换为 APNG / Animated WebP

如果动画较短（<5秒），可考虑转为 APNG 或 Animated WebP + `<img>` 标签：

```bash
# WebM → APNG（保留透明通道）
ffmpeg -i pet.webm -vf "fps=15" pet.png
# 然后将帧序列合成为 APNG（需 apngasm 或在线工具）
```

优点：`<img>` 标签完全不被识别为视频。缺点：无法承载长动画（文件体积过大）。

---

## 推荐实施步骤

1. **首选 Canvas 渲染**：隐藏 `<video>`，用 Canvas 逐帧绘制显示
2. **保留防护属性**：即使用了 Canvas，`<video>` 源元素也加上 `disablePictureInPicture` 等
3. **添加 MP4 Fallback**：为不支持 WebM 的浏览器（微信 X5）提供备用 MP4
4. **预加载优化**：`preload="auto"` + 监听 `canplaythrough` 事件后再显示 Canvas
5. **移动端点击触发播放**：部分移动浏览器要求用户手势才能 `play()`，需在首次 touch 事件中触发

---

## 文件说明

| 文件 | 用途 |
|---|---|
| `index.html` | 主页面（含 Canvas 渲染逻辑） |
| `pet-avatar.html` | 简化版页面 |
| `transparent-1778528599049.webm` | 宠物动画（VP9 + Alpha） |
| `siri.webm` | 语音球动画（VP9 + Alpha） |
| `manifest.webmanifest` | PWA 清单 |

---

## 浏览器兼容性速查

| 浏览器 | VP9 解码 | WebM Alpha | 推荐方案 |
|---|---|---|---|
| Chrome/Edge 桌面 | ✅ | ✅ | Canvas / video |
| Safari 桌面 14+ | ✅ | ⚠️ 部分 | Canvas |
| iOS Safari 14+ | ⚠️ A12+ | ❌ | Canvas |
| 微信安卓 X5 | ❌ | ❌ | Canvas + MP4 Fallback |
| QQ 浏览器安卓 | ❌ 部分 | ❌ | Canvas + MP4 Fallback |
| Firefox 桌面 | ✅ | ✅ | Canvas / video |
