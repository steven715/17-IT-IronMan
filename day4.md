# ZeroMq 體驗

今天來體驗 ZeroMq ，這是一個高性能非同步信息庫，主要就是想試試其中高性能的部分，以及作為一個信息佇列，也能透過不同語言來實作。

## ZeroMq 使用

今天就先用go來使用，因為最近用下來，真的覺得go相對簡潔很多。

- 先建個專案

```sh
mkdir go-zeromq-demo
cd go-zeromq-demo
go mod init example.com/go-zeromq-demo

# 安裝 ZeroMQ 的 Go 套件
go get github.com/go-zeromq/zmq4
```

- Server實作:

```go
// server.go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/go-zeromq/zmq4"
)

func main() {
	// 建立 REP socket
	server := zmq4.NewRep(context.Background())
	defer server.Close()

	// 綁定在 TCP 5555 port
	if err := server.Listen("tcp://*:5555"); err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	fmt.Println("Server listening on tcp://*:5555")

	for {
		// 收 client 訊息
		msg, err := server.Recv()
		if err != nil {
			log.Fatalf("recv error: %v", err)
		}
		fmt.Println("Server got:", string(msg.Bytes()))

		// 回覆訊息
		reply := fmt.Sprintf("Echo: %s", string(msg.Bytes()))
		_ = server.Send(zmq4.NewMsgString(reply))
		time.Sleep(100 * time.Millisecond) // 模擬處理時間
	}
}

```

- Client實作:

```go
// client.go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/go-zeromq/zmq4"
)

func main() {
	// 建立 REQ socket
	client := zmq4.NewReq(context.Background())
	defer client.Close()

	// 連線到 server
	if err := client.Dial("tcp://127.0.0.1:5555"); err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	fmt.Println("Client connected to tcp://127.0.0.1:5555")

	// 發送文字訊息
	msg := "Hello from client"
	_ = client.Send(zmq4.NewMsgString(msg))
	fmt.Println("Client sent:", msg)

	// 等待回覆
	reply, err := client.Recv()
	if err != nil {
		log.Fatalf("recv error: %v", err)
	}
	fmt.Println("Client got:", string(reply.Bytes()))
}

```

- 編譯與運行

```sh
go build -o server.exe server.go
go build -o client.exe client.go

# Server端先運行
.\server.exe
Server listening on tcp://*:5555
Server got: Hello from client

# Client端再運行
.\client.exe
Client connected to tcp://127.0.0.1:5555
Client sent: Hello from client
Client got: Echo: Hello from client
```

## 結論

今天先簡單透過go體驗一下zeroMq的基礎，明日再從一些使用場景上發想看如何針對高性能做一些探討。
