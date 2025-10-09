# 網路協議 tcp 體驗

因為做工作上有使用到 tcp socket 作為一個自定義的傳輸協議，能做到多路分發消息，就對這樣的應用很感興趣，所以會想透過找些相關應用來多體驗這塊。

## 今日目標

使用 go 語言來實作 tcp socket 的Server跟Client，讓Server能接收Client的訊息。

## TCP Server

```go
// server.go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
)

func main() {
	listener, err := net.Listen("tcp", ":9000")
	if err != nil {
		fmt.Println("Error starting server:", err)
		return
	}
	defer listener.Close()
	fmt.Println("✅ TCP Server listening on :9000")

	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("Error accepting:", err)
			continue
		}
		fmt.Println("🔗 New client connected:", conn.RemoteAddr())

		// 啟動 goroutine 處理每個連線
		go handleConnection(conn)
	}
}

func handleConnection(conn net.Conn) {
	defer conn.Close()
	reader := bufio.NewReader(conn)

	for {
		msg, err := reader.ReadString('\n')
		if err != nil {
			fmt.Println("❌ Client disconnected:", conn.RemoteAddr())
			return
		}

		fmt.Printf("📩 Received from %v: %s", conn.RemoteAddr(), msg)

		// 回傳訊息給 client
        reply := fmt.Sprintf("Echo from server: %s", msg)
		conn.Write([]byte(reply))
	}
}

```

## TCP Client

```go
// client.go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
)

func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:9000")
	if err != nil {
		fmt.Println("Error connecting:", err)
		return
	}
	defer conn.Close()

	fmt.Println("✅ Connected to server. Type messages and press Enter.")
	reader := bufio.NewReader(os.Stdin)

	for {
		fmt.Print("You: ")
		text, _ := reader.ReadString('\n')
		_, err := conn.Write([]byte(text))
		if err != nil {
			fmt.Println("Write error:", err)
			return
		}

		// 等待回應
		reply := make([]byte, 1024)
		n, err := conn.Read(reply)
		if err != nil {
			fmt.Println("Read error:", err)
			return
		}
		fmt.Printf("💬 Server reply: %s", string(reply[:n]))
	}
}

```

## Run

```sh
# 終端 A：啟動 server
go run server.go

# 終端 B：啟動 client
go run client.go
```

```sh
# 終端 A： server
✅ TCP Server listening on :9000
🔗 New client connected: 127.0.0.1:50465
📩 Received from 127.0.0.1:50465: hi
🔗 New client connected: 127.0.0.1:52590
📩 Received from 127.0.0.1:52590: ha

# 終端 B： client
✅ Connected to server. Type messages and press Enter.
You: hi  
💬 Server reply: Echo from server: hi
```

## 結論

今天就先以這個簡單的小程序，並使用go語言來做開頭，明天再繼續看看還有哪些功能可以擴展的。
