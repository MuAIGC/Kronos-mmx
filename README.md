<div align="center">
  <h2><b>Kronos-MMX: 金融K线基础模型 + WebUI</b></h2>
  <p>基于 <a href="https://github.com/shiyu-coder/Kronos">Kronos</a> 的增强分支，新增可视化Web界面与REST API</p>
</div>

<div align="center">
  <a href="https://github.com/shiyu-coder/Kronos">
    <img src="https://img.shields.io/badge/上游项目-Kronos-blue" alt="Upstream">
  </a>
  <a href="./LICENSE">
    <img src="https://img.shields.io/github/license/MuAIGC/Kronos-mmx?color=green" alt="License">
  </a>
  <a href="https://huggingface.co/NeoQuasar">
    <img src="https://img.shields.io/badge/🤗-Hugging_Face-yellow" alt="Hugging Face">
  </a>
</div>

---

## 关于本分支

本项目基于 [Kronos](https://github.com/shiyu-coder/Kronos)（AAAI 2026）进行增强，核心目标是**降低使用门槛**——让不懂代码的用户也能通过浏览器或API直接使用金融K线预测模型。

### 相比原版新增的功能

| 功能 | 说明 |
|------|------|
| **WebUI 可视化界面** | Flask 前后端一体化，浏览器打开即用 |
| **中英文切换** | 金融术语专业汉化（开盘/收盘/K线/回看窗口等） |
| **REST API** | 完整的 HTTP 接口，支持程序化调用 |
| **文件上传** | 支持通过界面或API上传 CSV/Feather 数据文件 |
| **公告栏** | 顶部公告栏，标注项目来源 |
| **步骤引导** | 三步操作：加载模型 → 加载数据 → 开始预测 |
| **本地模型支持** | 模型路径指向本地目录，不依赖 HuggingFace 缓存 |
| **systemd 服务** | 开机自启，进程守护 |

---

## 模型列表

| 模型 | 分词器 | 上下文长度 | 参数量 |
|------|--------|-----------|--------|
| Kronos-mini | Kronos-Tokenizer-2k | 2048 | 4.1M |
| Kronos-small | Kronos-Tokenizer-base | 512 | 24.7M |
| Kronos-base | Kronos-Tokenizer-base | 512 | 102.3M |
| Kronos-large | Kronos-Tokenizer-base | 512 | 499.2M（未开源） |

模型权重下载：[HuggingFace - NeoQuasar](https://huggingface.co/NeoQuasar)

---

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
cd webui && pip install -r requirements.txt
```

### 2. 下载模型

```python
from huggingface_hub import snapshot_download
import os

os.environ["HF_ENDPOINT"] = "https://hf-mirror.com"  # 国内镜像

for repo in ["NeoQuasar/Kronos-mini", "NeoQuasar/Kronos-small", "NeoQuasar/Kronos-base",
             "NeoQuasar/Kronos-Tokenizer-2k", "NeoQuasar/Kronos-Tokenizer-base"]:
    snapshot_download(repo_id=repo, local_dir=f"./model/{repo.split(/)[-1]}")
```

### 3. 启动 WebUI

```bash
cd webui
python app.py
# 访问 http://localhost:7861
```

---

## WebUI 使用

浏览器打开后，按三步操作：

1. **第一步：加载模型** — 选择模型（默认 Kronos-base）和设备（默认 CUDA），点击加载
2. **第二步：加载数据** — 从下拉列表选择数据文件，或上传自己的 CSV/Feather 文件
3. **第三步：开始预测** — 调整参数后点击预测，查看K线图表

支持中文/英文切换（右上角按钮），金融术语专业翻译。

---

## API 接口

WebUI 同时提供 REST API，可直接通过 HTTP 调用。

> 完整文档见 [`API.md`](./API.md)

### 接口总览

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/health` | GET | 健康检查 |
| `/api/available-models` | GET | 获取可用模型列表 |
| `/api/model-status` | GET | 查看模型加载状态 |
| `/api/load-model` | POST | 加载模型 |
| `/api/data-files` | GET | 获取数据文件列表 |
| `/api/upload-data` | POST | 上传数据文件 |
| `/api/load-data` | POST | 加载数据信息 |
| `/api/predict` | POST | 执行预测 |

### 快速示例

```bash
# 加载模型
curl -X POST http://localhost:7861/api/load-model \
  -H "Content-Type: application/json" \
  -d {model_key: kronos-base, device: cuda}

# 上传数据
curl -X POST http://localhost:7861/api/upload-data -F "file=@data.csv"

# 执行预测
curl -X POST http://localhost:7861/api/predict \
  -H "Content-Type: application/json" \
  -d {file_path: /path/to/data.csv, lookback: 400, pred_len: 120}
```

### Python 调用

```python
import requests

BASE_URL = "http://localhost:7861"

# 加载模型
requests.post(f"{BASE_URL}/api/load-model", json={"model_key": "kronos-base", "device": "cuda"})

# 上传数据
with open("data.csv", "rb") as f:
    r = requests.post(f"{BASE_URL}/api/upload-data", files={"file": f})
file_path = r.json()["file"]["path"]

# 预测
r = requests.post(f"{BASE_URL}/api/predict", json={
    "file_path": file_path,
    "lookback": 400,
    "pred_len": 120,
    "temperature": 1.0,
    "top_p": 0.9,
    "sample_count": 1
})
print(r.json())
```

---

## SDK 调用（无需WebUI）

```python
from model import Kronos, KronosTokenizer, KronosPredictor
import pandas as pd

tokenizer = KronosTokenizer.from_pretrained("./model/Kronos-Tokenizer-base")
model = Kronos.from_pretrained("./model/Kronos-base")
predictor = KronosPredictor(model, tokenizer, max_context=512)

df = pd.read_csv("./data/XSHG_5min_600977.csv")
df["timestamps"] = pd.to_datetime(df["timestamps"])

lookback, pred_len = 400, 120
x_df = df.loc[:lookback-1, ["open", "high", "low", "close", "volume", "amount"]]
x_timestamp = df.loc[:lookback-1, "timestamps"]
y_timestamp = df.loc[lookback:lookback+pred_len-1, "timestamps"]

pred_df = predictor.predict(
    df=x_df, x_timestamp=x_timestamp, y_timestamp=y_timestamp,
    pred_len=pred_len, T=1.0, top_p=0.9, sample_count=1
)
print(pred_df.head())
```

---

## 目录结构

```
Kronos/
├── model/                      # 模型代码 + 预训练权重
│   ├── kronos.py               # 模型定义
│   ├── module.py               # 模块组件
│   ├── Kronos-mini/            # 4.1M 参数
│   ├── Kronos-small/           # 24.7M 参数
│   ├── Kronos-base/            # 102.3M 参数
│   ├── Kronos-Tokenizer-2k/    # mini 专用分词器
│   └── Kronos-Tokenizer-base/  # small/base 通用分词器
├── webui/                      # WebUI（Flask 前后端一体）
│   ├── app.py                  # 主服务（含 REST API）
│   ├── templates/index.html    # 前端页面
│   └── requirements.txt        # WebUI 依赖
├── API.md                      # API 接口文档
├── finetune/                   # 微调脚本（Qlib A股示例）
├── finetune_csv/               # CSV 数据微调
├── examples/                   # 预测示例
├── data/                       # 样本数据
└── tests/                      # 测试
```

---

## 服务部署（systemd）

```bash
# 创建服务文件
cat > /etc/systemd/system/kronos-webui.service << EOF
[Unit]
Description=Kronos WebUI Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 app.py
WorkingDirectory=/path/to/Kronos/webui
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 启用并启动
systemctl daemon-reload
systemctl enable --now kronos-webui
```

---

## 数据格式要求

CSV 文件必须包含以下列：

| 列名 | 必填 | 说明 |
|------|------|------|
| `open` | 是 | 开盘价 |
| `high` | 是 | 最高价 |
| `low` | 是 | 最低价 |
| `close` | 是 | 收盘价 |
| `volume` | 否 | 成交量 |
| `amount` | 否 | 成交额 |
| `timestamps` / `timestamp` / `date` | 否 | 时间戳 |

---

## 致谢

本项目基于以下优秀工作：

- **[Kronos](https://github.com/shiyu-coder/Kronos)** — 首个开源金融K线基础模型（AAAI 2026）
- 作者：Yu Shi, Zongliang Fu, Shuo Chen, Bohan Zhao, Wei Xu, Changshui Zhang, Jian Li
- 论文：[arXiv:2508.02739](https://arxiv.org/abs/2508.02739)

## 引用

```bibtex
@misc{shi2025kronos,
    title={Kronos: A Foundation Model for the Language of Financial Markets},
    author={Yu Shi and Zongliang Fu and Shuo Chen and Bohan Zhao and Wei Xu and Changshui Zhang and Jian Li},
    year={2025},
    eprint={2508.02739},
    archivePrefix={arXiv},
    primaryClass={q-fin.ST},
    url={https://arxiv.org/abs/2508.02739},
}
```

## License

[MIT](./LICENSE)
