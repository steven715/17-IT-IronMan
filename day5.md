# ZeroMq 挖掘

今天想接續昨天作的，來試試看換成對吞吐量的表現如何。

## 吞吐量測試

- Server端

```go
// tput_server.go
package main

import (
	"context"
	"errors"
	"flag"
	"fmt"
	"io"
	"log"
	"time"

	"github.com/go-zeromq/zmq4"
)

func main() {
	bind := flag.String("bind", "tcp://*:5556", "bind address (PULL)")
	flag.Parse()

	sock := zmq4.NewPull(context.Background())
	defer sock.Close()
	if err := sock.Listen(*bind); err != nil {
		log.Fatalf("listen: %v", err)
	}
	fmt.Println("tput server listening on", *bind)

	var (
		total   int64
		window  int64
		start   = time.Now()
		lastLog = time.Now()
	)
	t := time.NewTicker(time.Second)
	defer t.Stop()
	go func() {
		for range t.C {
			el := time.Since(lastLog).Seconds()
			mps := float64(window) / el
			fmt.Printf("[%.0fs] window=%d msgs  rate=%.0f msgs/s  total=%d\n",
				time.Since(start).Seconds(), window, mps, total)
			window = 0
			lastLog = time.Now()
		}
	}()

	for {
		msg, err := sock.Recv()
		if err != nil {
			if errors.Is(err, io.EOF) {
				// 所有現有連線都關了；繼續等新的連線/訊息（短暫歇一下防止忙等）
				time.Sleep(10 * time.Millisecond)
				continue
			}
			// 其它錯誤才結束
			log.Fatalf("recv: %v", err)
		}
		_ = msg
		total++
		window++
	}
}

```

- Client端

```go
// tput_client.go
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"math"
	"sync"
	"time"

	"github.com/go-zeromq/zmq4"
)

func main() {
	connect := flag.String("connect", "tcp://127.0.0.1:5556", "sink address")
	count := flag.Int("count", 1_000_000, "total messages to send")
	size := flag.Int("size", 100, "payload size (bytes)")
	concurrency := flag.Int("concurrency", 4, "parallel senders")
	flag.Parse()

	payload := make([]byte, *size)
	for i := range payload {
		payload[i] = byte(i)
	}

	var wg sync.WaitGroup
	per := int(math.Ceil(float64(*count) / float64(*concurrency)))

	start := time.Now()
	for w := 0; w < *concurrency; w++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			sock := zmq4.NewPush(context.Background())
			defer sock.Close()
			if err := sock.Dial(*connect); err != nil {
				log.Fatalf("dial: %v", err)
			}
			for i := 0; i < per; i++ {
				if err := sock.Send(zmq4.NewMsg(payload)); err != nil {
					log.Fatalf("send: %v", err)
				}
			}
		}()
	}
	wg.Wait()
	elapsed := time.Since(start).Seconds()
	sent := per * (*concurrency)
	mps := float64(sent) / elapsed
	mbps := (float64(sent*(*size)) / (1024 * 1024)) / elapsed
	fmt.Printf("Sent %d msgs (size=%d) in %.3fs => %.0f msgs/s, %.2f MiB/s\n",
		sent, *size, elapsed, mps, mbps)
}

```

- 編譯與運行

```sh
go build -o tput_server.exe tput_server.go
go build -o tput_client.exe tput_client.go

.\tput_server.exe -bind tcp://*:5556

# 再新開一個cmd執行
.\tput_client.exe -connect tcp://127.0.0.1:5556 -count 1000000 -size 100 -concurrency 4
```

## 測試結果

從結果來看，來觀察到100萬消息，zeromq大概5秒送完，且前前3秒都有20萬左右的消息量。

```sh
# Server紀錄
tput server listening on tcp://*:5556
[1s] window=0 msgs  rate=0 msgs/s  total=0
[2s] window=0 msgs  rate=0 msgs/s  total=0
[3s] window=0 msgs  rate=0 msgs/s  total=0
[4s] window=0 msgs  rate=0 msgs/s  total=0
[5s] window=222110 msgs  rate=222220 msgs/s  total=222110
[6s] window=281650 msgs  rate=281727 msgs/s  total=503804
[7s] window=270705 msgs  rate=270666 msgs/s  total=774547
[8s] window=181444 msgs  rate=181431 msgs/s  total=956011
[9s] window=43964 msgs  rate=43945 msgs/s  total=1000000
[10s] window=0 msgs  rate=0 msgs/s  total=1000000
[11s] window=0 msgs  rate=0 msgs/s  total=1000000
```

## 明日接續

有了這樣的測試之後，明天就準備換成C++語言來做個比較。
