# Face Recognition Server

> High Performance Face Recognition Server for Edge AI

Face Recognition Server 是一个面向边缘 AI 的高性能人脸识别服务。

它将人脸检测、人脸识别、人脸管理等能力封装为统一的 WebSocket API，开发者无需关注底层 AI 推理框架，即可快速构建门禁、考勤、访客管理、AI 摄像机等应用。

支持 RK3576、RV1106 等 Rockchip NPU 平台，同时提供 Windows 开发验证环境，所有平台保持一致的接口协议。

---

## Why Face Recognition Server

### 🚀 高性能

针对边缘 AI 平台优化，充分利用 Rockchip NPU，实现实时人脸检测与识别。

### 🔌 统一 WebSocket API

所有平台均提供统一 WebSocket API，客户端无需关心底层运行环境。

支持：

- Web
- Windows
- Linux
- Android
- iOS
- Python
- Java
- C#
- C/C++
- Go

### 💻 本地离线部署

所有模型均运行于本地设备，无需联网，不依赖任何云服务。

### ⚡ 快速集成

客户端只需连接 WebSocket，即可完成：

- 人脸检测
- 人脸识别
- 人脸注册
- 人脸管理

无需了解 AI 模型、NPU 或推理框架。

---

## Features

- Face Detection
- Face Recognition
- Face Enrollment
- Face Search
- Face Management
- SQLite Face Database
- WebSocket API
- HTTP Health Check
- Offline Deployment
- Multi-platform Support

---

## Applications

适用于：

- 门禁系统
- 人脸考勤
- 访客管理
- AI 摄像机
- 边缘计算设备
- 工业终端
- 智能零售
- 本地离线 AI 应用

---

## Architecture

```text
                    WebSocket API

        Web
        Windows
        Android
        Linux
        Python
        Java
        C#
        Go
            │
            │
            ▼

    Face Recognition Server

            │

    Face Detection
    Face Recognition
    Face Management

            │

     RK3576 / RV1106 / Windows
```

---

## Performance

典型性能（单人图片，参考数据）。

| Platform | Face Detection | Face Recognition | Deployment |
|----------|---------------:|-----------------:|------------|
| RK3576 | ~45 ms | ~6 ms | ⭐ 推荐 |
| RV1106 | ~80 ms | ~22 ms | ⭐ 推荐 |
| Windows (ONNX CPU) | 开发验证 | 开发验证 | ⭐ 推荐 |

---

## Supported Platforms

| Platform | Status | Description |
|----------|--------|-------------|
| RK3576 | ✅ | Production |
| RV1106 | ✅ | Production |
| Windows x64 | ✅ | Production & Development & Testing |

---

## Product Components

### Face Recognition Server

核心服务程序。

提供统一的人脸识别服务，包括：

- 人脸检测
- 人脸识别
- 人脸注册
- 人脸管理
- WebSocket API
- HTTP Health Check

推荐部署于：

- RK3576
- RV1106

---

### Face Recognition App

官方桌面客户端。

用于：

- 摄像头实时预览
- 实时人脸识别
- 人脸管理
- 功能演示
- API 验证

所有 AI 推理均由 Face Recognition Server 完成。

---

### Android AAR

Android AAR 是独立的本地 SDK，直接在 Android 设备上执行人脸检测、识别、人脸注册和人脸管理，不依赖 Face Recognition Server 或网络连接。

Android 集成包包含：

- `demi-face.aar`
- `face_detection.dfme`
- `face_recognition.dfme`
- 可选的 `head_pose.dfme`
- Android `arm64-v8a` 的 RKNN 运行库

Android 项目需要将 AAR、模型和运行库分别放入 `libs`、`assets/models` 和 `jniLibs/arm64-v8a` 目录。完整的集成、初始化、接口调用、摄像头输入和错误排查说明请参考 [Android AAR 使用手册](android-aar.md)。

---

## Platform Support

支持以下 Rockchip 平台运行方式：

- Linux RK 系列设备：rk3588, rk3576, rk3568, rk3566等
- Android RK 系列设备：rk3588, rk3576, rk3568, rk3566等

## Development Services

支持基于现有 SDK 和服务进行二次开发，包括：

- Android 应用集成
- Linux RK 设备部署
- 人脸识别业务流程开发
- 接口和数据结构扩展
- 摄像头和设备适配
- 模型、性能和功能定制

如需 SDK 集成、二次开发或定制开发，请联系：

`QQ：850246539@qq.com`

## Documentation

- [Android AAR 使用手册](android-aar.md)

完整的接口协议和使用说明请参考：

- 📘 [Server API](server-api.md)

包括：

- WebSocket API
- HTTP Health API
- 请求格式
- 响应格式
- 错误码
- 示例代码

---
