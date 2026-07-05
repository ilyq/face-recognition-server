# Face Recognition Server 接口文档

## 服务启动

```bash
# rknn
./face-server -addr :8080 -det-model ./models/face_detection.rkme -rec-model ./models/face_recognition.rkme

# onnx
./face-server -addr :8080 -det-model ./models/face_detection.dfme -rec-model ./models/face_recognition.dfme
```

常用启动参数：

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `-addr` | `:8080` | HTTP/WebSocket 监听地址 |
| `-db` | `face.db` | 人脸库文件路径 |
| `-det-model` | 空 | 人脸检测模型路径 |
| `-rec-model` | 空 | 人脸识别模型路径 |
| `-score-thresh` | `0.6` | 人脸检测置信度阈值 |
| `-nms-thresh` | `0.3` | NMS 阈值 |
| `-max-num` | `10` | NMS 后最多保留的人脸数量 |
| `-version` | `false` | 打印版本并退出 |

## HTTP 接口

### 健康检查

```http
GET /healthz
```

响应：

```json
{"ok":true}
```

## WebSocket 接口

连接地址示例：

```text
ws://127.0.0.1:8080/ws
```

### 通用消息格式

客户端发送文本 JSON：

```json
{
  "id": "req-001",
  "action": "ping",
  "payload": {}
}
```

字段说明：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | string | 否 | 请求 ID，响应会原样带回 |
| `action` | string | 是 | 操作名称 |
| `payload` | object | 否 | 操作参数 |

服务端响应：

```json
{
  "id": "req-001",
  "action": "ping",
  "code": 0,
  "msg": "ok",
  "data": {}
}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 对应请求的 `id` |
| `action` | string | 对应请求的 `action` |
| `code` | int | 状态码，`0` 表示成功 |
| `msg` | string | 状态信息 |
| `data` | object/array | 业务数据；失败时为空对象 |

### 状态码

| code | 含义 |
| --- | --- |
| `0` | 成功 |
| `100` | 业务错误，例如模型推理、图片解码、数据存储等错误 |
| `1001` | JSON 格式错误 |
| `1002` | 请求参数错误 |
| `1003` | WebSocket 消息类型错误 |
| `1004` | 未知 action |
| `9000` | 服务内部错误，例如 App 未初始化 |

### 数据结构

#### Detection

```json
{
  "score": 0.98,
  "box": [10, 20, 110, 160],
  "landmarks": [[35, 60], [85, 60], [60, 90], [42, 125], [82, 125]]
}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `score` | number | 人脸检测置信度 |
| `box` | array | 人脸框，格式为 `[minX, minY, maxX, maxY]` |
| `landmarks` | array | 关键点坐标，最多 5 个点 |

#### Face

