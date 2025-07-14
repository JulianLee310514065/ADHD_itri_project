# Rembg 分散式圖片去背系統

一個基於 Flask + Docker 的分散式圖片背景去除系統，利用內網架構將計算任務分配到高性能GPU服務器上處理。

## 🏗️ 系統架構

```
Internet ──► 前端服務器 (140.96.170.75:8495) ──► 後端服務器 (192.168.50.120:7000)
             [有對外IP，無GPU]                    [高級GPU，內網]
             rembg_frontend                       rembg_backend
```

### 架構說明

- **前端服務器 (Frontend)**：
  - 部署在有對外IP的服務器上 (140.96.170.75:8495)
  - 作為代理服務器，接收用戶請求並轉發到後端
  - 使用 Docker 容器化部署，提供 Web 界面
  - 無需GPU，主要負責請求處理和轉發

- **後端服務器 (Backend)**：
  - 部署在內網高性能GPU服務器上 (192.168.50.120:7000)
  - 執行實際的圖片去背處理 (使用 rembg + birefnet-general 模型)
  - 使用 Waitress WSGI 服務器提供穩定服務
  - 負責高計算量的AI推理工作

## 📁 項目結構

```
rembg_fullend/
├── rembg_frontend/           # 前端服務器
│   ├── Dockerfile           # Docker 構建文件
│   ├── docker-compose.yml   # Docker Compose 配置
│   ├── requirements.txt     # Python 依賴
│   ├── rembg_waitress_server.py  # 前端服務器主程序
│   └── logs/               # 日誌目錄
└── rembg_backend/           # 後端服務器
    ├── server.py           # 開發用服務器
    ├── server_waitress.py  # 生產用 Waitress 服務器
    └── server.ipynb        # 測試筆記本
```

## 🚀 安裝部署

### 後端服務器安裝 (GPU服務器)

1. **環境要求**：
   ```bash
   # Python 3.8+
   # CUDA 支持的 GPU
   # 充足的 GPU 記憶體 (建議 4GB+)
   ```

2. **安裝依賴**：
   ```bash
   cd rembg_backend
   pip install flask rembg waitress pillow requests urllib3
   ```

3. **啟動後端服務**：
   ```bash
   python server_waitress.py
   ```
   
   服務將在 `http://192.168.50.120:7000` 啟動

### 前端服務器安裝 (對外IP服務器)

1. **Docker 部署 (推薦)**：
   ```bash
   cd rembg_frontend
   
   # 使用 Docker Compose 啟動
   docker-compose up -d
   ```

2. **手動部署**：
   ```bash
   cd rembg_frontend
   pip install -r requirements.txt
   python rembg_waitress_server.py
   ```

## 🌐 網絡配置

### 內網設定

確保前端服務器可以訪問後端服務器：

```bash
# 在前端服務器測試連接
curl http://192.168.50.120:7000/

# 修改後端URL（如果IP不同）
# 編輯 docker-compose.yml 中的 BACKEND_URL
environment:
  - BACKEND_URL=http://192.168.50.120:7000
```

### 防火牆設定

**後端服務器**：
```bash
# 開放端口 7000 給內網
sudo ufw allow from 192.168.0.0/16 to any port 7000
```

**前端服務器**：
```bash
# 開放端口 8495 對外
sudo ufw allow 8495
```

## 📚 API 使用方法

### Web 界面

訪問 `http://140.96.170.75:8495` 使用圖形界面

### API 調用

#### 1. POST 請求
```bash
curl -X POST http://140.96.170.75:8495/api/remove \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/your-image.jpg"}' \
  --output removed_bg.png
```

#### 2. GET 請求
```bash
curl "http://140.96.170.75:8495/api/remove?url=https://example.com/your-image.jpg" \
  --output removed_bg.png
```

#### 3. Python 調用範例
```python
import requests

# 使用 POST
response = requests.post(
    "http://140.96.170.75:8495/api/remove",
    json={"url": "https://example.com/your-image.jpg"}
)

# 保存結果
if response.status_code == 200:
    with open("removed_bg.png", "wb") as f:
        f.write(response.content)
    print("去背完成！")
else:
    print(f"錯誤: {response.text}")
```

#### 4. JavaScript 調用範例
```javascript
// 使用 fetch API
async function removeBackground(imageUrl) {
    try {
        const response = await fetch('http://140.96.170.75:8495/api/remove', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
            },
            body: JSON.stringify({ url: imageUrl })
        });
        
        if (response.ok) {
            const blob = await response.blob();
            const url = window.URL.createObjectURL(blob);
            
            // 創建下載鏈接
            const a = document.createElement('a');
            a.href = url;
            a.download = 'removed_bg.png';
            a.click();
        } else {
            console.error('去背失敗:', await response.text());
        }
    } catch (error) {
        console.error('請求錯誤:', error);
    }
}
```

## ⚙️ 配置選項

### 後端配置

編輯 `rembg_backend/server_waitress.py` 中的設定：

```python
# 更改監聽端口
if __name__ == "__main__":
    serve(app, host="0.0.0.0", port=7000)

# 更改 rembg 模型
session = new_session("birefnet-general")  # 可改為其他模型
```

### 前端配置

編輯 `rembg_frontend/docker-compose.yml`：

```yaml
environment:
  - BACKEND_URL=http://192.168.50.120:7000  # 後端服務器地址
ports:
  - "8495:8498"  # 對外端口:內部端口
```

## 🔧 故障排除

### 常見問題

1. **前端無法連接後端**：
   ```bash
   # 檢查網絡連通性
   ping 192.168.50.120
   
   # 檢查端口開放
   telnet 192.168.50.120 7000
   
   # 檢查後端服務狀態
   curl http://192.168.50.120:7000/
   ```

2. **GPU 記憶體不足**：
   ```bash
   # 檢查 GPU 狀態
   nvidia-smi
   
   # 重啟後端服務釋放記憶體
   pkill -f server_waitress.py
   python server_waitress.py
   ```

3. **Docker 容器無法啟動**：
   ```bash
   # 檢查日誌
   docker-compose logs rembg-server
   
   # 重建容器
   docker-compose down
   docker-compose up --build -d
   ```

### 日誌檢查

**前端日誌**：
```bash
# Docker 日誌
docker-compose logs -f rembg-server

# 文件日誌
tail -f logs/app.log
```

**後端日誌**：
```bash
# 直接運行的後端會在終端顯示日誌
python server_waitress.py
```

## 📊 性能優化

### 後端優化

1. **GPU 記憶體管理**：
   - 使用適當的批次大小
   - 定期清理 GPU 記憶體

2. **模型優化**：
   - 選擇適合的 rembg 模型
   - 考慮使用量化模型

### 前端優化

1. **連接池**：
   - 使用 requests.Session() 重用連接
   - 設置適當的超時時間

2. **緩存**：
   - 考慮添加 Redis 緩存常用結果
   - 實現圖片URL的結果緩存

## 📄 授權

本項目基於開源協議，請遵循相關組件的授權條款：
- [rembg](https://github.com/danielgatis/rembg)
- [Flask](https://flask.palletsprojects.com/)
- [Docker](https://www.docker.com/)

## 🤝 貢獻

歡迎提交 Issue 和 Pull Request 來改進這個項目。

## 📞 聯絡

如有問題或建議，請聯繫項目維護者。 