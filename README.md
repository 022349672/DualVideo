# 双屏同步播（DualVideo）

极简 Android App：选一个本地视频，左右并排显示 **两个完全同步** 的画面，一个播放/暂停按钮控制，可拖动进度条，支持横屏全屏。**不联网、不上传，全程本地解码播放。**

## 直接安装

**`DualVideo-v1.4.apk`**（约 4.7 MB，已签名，Android 7.0+ / minSdk 24）

1. 传到手机（微信/QQ/USB/蓝牙均可），点击安装，允许"未知来源应用"。
2. 打开 → 「选择本地视频」→ 选一个视频 → 左右双屏同步播放。
3. 已装过旧版可直接覆盖安装（同一签名，versionCode 5）。

> **崩溃了怎么办**：v1.3 起，再次打开 App 会**自动弹出上次崩溃的堆栈**，点「复制」后直接发给我就行，不用去翻文件夹。
> 同时也会写一份到手机「下载 / dualvideo-crash.txt」。
> （非主线程崩溃没有系统弹窗，看起来就是"应用直接消失"。）

签名信息（自签名，仅用于侧载）：

```
CN=DualVideo, O=Local, C=CN
SHA-256: caab1e4f2156be70ce39d93e62e05e527f0075f76a7c1f5391338a30b94fd42d
```

> 自签名包：手机若装过其他渠道的同名包，需先卸载再装。

## 同步原理（v1.1：单解码器）

v1.0 用的是"两个 ExoPlayer + 定时校准"，两个独立解码器必然存在几十毫秒漂移，且 4K 双路解码会直接把内存/解码资源吃满导致闪退。

v1.1 改为 **单解码器 + GPU 双视口**：

```
本地文件 → 唯一一个 ExoPlayer/MediaCodec 解码
                ↓（输出到我们自己创建的 SurfaceTexture）
        OES 外部纹理（同一帧画面）
                ↓ 同一次 onDrawFrame、同一次 vsync
        ┌───────────────┬───────────────┐
        │   左视口       │   右视口       │
        └───────────────┴───────────────┘
```

- 左右两侧来自**同一张纹理、同一次绘制、同一次 vsync**，不存在"右边慢一点"的可能；
- 解码器从 2 个降到 1 个，CPU/GPU/内存占用减半，4K 不再双路打满；
- 音频也只有一路，不会出现双音轨叠加；
- 代码里不再需要任何"校准/追赶"逻辑。

相关实现：
- `DualStageView.kt`：`GLSurfaceView` + OES 纹理 + `SurfaceTexture`，把视频帧按等比缩放画到左右两个 viewport；
- `MainActivity.kt`：单个 ExoPlayer，通过 `setVideoSurface()` 把输出接到上面的渲染面。

## 4K 优化

| 措施 | 作用 |
| --- | --- |
| 单解码器 | 解码/内存开销减半（最主要的一条） |
| `setEnableDecoderFallback(true)` | 硬件解码器不支持该 4K 编码（如 HEVC 10bit）时回退软解，而不是直接崩 |
| `setTargetBufferBytes(16MB)` + 限制缓冲时长 | 4K 码率高，避免缓冲一次性吃掉几百 MB |
| `android:largeHeap="true"` | 给 4K 解码留出更多堆内存 |
| GLSurfaceView（SurfaceView 体系） | 直接送显，比 TextureView 少一次 GPU 拷贝 |
| 播放失败走 `onPlayerError` | 提示"无法播放：xxx"，而不是崩溃退出 |

## 功能

| 功能 | 说明 |
| --- | --- |
| 选择视频 | 系统文件选择器（SAF），无需存储权限；也支持从文件管理器"用其他应用打开" |
| 双画面 | 左右等宽并排，同一帧画面 |
| 播放/暂停 | 单一按钮 |
| 进度条 | 可拖动 seek，实时显示 `当前 / 总时长`，并显示分辨率 |
| 横屏全屏 | 「全屏」进入沉浸式横屏，控制条 3 秒自动隐藏，点画面唤出，返回键退出 |

## 重新构建

方式一：Android Studio 打开本目录，Gradle 同步后 Run。

方式二：命令行（本机工具链在 `D:\AndroidTools`）：

```bat
set JAVA_HOME=D:\AndroidTools\jdk-17.0.20.1+1
set ANDROID_HOME=D:\AndroidTools\Sdk
set GRADLE_USER_HOME=D:\AndroidTools\.gradle
gradlew assembleRelease
```

产物：`app\build\outputs\apk\release\app-release.apk`

## 目录结构

```
app/src/main/
├── AndroidManifest.xml
├── java/com/example/dualvideo/
│   ├── MainActivity.kt      # 单 ExoPlayer + 控制逻辑
│   └── DualStageView.kt     # OES 纹理 + OpenGL 左右双视口渲染
└── res/
    ├── layout/activity_main.xml
    └── values/{strings.xml, themes.xml}
```

## 版本记录

- **v1.4**：修复"选完视频必崩"。真凶是 `DefaultLoadControl.Builder.setBufferDurationsMs()` 抛
  `IllegalArgumentException: maxBufferMs cannot be less than minBufferMs` —— 我传了
  `DEFAULT_MIN_BUFFER_MS`（Media3 里默认就是 50s）当 min、却把 max 写成 30s。
  改为显式给出 min=15s / max=30s / 起播 2.5s / 卡顿后 5s 四个值，不再混用 `DEFAULT_*` 常量。
- **v1.3**：崩溃可自查。GL 线程异常不再静默杀进程（渲染器内部兜住并提示 + 落盘）；下次启动自动弹窗显示上次崩溃堆栈并支持一键复制。
- **v1.2**：修复"选完视频就闪退"。原因：`SurfaceTexture` 的可用回调发生在 **GL 渲染线程**，而回调里直接调用了 `player.setVideoSurface()` / `seekTo()`，ExoPlayer 强制要求主线程访问，于是在 GL 线程抛出 `IllegalStateException: Player is accessed on the wrong thread`；非主线程异常没有崩溃弹窗，进程被直接杀掉。修复方式：回调统一 `Handler(Looper.getMainLooper()).post` 回主线程。另加了崩溃日志落盘。
- **v1.1**：改为单解码器 + OpenGL 双视口，彻底解决左右不同步；4K 不再双路解码。
- **v1.0**：两个 ExoPlayer + 定时校准（已废弃）。

## 已知限制

- 4K@60fps 或高码率 4K 在老旧机型上仍可能掉帧（单解码器已是最优解）。
- HDR/10bit 视频按 SDR 输出，色彩会偏淡（未做 tone mapping）。
- 不支持后台播放：切到后台会暂停（省电考虑）。
