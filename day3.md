# MinIO 應用

昨天在部屬了MinIO的server以及試過了他的一些基礎功能後，今天就能試著透過用代碼的方式，來更好的使用MinIO所支援的功能。

## 場景發想

昨天沒提到的部分是MinIO能針對事件(物件的新增/刪除/存取)發出通知，同時我們也能對物件或bucket下標籤，來做使用上的管理，那這邊就透過建一個 web server來接收MinIO透過 webhook 的通知事件。

## MinIO 設定 Webhook + 綁定事件

首先要先透過 `mc` 設定一個 webhook 的目標，然後再重啟服務，讓配置生效。

```sh
# 這邊因為minio是透過podman架設的，podman的網卡跟下面用go在本機啟動web server的網卡不同，所以要改用本機網路的ip，不能用127.0.0.1
mc admin config set local notify_webhook:events1 endpoint="http://127.0.0.1:8080/minio-event"

mc admin service restart local
```

接著就是對指定 bucket 的綁定事件註冊。

```sh
# 綁定 ObjectCreated（Put/Post/Multipart 完成）事件到剛剛的 webhook 目標
mc event add local/videos arn:minio:sqs::events1:webhook --event put

# 檢查規則
mc event list local/videos
```

## 實作

這邊使用`go`啟一個web server來監聽MinIO會發送的事件。

```sh
go mod init minio-webhook
go mod tidy
go run .
```

```go
// main.go
package main

import (
	"encoding/json"
	"io"
	"log"
	"net/http"
	"os"
	"time"
)

// 簡化版的 S3/MinIO 事件結構，只取常用欄位。
// 若需要完整欄位，可依官方格式擴充。
type MinioEvent struct {
	EventName string `json:"EventName"`
	// MinIO 也常見 "Records" 陣列，保留兩種兼容
	Records []struct {
		EventName string    `json:"eventName"`
		EventTime time.Time `json:"eventTime"`
		S3        struct {
			Bucket struct {
				Name string `json:"name"`
			} `json:"bucket"`
			Object struct {
				Key  string `json:"key"`
				ETag string `json:"eTag"`
				Size int64  `json:"size"`
			} `json:"object"`
		} `json:"s3"`
		Source struct {
			Host      string `json:"host"`
			UserAgent string `json:"userAgent"`
		} `json:"source"`
	} `json:"Records"`
}

func main() {
	addr := getEnv("ADDR", ":8080")

	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ok"))
	})

	http.HandleFunc("/minio-event", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}

		body, err := io.ReadAll(r.Body)
		if err != nil {
			log.Printf("read body error: %v", err)
			http.Error(w, "bad request", http.StatusBadRequest)
			return
		}
		defer r.Body.Close()

		var evt MinioEvent
		if err := json.Unmarshal(body, &evt); err != nil {
			// 直接記 raw，方便排查
			log.Printf("json unmarshal error: %v, raw=%s", err, string(body))
			// 回 204，避免 MinIO 重試過多（也可回 200）
			w.WriteHeader(http.StatusNoContent)
			return
		}

		// 事件可能在 EventName 或 Records[0].eventName
		if evt.EventName != "" {
			log.Printf("[EventName] %s", evt.EventName)
		}

		for _, rec := range evt.Records {
			ename := rec.EventName
			// 常見：s3:ObjectCreated:Put / Post / CompleteMultipartUpload
			log.Printf("Event=%s Time=%s Bucket=%s Key=%s Size=%d UA=%s",
				ename, rec.EventTime.Format(time.RFC3339),
				rec.S3.Bucket.Name, rec.S3.Object.Key, rec.S3.Object.Size,
				rec.Source.UserAgent)
		}

		// 回 204/200 都行；204 表示不需要 response body
		w.WriteHeader(http.StatusNoContent)
	})

	log.Printf("listening on %s ...", addr)
	log.Fatal(http.ListenAndServe(addr, nil))
}

func getEnv(k, def string) string {
	if v := os.Getenv(k); v != "" {
		return v
	}
	return def
}
```

## 結果測試

這邊再透過後台UI或是`mc`上傳任意檔案上去，就能從web server中看到MinIO推過來的事件了。

```sh
2025/09/17 23:14:31 [EventName] s3:ObjectCreated:Put
2025/09/17 23:14:31 Event=s3:ObjectCreated:Put Time=2025-09-17T15:14:31Z Bucket=videos Key=1080P_1500K_224195221.mp4.mp4 Size=1011099394 UA=MinIO (linux; amd64) minio-go/v7.0.91 MinIO Console/(dev)
```