```json
{
  "id": 1,
  "name": "alice",
  "image": "images/alice.jpg",
  "thumbnail": "images/alice.jpg"
}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | int64 | 人脸 ID |
| `name` | string | 人名 |
| `image` | string | 注册时保存的原始图片路径或 Data URL |
| `thumbnail` | string | 缩略图；当前实现直接复用 `image` |

#### FaceHit

```json
{
  "ID": 1,
  "Name": "alice",
  "Image": "images/alice.jpg",
  "Distance": 0.12,
  "ApproxCosine": 0.9928
}
```

字段说明：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `ID` | int64 | 命中的人脸 ID |
| `Name` | string | 命中的人名 |
| `Image` | string | 命中的图片路径或 Data URL |
| `Distance` | number | 欧氏距离，越小越相似 |
| `ApproxCosine` | number | 近似余弦相似度，越大越相似 |

#### IdentifyResult

```json
{
  "matched": true,
  "hit": {
    "ID": 1,
    "Name": "alice",
    "Image": "images/alice.jpg",
    "Distance": 0.12,
    "ApproxCosine": 0.9928
  }
}
```

说明：
- 如果当前帧没有任何候选命中，`matched=false`，`hit` 为空。
- 如果存在候选，但 `score_min > 0` 且 `ApproxCosine < score_min`，`hit` 仍然会返回，`matched=false`。
- 如果 `score_min <= 0`，只要存在候选就会判定为 `matched=true`。

## Action 列表

### ping

连通性测试。

请求：

```json
{
  "id": "1",
  "action": "ping"
}
```

成功响应：

```json
{
  "id": "1",
  "action": "ping",
  "code": 0,
  "msg": "ok",
  "data": {
    "message": "pong"
  }
}
```

### detect_face

检测图片中的人脸。

请求参数：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `image` | string | 是 | 图片路径或 Data URL |

请求：

```json
{
  "id": "2",
  "action": "detect_face",
  "payload": {
    "image": "images/face1.jpg"
  }
}
```

成功响应：

```json
{
  "id": "2",
  "action": "detect_face",
  "code": 0,
  "msg": "ok",
  "data": {
    "detections": [
      {
        "score": 0.98,
        "box": [10, 20, 110, 160],
        "landmarks": [[35, 60], [85, 60], [60, 90], [42, 125], [82, 125]]
      }
    ]
  }
}
```

### add_face

把图片中的最佳人脸加入人脸库。

请求参数：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | int64 | 是 | 人脸 ID，必须大于 0 |
| `name` | string | 是 | 人名 |
| `image` | string | 是 | 图片路径或 Data URL |

请求：

```json
{
  "id": "3",
  "action": "add_face",
  "payload": {
    "id": 1,
    "name": "alice",
    "image": "images/alice.jpg"
  }
}
```

成功响应：

```json
{
  "id": "3",
  "action": "add_face",
  "code": 0,
  "msg": "ok",
  "data": {
    "face": {
      "id": 1,
      "name": "alice",
      "image": "images/alice.jpg",
      "thumbnail": "images/alice.jpg"
    },
    "detection": {
      "score": 0.98,
      "box": [10, 20, 110, 160],
      "landmarks": [[35, 60], [85, 60], [60, 90], [42, 125], [82, 125]]
    }
  }
}
```

说明：服务端会先抽取特征，再按 `id` 保存；如果 `id` 已存在，会覆盖该记录。

### update_face

更新人脸库记录，可以只更新名称，也可以同时更新图片和特征。

请求参数：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | int64 | 是 | 人脸 ID，必须大于 0 |
| `name` | string | 条件必填 | 新人名；`name` 和 `image` 至少传一个 |
| `image` | string | 条件必填 | 新图片路径或 Data URL |

仅更新名称请求：

```json
{
  "id": "4",
  "action": "update_face",
  "payload": {
    "id": 1,
    "name": "alice-updated"
  }
}
```

更新图片请求：

```json
{
  "id": "5",
  "action": "update_face",
  "payload": {
    "id": 1,
    "name": "alice",
    "image": "images/alice-new.jpg"
  }
}
```

成功响应：

```json
{
  "id": "5",
  "action": "update_face",
  "code": 0,
  "msg": "ok",
  "data": {
    "affected": 1,
    "face": {
      "ID": 1,
      "Name": "alice",
      "Image": "images/alice-new.jpg"
    },
    "detection": {
      "score": 0.98,
      "box": [10, 20, 110, 160],
      "landmarks": [[35, 60], [85, 60], [60, 90], [42, 125], [82, 125]]
    }
  }
}
```

说明：
- 仅更新名称时，响应里不会返回 `detection`。
- `face` 字段名为 `ID`、`Name`、`Image`。

### search_face

使用单张图片搜索人脸库中的最近邻。

请求参数：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `image` | string | 是 | 图片路径或 Data URL |
| `topk` | int | 否 | 返回最近邻数量；小于 0 会报错，等于 0 时默认取 5 |

请求：

```json
{
  "id": "6",
  "action": "search_face",
  "payload": {
    "image": "images/query.jpg",
    "topk": 5
  }
}
```

成功响应：

```json
{
  "id": "6",
  "action": "search_face",
  "code": 0,
  "msg": "ok",
  "data": {
    "image": "images/query.jpg",
    "detection": {
      "score": 0.98,
      "box": [10, 20, 110, 160],
      "landmarks": [[35, 60], [85, 60], [60, 90], [42, 125], [82, 125]]
    },
    "hits": [
      {
        "ID": 1,
        "Name": "alice",
        "Image": "images/alice.jpg",
        "Distance": 0.12,
        "ApproxCosine": 0.9928
      }
    ]
  }
}
```

### identify_face

识别单张图片中的所有人脸，并为每张脸返回最近邻结果。

请求参数：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `image` | string | 是 | 图片路径或 Data URL |
| `score_min` | number | 否 | 匹配阈值；大于 0 时，只有 `ApproxCosine >= score_min` 才算 matched |

请求：

```json
{
  "id": "7",
  "action": "identify_face",
  "payload": {
    "image": "images/group.jpg",
    "score_min": 0.8
  }
}
```

成功响应：

```json
{
  "id": "7",
  "action": "identify_face",
  "code": 0,
  "msg": "ok",
  "data": {
    "image": "images/group.jpg",
    "faces": [
      {
        "detection": {
          "score": 0.98,
          "box": [10, 20, 110, 160],
          "landmarks": [[35, 60], [85, 60], [60, 90], [42, 125], [82, 125]]
        },
        "result": {
          "matched": true,
          "hit": {
            "ID": 1,
            "Name": "alice",
            "Image": "images/alice.jpg",
            "Distance": 0.12,
            "ApproxCosine": 0.9928
          }
        }
      }
    ]
  }
}
```

### identify_face_binary

通过 WebSocket 二进制帧上传图片，识别图片中的所有人脸，并为每张脸返回最近邻结果。

调用顺序：

1. 先发送文本 JSON，`action` 为 `identify_face_binary`。
2. 紧接着发送一帧二进制图片数据。
3. 服务端返回一帧文本 JSON 响应。

文本请求参数：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `format` | string | 否 | 图片格式；当前仅允许为空或 `jpeg` |
| `score_min` | number | 否 | 匹配阈值，规则同 `identify_face` |
| `width` | int | 否 | 当前服务端未使用 |
| `height` | int | 否 | 当前服务端未使用 |

第 1 帧文本消息：

```json
{
  "id": "8",
  "action": "identify_face_binary",
  "payload": {
    "format": "jpeg",
    "score_min": 0.8
  }
}
```

第 2 帧二进制消息：

```text
JPEG 图片原始字节
```

成功响应：

```json
{
  "id": "8",
  "action": "identify_face_binary",
  "code": 0,
  "msg": "ok",
  "data": {
    "faces": [
      {
        "detection": {
          "score": 0.98,
          "box": [10, 20, 110, 160],
          "landmarks": [[35, 60], [85, 60], [60, 90], [42, 125], [82, 125]]
        },
        "result": {
          "matched": true,
          "hit": {
            "ID": 1,
            "Name": "alice",
            "Image": "images/alice.jpg",
            "Distance": 0.12,
            "ApproxCosine": 0.9928
          }
        }
      }
    ]
  }
}
```

注意事项：
- 文本请求之后必须紧跟二进制消息，否则会返回 `1003` 或 `1002`。
- WebSocket 读取上限为 8 MiB，图片二进制消息不要超过该限制。
- `format` 非空时必须是 `jpeg`。

### get_face

根据 ID 获取人脸库记录。

请求参数：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | int64 | 是 | 人脸 ID，必须大于 0 |

请求：

```json
{
  "id": "9",
  "action": "get_face",
  "payload": {
    "id": 1
  }
}
```

成功响应：

```json
{
  "id": "9",
  "action": "get_face",
  "code": 0,
  "msg": "ok",
  "data": {
    "ID": 1,
    "Name": "alice",
    "Image": "images/alice.jpg"
  }
}
```


### list_faces

列出人脸库中的全部记录，按 ID 升序返回。

请求：

```json
{
  "id": "10",
  "action": "list_faces"
}
```

成功响应：

```json
{
  "id": "10",
  "action": "list_faces",
  "code": 0,
  "msg": "ok",
  "data": [
    {
      "id": 1,
      "name": "alice",
      "image": "images/alice.jpg",
      "thumbnail": "images/alice.jpg"
    }
  ]
}
```

### delete_face

根据 ID 删除人脸库记录。

请求参数：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `id` | int64 | 是 | 人脸 ID，必须大于 0 |

请求：

```json
{
  "id": "11",
  "action": "delete_face",
  "payload": {
    "id": 1
  }
}
```

成功响应：

```json
{
  "id": "11",
  "action": "delete_face",
  "code": 0,
  "msg": "ok",
  "data": {
    "id": 1,
    "affected": 1
  }
}
```

说明：`affected` 为实际删除的记录数，ID 不存在时为 `0`。

## 参数校验规则

| action | 校验规则 |
| --- | --- |
| `detect_face` | `image` 必填 |
| `add_face` | `id > 0`，`name` 必填，`image` 必填 |
| `update_face` | `id > 0`，`name` 和 `image` 至少传一个 |
| `search_face` | `image` 必填，`topk >= 0` |
| `identify_face` | `image` 必填 |
| `identify_face_binary` | `format` 为空或 `jpeg` |
| `get_face` | `id > 0` |
| `delete_face` | `id > 0` |

校验失败时示例：

```json
{
  "id": "bad-1",
  "action": "add_face",
  "code": 1002,
  "msg": "image is required",
  "data": {}
}
```

## JavaScript WebSocket 示例

```js
const ws = new WebSocket("ws://127.0.0.1:8080/ws");

ws.onopen = () => {
  ws.send(JSON.stringify({
    id: "demo-1",
    action: "list_faces"
  }));
};

ws.onmessage = (event) => {
  const resp = JSON.parse(event.data);
  console.log(resp);
};
```

二进制识别示例：

```js
const ws = new WebSocket("ws://127.0.0.1:8080/ws");
ws.binaryType = "arraybuffer";

ws.onopen = async () => {
  ws.send(JSON.stringify({
    id: "binary-1",
    action: "identify_face_binary",
    payload: {
      format: "jpeg",
      score_min: 0.8
    }
  }));

  const file = document.querySelector("input[type=file]").files[0];
  const bytes = await file.arrayBuffer();
  ws.send(bytes);
};

ws.onmessage = (event) => {
  console.log(JSON.parse(event.data));
};
```
