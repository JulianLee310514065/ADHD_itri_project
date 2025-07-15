# CosyVoice 語音複製系統

基於 CosyVoice2 的分散式語音複製系統，採用前後端分離架構，支援高品質語音克隆和語音合成。


## 🏗️ 系統架構

### 架構圖
```
[用戶] → [前端代理伺服器] → [後端GPU伺服器] → [CosyVoice2模型]
          (對外IP)            (192.168.50.120:7500)
          (Docker部署)         (高級GPU運算)
```

### 組件說明

#### 🖥️ 後端伺服器 (`cosyvoice_backend`)
- **位置**: 192.168.50.120:7500
- **功能**: 實際執行語音合成和特徵提取
- **要求**: 高級 GPU (建議 NVIDIA RTX 系列)
- **運行**: 直接運行 Python 服務

#### 🌐 前端代理 (`cosyvoice_frontend`)
- **位置**: 對外可訪問的 IP:8492
- **功能**: 請求代理和轉發
- **要求**: 基本 CPU 運算即可
- **運行**: Docker 容器化部署

## ✨ 功能特性

- 🎤 **語音特徵提取**: 從音頻樣本中提取個人語音特徵
- 🔊 **語音合成**: 使用提取的特徵合成新語音
- 💾 **特徵保存/載入**: 持久化保存語音特徵，支援重複使用
- 🌐 **網頁界面**: 友好的 Web UI，無需編程知識
- 📡 **RESTful API**: 完整的 API 支援，便於集成
- 🐳 **Docker 支援**: 前端支援容器化部署
- ⚡ **高性能**: 基於 CosyVoice2 模型，生成速度快

## 💻 環境要求

### 後端伺服器要求
- **硬體**:
  - NVIDIA GPU (建議 RTX 4070 Ti或以上)
  - 記憶體: 16GB+
  - 儲存空間: 50GB+ (用於模型和生成檔案)
- **軟體**:
  - Python 3.8+
  - CUDA 11.8+
  - PyTorch 2.0+

### 前端伺服器要求
- **硬體**:
  - 基本 CPU 即可
  - 記憶體: 2GB+
  - 儲存空間: 5GB+
- **軟體**:
  - Docker & Docker Compose
  - 或 Python 3.8+ (直接運行)

## 🚀 安裝部署

### 步驟 1: 後端伺服器部署

1. **準備環境**
```bash
cd cosyvoice_backend
```

2. **安裝依賴**
```bash
pip install -r requirements_api.txt
```

3. **下載 CosyVoice2 模型**
```bash
# 創建模型目錄
mkdir -p pretrained_models/CosyVoice2-0.5B

# 下載模型檔案到該目錄 (具體下載方式請參考 CosyVoice 官方文檔)
# 或使用 ModelScope
python -c "from modelscope import snapshot_download; snapshot_download('iic/CosyVoice2-0.5B', local_dir='./pretrained_models/CosyVoice2-0.5B')"
```

4. **啟動後端服務**
```bash
python voice_cloning_api.py
```

服務將在 `http://192.168.50.120:7500` 啟動。

### 步驟 2: 前端代理部署

#### 方法 A: Docker 部署 (推薦)

1. **準備環境**
```bash
cd cosyvoice_frontend
```

2. **配置環境變數**
編輯 `docker-compose.yml` 確認後端 URL:
```yaml
environment:
  - BACKEND_URL=http://192.168.50.120:7500
```

3. **啟動服務**
```bash
docker-compose up -d
```

#### 方法 B: 直接運行

1. **安裝依賴**
```bash
cd cosyvoice_frontend
pip install -r requirements_frontend.txt
```

2. **啟動服務**
```bash
python voice_clone_frontend.py
```

前端服務將在 `http://[對外IP]:8492` 啟動。

## 📖 使用說明

### 網頁界面使用

1. **訪問系統**
   - 打開瀏覽器訪問前端服務地址: `http://[對外IP]:8492`

2. **提取語音特徵** (首次使用)
   - 上傳清晰的音頻檔案 (WAV 格式，10-30秒)
   - 輸入與音頻完全一致的文字內容
   - 設定音色名稱
   - 點擊「提取特徵」

3. **選擇音色**
   - 從下拉選單中選擇已保存的音色
   - 可試聽音色樣本

4. **生成語音**
   - 輸入要合成的文字內容
   - 選擇語音語言 (中文/英文)
   - 點擊「生成語音」
   - 下載生成的音頻檔案

### 程式化使用

```python
import requests

# 提取語音特徵
files = {'audio_file': open('sample.wav', 'rb')}
data = {
    'voice_name': 'my_voice',
    'reference_text': '這是參考文字'
}
response = requests.post('http://[前端IP]:8492/extract_voice', files=files, data=data)

# 生成語音
data = {
    'text': '你好，這是生成的語音',
    'voice_name': 'my_voice',
    'language': 'zh'
}
response = requests.post('http://[前端IP]:8492/generate_voice', data=data)
```

## 📚 API 文檔

### 基礎 URL
```
前端 API: http://[對外IP]:8492
後端 API: http://192.168.50.120:7500
```

### 主要端點

#### 1. 提取語音特徵
```http
POST /extract_voice
Content-Type: multipart/form-data

參數:
- voice_name: 音色名稱
- audio_file: WAV 音頻檔案
- reference_text: 參考文字

回應:
{
    "message": "音色特徵提取成功",
    "voice_file": "voice_name.pt",
    "sample_audio": "sample_voice_name.wav"
}
```

#### 2. 列出可用音色
```http
GET /list_voices

回應:
{
    "voices": ["voice1.pt", "voice2.pt"],
    "message": "獲取音色列表成功"
}
```

#### 3. 生成語音
```http
POST /generate_voice
Content-Type: application/json

參數:
{
    "text": "要合成的文字",
    "voice_name": "音色名稱",
    "language": "zh"  // zh 或 en
}

回應:
{
    "message": "語音生成成功",
    "audio_file": "generated_123456.wav"
}
```

#### 4. 下載檔案
```http
GET /download/<filename>
```

### 錯誤處理
```json
{
    "error": "錯誤描述",
    "details": "詳細錯誤信息"
}
```

## 🛠️ 故障排除

### 常見問題

#### Q1: 後端服務無法啟動
**A1**: 檢查 GPU 和 CUDA 安裝
```bash
nvidia-smi
python -c "import torch; print(torch.cuda.is_available())"
```

#### Q2: 前端無法連接後端
**A2**: 檢查網路連接和防火牆設定
```bash
# 測試連接
curl http://192.168.50.120:7500/
```

#### Q3: 語音生成失敗
**A3**: 檢查音頻格式和文字編碼
- 音頻必須是 WAV 格式，16kHz 採樣率
- 文字必須與音頻內容完全一致

#### Q4: 記憶體不足 (OOM)
**A4**: 調整批處理大小或使用更小的模型
```python
# 在 voice_cloning_api.py 中調整
cosyvoice = CosyVoice2('pretrained_models/CosyVoice2-0.5B', fp16=True)
```



## 🔗 相關連結

- [CosyVoice 官方項目](https://github.com/FunAudioLLM/CosyVoice)
- [ModelScope 模型庫](https://modelscope.cn/models/iic/CosyVoice2-0.5B)
- [Docker 官方文檔](https://docs.docker.com/)

## 📞 技術支援

如有問題請查看 [故障排除](#故障排除) 部分，或提交 Issue 描述詳細錯誤信息。

---

*最後更新: 2024年* 
