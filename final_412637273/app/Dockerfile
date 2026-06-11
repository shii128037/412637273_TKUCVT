# ~/final/app/Dockerfile
FROM python:3.12-slim

WORKDIR /app

# 1. 先複製依賴宣告（極少變動，快取關鍵）
COPY requirements.txt .

# 2. 安裝依賴（--no-cache-dir 縮小鏡像體積）
RUN pip install --no-cache-dir -r requirements.txt

# 3. 安全加固：建立非 root 使用者與群組
RUN groupadd -g 1000 appuser && \
    useradd -r -u 1000 -g appuser appuser

# 4. 複製常變動的原始碼（放最後面，避免破壞前面的快取）
COPY app.py .

# 5. 切換成非 root 使用者執行
USER appuser

EXPOSE 8080

# exec form 啟動
CMD ["python", "app.py"]
