# 双屏同步播（DualVideo）

极简 Android App：选一个本地视频，左右并排显示 **两个完全同步** 的画面，一个播放/暂停按钮同时控制两边，可拖动进度条，支持横屏全屏。**不联网、不上传，全程本地解码播放。**

## 功能

| 功能 | 说明 |
| --- | --- |
| 选择视频 | 系统文件选择器（SAF），无需任何存储权限；也支持从文件管理器"用其他应用打开"视频 |
| 双画面 | 左、右两个 `PlayerView` 等宽并排，同一视频同一时刻 |
| 同步 | 左画面为主画面，每 100ms 校准一次右画面，偏差 > 60ms 立即对齐；主画面缓冲时副画面一起等待 |
| 播放/暂停 | 一个按钮同时控制两个画面 |
| 进度条 | 可拖动 seek（松手后两边同时跳转到同一位置） |
| 横屏全屏 | 「全屏」按钮：沉浸式隐藏状态栏/导航栏 + 横屏，3 秒后自动隐藏控制条，点画面可唤出，返回键退出 |

## 直接安装 APK（推荐）

产物：**`DualVideo-v1.0.apk`**（约 5.8 MB，已签名，支持 Android 7.0+ / minSdk 24）

1. 把 `DualVideo-v1.0.apk` 传到手机（微信/QQ/USB/蓝牙均可）。
2. 在手机上点击安装，允许"未知来源应用"即可。
3. 打开 App → 点「选择本地视频」→ 选中一个视频 → 左右双屏同步播放。

签名信息（自签名，用途仅为侧载安装）：

```
CN=DualVideo, O=Local, C=CN
SHA-256: caab1e4f2156be70ce39d93e62e05e527f0075f76a7c1f5391338a30b94fd42d
```

> 因为是自签名，手机上若已装过其他渠道的同名包需要先卸载；升级本 APK 也需要先卸载旧版（签名一致时可直接覆盖，本项目固定使用 `dualvideo.jks`）。

## 重新构建

用 **Android Studio** 打开本目录，等待 Gradle 同步后 Run。

产物：`app\build\outputs\apk\release\app-release.apk`

## 实现要点

- `MainActivity` 内创建 **两个 ExoPlayer 实例**（Media3），共享同一个本地 `Uri` 的 `MediaItem`。
- 同步策略：两个独立解码器必然产生毫秒级漂移，因此以左侧为主画面，右侧持续跟随；
  - `|主 - 副| > 60ms` → 副画面 `seekTo(主位置)`
  - 主画面缓冲（`playWhenReady && !isPlaying`）→ 副画面暂停，追上后再继续
  - 进度条 seek → 两个播放器跳到同一时间戳，天然重新对齐
- `android:configChanges` 已声明旋转等配置由 Activity 自行处理，横竖屏切换不中断播放。
- 无任何网络权限，视频文件不会被写出设备。

## 目录结构

```
app/src/main/
├── AndroidManifest.xml
├── java/com/example/dualvideo/MainActivity.kt
└── res/
    ├── layout/activity_main.xml
    └── values/{strings.xml, themes.xml}
```

## 已知限制

- 双路解码 CPU 开销约为单路的 2 倍，4K/高码率视频在低端机上可能掉帧，建议 1080p 及以下。
- 两个独立解码器只能做到"视觉同步"，无法做到单解码器那种逐帧严格一致。
