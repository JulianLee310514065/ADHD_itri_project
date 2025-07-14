# Whisper 分散式語音轉文字系統

一個基於 Flask + Docker 的分散式語音轉文字系統，利用內網架構將計算任務分配到高性能GPU服務器上處理。支持多種音頻/視頻格式，自動轉換為繁體中文。

## 🏗️ 系統架構

```
Internet ──► 前端服務器 (140.96.170.75:8496) ──► 後端服務器 (192.168.50.120:8799)
             [有對外IP，無GPU]                    [高級GPU，內網]
             Whisper_frontend                     whisper_backend
```

### 架構說明

- **前端服務器 (Frontend)**：
  - 部署在有對外IP的服務器上 (140.96.170.75:8496)
  - 作為代理服務器，接收用戶請求並轉發到後端
  - 使用 Docker 容器化部署，提供 Web 界面
  - 無需GPU，主要負責請求處理和轉發

- **後端服務器 (Backend)**：
  - 部署在內網高性能GPU服務器上 (192.168.50.120:8799)
  - 執行實際的語音轉文字處理 (使用 OpenAI Whisper large-v3 模型)
  - 使用 Waitress WSGI 服務器提供穩定服務
  - 負責高計算量的AI推理工作，支持隊列處理

## 📁 項目結構

```
Whisper_fullend/
├── Whisper_frontend/           # 前端服務器
│   ├── Dockerfile             # Docker 構建文件
│   ├── docker-compose.yml     # Docker Compose 配置
│   ├── requirements.txt       # Python 依賴
│   ├── whisper_trans_server.py # 前端服務器主程序
│   └── logs/                  # 日誌目錄
└── whisper_backend/           # 後端服務器
    ├── flask_server.py        # 開發用服務器
    ├── waitress_server.py     # 生產用 Waitress 服務器
    └── test.ipynb            # 測試筆記本
```

## 🚀 安裝部署

### 後端服務器安裝 (GPU服務器)

1. **環境要求**：
   ```bash
   # Python 3.8+
   # CUDA 支持的 GPU (建議 8GB+ VRAM)
   # FFmpeg (用於視頻音頻處理)
   ```

2. **安裝依賴**：
   ```bash
   cd whisper_backend
   
   # 安裝 PyTorch (根據你的 CUDA 版本)
   pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
   
   # 安裝其他依賴
   pip install openai-whisper flask waitress opencc-python-reimplemented
   
   # 安裝 FFmpeg (Ubuntu/Debian)
   sudo apt update
   sudo apt install ffmpeg
   ```

3. **啟動後端服務**：
   ```bash
   python waitress_server.py
   ```
   
   服務將在 `http://192.168.50.120:8799` 啟動

### 前端服務器安裝 (對外IP服務器)

1. **Docker 部署 (推薦)**：
   ```bash
   cd Whisper_frontend
   
   # 使用 Docker Compose 啟動
   docker-compose up -d
   ```

2. **手動部署**：
   ```bash
   cd Whisper_frontend
   pip install -r requirements.txt
   python whisper_trans_server.py
   ```

## 🌐 網絡配置

### 內網設定

確保前端服務器可以訪問後端服務器：

```bash
# 在前端服務器測試連接
curl http://192.168.50.120:8799/health

# 修改後端URL（如果IP不同）
# 編輯 docker-compose.yml 中的 BACKEND_URL
environment:
  - BACKEND_URL=http://192.168.50.120:8799
```

### 防火牆設定

**後端服務器**：
```bash
# 開放端口 8799 給內網
sudo ufw allow from 192.168.0.0/16 to any port 8799
```

**前端服務器**：
```bash
# 開放端口 8496 對外
sudo ufw allow 8496
```

## 📚 API 使用方法

### Web 界面

訪問 `http://140.96.170.75:8496` 使用圖形界面上傳音頻/視頻文件

### API 調用

#### 1. 文件上傳轉錄 (支援格式：mp3, wav, ogg, flac, m4a, mp4)
```bash
curl -X POST http://140.96.170.75:8496/transcribe/file \
  -F "file=@your-audio.mp3" \
  -F "model=large-v3" \
  -F "timestamps=true"
```

#### 2. 二進制數據轉錄
```bash
curl -X POST "http://140.96.170.75:8496/transcribe/binary?model=large-v3&ext=wav" \
  --data-binary "@your-audio.wav" \
  -H "Content-Type: application/octet-stream"
```

#### 3. 獲取可用模型
```bash
curl http://140.96.170.75:8496/models
```

#### 4. Python 調用範例
```python
import requests

# 文件上傳轉錄
with open("audio.mp3", "rb") as f:
    files = {"file": f}
    data = {
        "model": "large-v3",
        "timestamps": "true"
    }
    response = requests.post(
        "http://140.96.170.75:8496/transcribe/file",
        files=files,
        data=data
    )

result = response.json()
if result["success"]:
    print(f"轉錄結果: {result['text']}")
    print(f"處理時間: {result['process_time']:.2f}秒")
    
    # 如果有時間軸數據
    if "segments" in result:
        for segment in result["segments"]:
            print(f"{segment['start']:.2f}s - {segment['end']:.2f}s: {segment['text']}")
else:
    print(f"錯誤: {result['error']}")
```

