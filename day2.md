# MinIO 體驗

今天來體驗一下MinIO，很久以前在前公司有使用過這個軟體，當初是做類似社群軟體，用來存放使用者上傳的照片，前一陣子想自己做個影片網站，就想說可以使用MinIO作為影片的儲存空間，所以今天來重新回味一下MinIO。

## MinIO 介紹

MinIO 是一套 高效能、S3 相容的物件儲存的開源專案。

## MinIO 安裝

安裝這邊就參考[官網提供的方式](https://docs.min.io/enterprise/aistor-object-store/installation/container/install/?_gl=1*72kgoo*_gcl_au*MTQ0NjY1MDM2Mi4xNzU4MDM0MjI3*_ga*ODAzNDYzMDEuMTc1ODAzNDIyMw..*_ga_SY0M3KFH50*czE3NTgwMzQyMjIkbzEkZzEkdDE3NTgwMzQ4MzEkajU5JGwwJGg3NDA1NjgwMw..&tab=download-image-docker)，使用docker的方式來安裝，不過官網已經換成[podman](https://podman.io/docs/installation)來安裝了，所以這邊也就跟隨官網也改用[podman](https://podman.io/docs/installation)來安裝，簡單來說podman算是改良版的docker，所以也支援docker的語法。

```sh
# minio server
podman pull quay.io/minio/aistor/minio:latest

# 建一個單節點的minio server，以及一個管理後台 http://127.0.0.1:9001/browser
podman run -p 9000:9000 -p 9001:9001 -e MINIO_ROOT_USER=minioadmin -e MINIO_ROOT_PASSWORD=minioadmin -v D:/minio/data:/data --name minio quay.io/minio/minio:latest server /data --console-address ":9001"
```

## MinIO 使用

透過前面指令啟動的MinIO Server還自帶一個[管理後臺](http://127.0.0.1:9001/browser)，這時候就可以透過管理後台來上傳影片了，那除了以UI的方式操作外，MinIO也提供`CLI`的方式來操作。

```sh
# powershell
Invoke-WebRequest https://dl.min.io/client/mc/release/windows-amd64/mc.exe -OutFile "C:\Windows\System32\mc.exe"

# 透過以下指令確認是否安裝成功
mc --version

# 安裝 mc (minio client) 後，加一個別名
mc alias set local http://127.0.0.1:9000 minioadmin minioadmin

# 建 bucket
mc mb local/tests

# 上傳單個檔案
mc cp D:\aaa.txt local/tests

# 上傳整個資料夾
mc cp --recursive D:\bbb local/tests

# 列出 bucket
mc ls local

# 顯示檔案詳細資訊
mc stat local/tests/aaa.txt

# 下載單一檔案到目前資料夾
mc cp local/tests/aaa.txt .

# 下載整個 bucket
mc cp --recursive local/bbb D:\downloaded

# 刪除單檔
mc rm local/tests/aaa.txt

# 刪除整個 bucket（需加 --recursive）
mc rm --recursive --force local/tests

# 刪除 bucket，前提是bucket 為空
mc rb local/tests
```

## 明天繼續

明天接著介紹如何透過SDK來擴充MinIO的使用場景。
