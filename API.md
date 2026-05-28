# Kronos WebUI API 接口文档

## 访问地址

- **内部地址**: `http://localhost:7861` 或 `http://<目标机IP>:7861`
- **外部地址**: 由云商平台动态分配，格式如 `https://<实例名>-7861.nm2.spacehpc.com:3391`

> 外部 URL 每个实例不同，请在云商平台查看你的实例对应的地址。

---

## 健康检查

```bash
curl <YOUR_URL>/api/health
```

返回：
```json
{
  "model_available": true,
  "model_loaded": false,
  "status": "ok",
  "version": "1.0.0"
}
```

---

## 使用流程

按顺序调用：加载模型 → 加载/上传数据 → 执行预测

---

## 1. 获取可用模型列表

```bash
curl <YOUR_URL>/api/available-models
```

## 2. 查看模型状态

```bash
curl <YOUR_URL>/api/model-status
```

## 3. 加载模型

```bash
curl -X POST <YOUR_URL>/api/load-model -H "Content-Type: application/json" -d "{\"model_key\": \"kronos-base\", \"device\": \"cuda\"}"
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| model_key | 字符串 | 是 | 可选值：kronos-mini / kronos-small / kronos-base |
| device | 字符串 | 是 | 可选值：cpu / cuda / mps |

## 4. 获取数据文件列表

```bash
curl <YOUR_URL>/api/data-files
```

## 5. 上传数据文件

```bash
curl -X POST <YOUR_URL>/api/upload-data -F "file=@你的文件.csv"
```

- 支持格式：`.csv`、`.feather`
- CSV 必须包含列：`open`, `high`, `low`, `close`
- 可选列：`volume`, `amount`, `timestamps`/`timestamp`/`date`
- 返回文件路径，用于后续预测

## 6. 加载数据（获取数据信息）

```bash
curl -X POST <YOUR_URL>/api/load-data -H "Content-Type: application/json" -d "{\"file_path\": \"/MMXTools/Kronos/data/XSHG_5min_000001.csv\"}"
```

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| file_path | 字符串 | 是 | 数据文件路径 |

## 7. 执行预测

**最简调用（使用默认参数）：**

```bash
curl -X POST <YOUR_URL>/api/predict -H "Content-Type: application/json" -d "{\"file_path\": \"/MMXTools/Kronos/data/XSHG_5min_000001.csv\"}"
```

**完整参数调用：**

```bash
curl -X POST <YOUR_URL>/api/predict -H "Content-Type: application/json" -d "{\"file_path\": \"/MMXTools/Kronos/data/XSHG_5min_000001.csv\", \"lookback\": 400, \"pred_len\": 120, \"temperature\": 1.0, \"top_p\": 0.9, \"sample_count\": 1}"
```

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| file_path | 字符串 | 是 | - | 数据文件路径 |
| lookback | 数字 | 否 | 400 | 回看窗口长度 |
| pred_len | 数字 | 否 | 120 | 预测长度 |
| temperature | 浮点数 | 否 | 1.0 | 预测温度，0.1-2.0，越高越多样 |
| top_p | 浮点数 | 否 | 0.9 | 核采样参数，0.1-1.0，越高越多样 |
| sample_count | 数字 | 否 | 1 | 采样次数，1-5 |
| start_date | 字符串 | 否 | - | 起始时间，ISO格式如 2024-01-01T00:00 |

**返回结果说明：**

| 字段 | 说明 |
|------|------|
| success | 是否成功 |
| chart | Plotly 图表 JSON 数据，可直接渲染 |
| prediction_results | 预测的 OHLCV 数据数组 |
| actual_data | 实际数据（用于对比） |
| has_comparison | 是否有对比数据 |
| prediction_type | 预测类型描述 |
| message | 结果消息 |

---

## Python 调用示例

```python
import requests

BASE_URL = "<YOUR_URL>"  # 替换为你的外部或内部地址

# 1. 加载模型
r = requests.post(f"{BASE_URL}/api/load-model", json={
    "model_key": "kronos-base",
    "device": "cuda"
})
print(r.json())

# 2. 上传数据
with open("data.csv", "rb") as f:
    r = requests.post(f"{BASE_URL}/api/upload-data", files={"file": f})
print(r.json())
file_path = r.json()["file"]["path"]

# 3. 执行预测
r = requests.post(f"{BASE_URL}/api/predict", json={
    "file_path": file_path,
    "lookback": 400,
    "pred_len": 120,
    "temperature": 1.0,
    "top_p": 0.9,
    "sample_count": 1
})
result = r.json()
print(result["message"])
print(f"预测数据点数: {len(result[prediction_results])}")
```

---

## 注意事项

- 首次使用必须先加载模型，再操作数据
- 外部 URL 每个实例不同，由云商平台分配
- 文件上传大小限制取决于云商平台配置
- 预测结果中的 chart 字段是 Plotly JSON，可直接用 Plotly.js 渲染