#### 5. JavaScript 調用範例
```javascript
// 文件上傳轉錄
async function transcribeAudio(file) {
    const formData = new FormData();
    formData.append('file', file);
    formData.append('model', 'large-v3');
    formData.append('timestamps', 'true');
    
    try {
        const response = await fetch('http://140.96.170.75:8496/transcribe/file', {
            method: 'POST',
            body: formData
        });
        
        const result = await response.json();
        
        if (result.success) {
            console.log('轉錄結果:', result.text);
            console.log('處理時間:', result.process_time + '秒');
            
            // 顯示時間軸數據
            if (result.segments) {
                result.segments.forEach(segment => {
                    console.log(`${segment.start.toFixed(2)}s - ${segment.end.toFixed(2)}s: ${segment.text}`);
                });
            }
        } else {
            console.error('轉錄失敗:', result.error);
        }
    } catch (error) {
        console.error('請求錯誤:', error);
    }
}

// 使用範例
document.getElementById('fileInput').addEventListener('change', function(e) {
    const file = e.target.files[0];
    if (file) {
        transcribeAudio(file);
    }
});
```

## ⚙️ 配置選項

### 後端配置

編輯 `whisper_backend/waitress_server.py` 中的設定：

```python
# 更改監聽端口
if __name__ == "__main__":
    serve(app, host="0.0.0.0", port=8799)

# 更改 GPU 設備
device = "cuda:1"  # 可改為 "cuda:0" 或其他GPU

# 更改默認模型
default_model_name = "large-v3"  # 可選: turbo, large-v3

# 支援的文件格式
ALLOWED_EXTENSIONS = {'mp3', 'wav', 'ogg', 'flac', 'm4a', 'mp4'}
```

### 前端配置

編輯 `Whisper_frontend/docker-compose.yml`：

```yaml
environment:
  - BACKEND_URL=http://192.168.50.120:8799  # 後端服務器地址
ports:
  - "8496:8499"  # 對外端口:內部端口
```

## 🎯 功能特色

### 支援格式
- **音頻格式**: MP3, WAV, OGG, FLAC, M4A
- **視頻格式**: MP4 (自動提取音軌)

### 模型支援
- **large-v3**: 高精度模型，適合重要場合
- **turbo**: 快速模型，適合即時轉錄

### 語言處理
- 自動識別中文語音
- 簡體轉繁體中文輸出
- 支援時間軸標記

### 隊列處理
- 支援並發請求隊列
- 智能任務排程
- GPU 資源保護

## 🔧 故障排除

### 常見問題

1. **前端無法連接後端**：
   ```bash
   # 檢查網絡連通性
   ping 192.168.50.120
   
   # 檢查端口開放
   telnet 192.168.50.120 8799
   
   # 檢查後端服務狀態
   curl http://192.168.50.120:8799/health
   ```

2. **GPU 記憶體不足**：
   ```bash
   # 檢查 GPU 狀態
   nvidia-smi
   
   # 重啟後端服務釋放記憶體
   pkill -f waitress_server.py
   python waitress_server.py
   ```

3. **音頻轉換失敗**：
   ```bash
   # 檢查 FFmpeg 安裝
   ffmpeg -version
   
   # 安裝 FFmpeg (如果未安裝)
   sudo apt install ffmpeg
   ```

4. **Docker 容器無法啟動**：
   ```bash
   # 檢查日誌
   docker-compose logs whisper-trans
   
   # 重建容器
   docker-compose down
   docker-compose up --build -d
   ```

### 日誌檢查

**前端日誌**：
```bash
# Docker 日誌
docker-compose logs -f whisper-trans

# 文件日誌
tail -f logs/app.log
```

**後端日誌**：
```bash
# 直接運行的後端會在終端顯示日誌
python waitress_server.py
```

## 📊 性能優化

### 後端優化

1. **GPU 記憶體管理**：
   - 使用適當的模型大小
   - 啟用 fp16 精度 (CUDA環境)
   - 實現請求隊列避免併發衝突

2. **模型優化**：
   - 預載常用模型到記憶體
   - 使用 turbo 模型提升速度
   - 考慮使用 whisper.cpp 進一步優化

### 前端優化

1. **連接池**：
   - 使用 requests.Session() 重用連接
   - 設置適當的超時時間

2. **緩存**：
   - 考慮添加 Redis 緩存轉錄結果
   - 實現文件哈希的結果緩存

## 🛡️ 安全注意事項

1. **文件上傳限制**：
   - 限制文件大小 (建議 100MB 以下)
   - 檢查文件類型和內容
   - 定期清理臨時文件

2. **網絡安全**：
   - 後端服務僅開放給內網
   - 前端服務配置適當的防火牆規則
   - 考慮添加 API 認證機制

## 📄 授權

本項目基於開源協議，請遵循相關組件的授權條款：
- [OpenAI Whisper](https://github.com/openai/whisper)
- [Flask](https://flask.palletsprojects.com/)
- [Docker](https://www.docker.com/)

## 🤝 貢獻

歡迎提交 Issue 和 Pull Request 來改進這個項目。

## 📞 聯絡

如有問題或建議，請聯繫項目維護者。 