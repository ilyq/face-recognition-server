# Android 人脸识别 AAR 使用手册

## 1. 交付内容

第三方项目需要获得以下文件：

```text
demi-face.aar
models/
  face_detection.dfme            # 检测模型
  face_recognition.dfme          # 特征模型
  head_pose.dfme                 # 头部姿态模型，可选
jni/arm64-v8a/
  librknnrt.so
```

文件说明：

| 文件 | 用途 |
|---|---|
| `demi-face.aar` | Android SDK 主库 |
| `face_detection.dfme` | 检测模型 |
| `face_recognition.dfme` | 特征模型 |
| `head_pose.dfme` | 头部姿态模型，可选 |
| `librknnrt.so` | Rockchip Android arm64 运行库 |

## 2. Android 工程配置

将 AAR 放入：

```text
app/libs/demi-face.aar
```

在 `app/build.gradle` 中添加本地依赖：

```kotlin
dependencies {
    implementation(files("libs/demi-face.aar"))
}
```

将模型放入：

```text
app/src/main/assets/models/
```

将运行库放入：

```text
app/src/main/jniLibs/arm64-v8a/librknnrt.so
```

应用必须包含 `arm64-v8a` ABI：

```kotlin
android {
    defaultConfig {
        ndk {
            abiFilters += "arm64-v8a"
        }
    }
}
```

## 3. 初始化 SDK

SDK 通过 `demiface.Demiface` 创建配置和服务。以下示例展示主要配置项：

```kotlin
val demiface = Class.forName("demiface.Demiface")
val config = demiface.getMethod("newConfig").invoke(null)
val configClass = config.javaClass

configClass.getMethod("setDBPath", String::class.java)
    .invoke(config, File(filesDir, "faces.db").absolutePath)
configClass.getMethod("setDetModel", String::class.java)
    .invoke(config, detectorModelFile.absolutePath)
configClass.getMethod("setRecModel", String::class.java)
    .invoke(config, recognitionModelFile.absolutePath)
configClass.getMethod("setInputSize", Long::class.javaPrimitiveType!!)
    .invoke(config, 256L)
configClass.getMethod("setScoreThresh", Double::class.javaPrimitiveType!!)
    .invoke(config, 0.6)
configClass.getMethod("setNMSThresh", Double::class.javaPrimitiveType!!)
    .invoke(config, 0.3)
configClass.getMethod("setMaxNum", Long::class.javaPrimitiveType!!)
    .invoke(config, 10L)

configClass.getMethod("setHeadPose", Boolean::class.javaPrimitiveType!!)
    .invoke(config, false)

val service = demiface.getMethod("newService", configClass)
    .invoke(null, config)
```

如果启用头部姿态，需要同时设置模型路径：

```kotlin
configClass.getMethod("setHeadPose", Boolean::class.javaPrimitiveType!!)
    .invoke(config, true)
configClass.getMethod("setHeadPoseModel", String::class.java)
    .invoke(config, headPoseModelFile.absolutePath)
```

注意：Go 的 `int` 参数通过 gomobile 导出后，在 Java 反射中对应 primitive `long`，因此 `setInputSize` 和 `setMaxNum` 必须使用 `Long`。

## 4. 模型文件处理

模型应从 APK assets 复制到应用私有目录后再传给 SDK：

```kotlin
fun copyModel(context: Context, assetName: String): File {
    val output = File(context.filesDir, "models/$assetName")
    output.parentFile?.mkdirs()
    if (!output.exists()) {
        context.assets.open("models/$assetName").use { input ->
            output.outputStream().use { outputStream -> input.copyTo(outputStream) }
        }
    }
    return output
}
```

模型文件必须与交付的 AAR 和 `librknnrt.so` 配套使用。更新模型后，应删除应用私有目录中的旧模型，或清除应用数据后重新启动。

## 5. 人脸注册

`addFace` 接收可解码的 JPEG 或 PNG 字节：

```kotlin
val json = service.javaClass
    .getMethod(
        "addFace",
        Long::class.javaPrimitiveType!!,
        String::class.java,
        ByteArray::class.java
    )
    .invoke(service, 1001L, "张三", imageBytes) as String
```

参数：

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | `Long` | 人脸唯一 ID |
| `name` | `String` | 人员名称 |
| `imageBytes` | `ByteArray` | JPEG 或 PNG 图片数据 |

## 6. 人脸识别

`identify` 接收一张图片，返回检测到的人脸和匹配结果：

```kotlin
val json = service.javaClass
    .getMethod("identify", ByteArray::class.java, Double::class.javaPrimitiveType!!)
    .invoke(service, imageBytes, 0.55) as String
```

返回 JSON 示例：

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "faces": [
      {
        "detection": {
          "score": 0.98,
          "box": [120, 80, 360, 340],
          "landmarks": [[180, 160], [290, 160]]
        },
        "result": {
          "matched": true,
          "hit": {
            "id": 1001,
            "name": "张三",
            "distance": 0.12,
            "approx_cosine": 0.88
          }
        }
      }
    ]
  }
}
```

`matched` 为 `false` 时，`hit` 可能为空。应用应同时检查响应 `code`、`data`、`faces` 和 `result.matched`。

## 7. 摄像头接入

使用 Camera2 的 `YUV_420_888` 时，应正确处理每个 plane 的：

- `rowStride`
- `pixelStride`
- U/V 平面顺序

然后转换为有效的 NV21 或 JPEG，再传给 `identify`。不能只复制 Y 平面或只复制 V 平面，否则 SDK 会持续返回空的人脸列表。

建议先确认：

1. Android 预览画面能看到清晰的人脸。
2. 转换后的 JPEG 能被普通图片查看器打开。
3. 输入图片尺寸和方向正确。
4. 再检查 SDK 检测结果和识别结果。

## 8. 生命周期

SDK 初始化完成后可以重复调用 `addFace` 和 `identify`。Activity 或 Service 销毁时必须释放 SDK：

```kotlin
service.javaClass.getMethod("close").invoke(service)
```

不要在每一帧图像上重复初始化 SDK。建议复用同一个 service，并在后台线程执行推理。

## 9. 错误排查

### SDK 初始化失败

检查：

- AAR、模型和 `librknnrt.so` 是否为同一版本交付物。
- APK 是否包含 `arm64-v8a`。
- 模型路径是否为应用私有目录中的实际路径。
- 是否启用了头部姿态但没有提供 `head_pose.dfme`。

### 检测结果为空

检查摄像头预览、输入 JPEG 内容、YUV 转换和模型文件。检测 timing 为几十毫秒但上层识别只有几毫秒时，通常表示检测到的人脸数量为 0，尚未进入特征提取和匹配阶段。

### 替换模型后仍使用旧模型

删除应用私有目录中的模型文件，或执行：

```bash
adb shell pm clear <application-id>
```

### 设备 ABI 不兼容

SDK 当前要求 Rockchip Android `arm64-v8a`。如果设备不是 arm64，不能使用本交付包。

## 10. 功能范围

当前 Android SDK 提供人脸检测、特征提取、识别、人脸注册、人脸管理和可选头部姿态信息。SDK 不提供真假人脸或活体判断接口。
