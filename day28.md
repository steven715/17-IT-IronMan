# 網路協議 tcp 體驗 4

今天延續昨天的項目，我想了解不同語言(go, rust)在ubuntu機器中的轉發器表現。

## Go 版本轉發器

### 前置條件

- go 的版本要 1.19 以上，[go 下載](https://go.dev/dl/)可以從官網上下載最新版來在本地使用

### 專案目錄

```sh
C:\USERS\ASUS\STEVEN\TCP_FORWARD_GO
│  Dockerfile
│  go.mod
│  main.go
│  README.md
│
├─cmd
│  └─bench_client
│          bench_client.go
│
└─tcpserver
        tcpserver.go
```

### go.mod

```go
module tcp_forward

go 1.19

```

### main.go

```go
package main

import (
	"fmt"
	"net"
	"os"
	"os/signal"
	"strconv"
	"sync/atomic"
	"syscall"
	"tcp_forward/tcpserver"
	"time"
)

func main() {
	recvPort := uint16(7001)
	sendPort := uint16(7002)

	if len(os.Args) >= 2 {
		if p, err := strconv.Atoi(os.Args[1]); err == nil {
			recvPort = uint16(p)
		}
	}
	if len(os.Args) >= 3 {
		if p, err := strconv.Atoi(os.Args[2]); err == nil {
			sendPort = uint16(p)
		}
	}

	recvSrv, err := tcpserver.New(recvPort)
	if err != nil {
		fmt.Printf("Failed to create RecvServer: %v\n", err)
		os.Exit(1)
	}

	sendSrv, err := tcpserver.New(sendPort)
	if err != nil {
		fmt.Printf("Failed to create SendServer: %v\n", err)
		os.Exit(1)
	}

	var recvCount atomic.Uint64
	var sendCount atomic.Uint64

	recvSrv.SetOnOpen(func(addr net.Addr) {
		fmt.Printf("🔗 RecvServer OPEN: %v\n", addr)
	})

	recvSrv.SetOnClose(func(addr net.Addr) {
		fmt.Printf("❌ RecvServer CLOSE: %v\n", addr)
	})

	recvSrv.SetOnMsg(func(msg string, addr net.Addr) {
		recvCount.Add(1)
		line := msg + "\n"
		sentTo := sendSrv.Broadcast(line)
		sendCount.Add(uint64(sentTo))
	})

	sendSrv.SetOnOpen(func(addr net.Addr) {
		fmt.Printf("🔗 SendServer OPEN: %v\n", addr)
	})

	sendSrv.SetOnClose(func(addr net.Addr) {
		fmt.Printf("❌ SendServer CLOSE: %v\n", addr)
	})

	sendSrv.SetOnMsg(func(msg string, addr net.Addr) {
		fmt.Printf("📩 SendServer got (unexpected) from %v: %s\n", addr, msg)
	})

	recvSrv.Start()
	sendSrv.Start()

	// Statistics timer
	stopStats := make(chan struct{})
	go func() {
		ticker := time.NewTicker(60 * time.Second)
		defer ticker.Stop()

		lastTime := time.Now()
		var lastRecv, lastSend uint64

		for {
			select {
			case <-stopStats:
				return
			case <-ticker.C:
				now := time.Now()
				elapsed := now.Sub(lastTime).Seconds()
				lastTime = now

				currRecv := recvCount.Load()
				currSend := sendCount.Load()
				deltaRecv := currRecv - lastRecv
				deltaSend := currSend - lastSend
				lastRecv = currRecv
				lastSend = currSend

				rps := float64(deltaRecv) / elapsed
				sps := float64(deltaSend) / elapsed

				fmt.Printf("⏱️  Interval %.0fs | Recv: %d (%.2f/s), Sent: %d (%.2f/s) | SendConn=%d RecvConn=%d\n",
					elapsed, deltaRecv, rps, deltaSend, sps,
					sendSrv.ConnectionCount(), recvSrv.ConnectionCount())
			}
		}
	}()

	// Signal handling
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	sig := <-sigChan
	fmt.Printf("\nSignal %v received, shutting down now...\n", sig)

	close(stopStats)
	recvSrv.Stop()
	sendSrv.Stop()

	fmt.Println("Bye.")
}

```

### tcpserver/tcpserver.go

```go
package tcpserver

import (
	"bufio"
	"fmt"
	"io"
	"net"
	"sync"
	"sync/atomic"
)

// TcpServer represents a TCP server that can accept connections and broadcast messages
type TcpServer struct {
	listener   net.Listener
	port       uint16
	sessions   map[*Session]struct{}
	mu         sync.Mutex
	onOpen     func(net.Addr)
	onClose    func(net.Addr)
	onMsg      func(string, net.Addr)
	stopChan   chan struct{}
	wg         sync.WaitGroup
	connCount  atomic.Int64
}

// Session represents a single client connection
type Session struct {
	conn   net.Conn
	server *TcpServer
}

// New creates a new TcpServer
func New(port uint16) (*TcpServer, error) {
	listener, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
	if err != nil {
		return nil, err
	}

	return &TcpServer{
		listener: listener,
		port:     port,
		sessions: make(map[*Session]struct{}),
		stopChan: make(chan struct{}),
	}, nil
}

// SetOnOpen sets the callback for when a connection is opened
func (s *TcpServer) SetOnOpen(cb func(net.Addr)) {
	s.onOpen = cb
}

// SetOnClose sets the callback for when a connection is closed
func (s *TcpServer) SetOnClose(cb func(net.Addr)) {
	s.onClose = cb
}

// SetOnMsg sets the callback for when a message is received
func (s *TcpServer) SetOnMsg(cb func(string, net.Addr)) {
	s.onMsg = cb
}

// Start starts the server and begins accepting connections
func (s *TcpServer) Start() {
	fmt.Printf("✅ Server listening on port %d\n", s.port)
	s.wg.Add(1)
	go s.acceptLoop()
}

// Stop stops the server and closes all connections
func (s *TcpServer) Stop() {
	close(s.stopChan)
	s.listener.Close()

	s.mu.Lock()
	for sess := range s.sessions {
		sess.conn.Close()
	}
	s.sessions = make(map[*Session]struct{})
	s.mu.Unlock()

	s.wg.Wait()
}

// Broadcast sends a message to all connected clients
// Returns the number of clients the message was sent to
func (s *TcpServer) Broadcast(msg string) int {
	s.mu.Lock()
	snapshot := make([]*Session, 0, len(s.sessions))
	for sess := range s.sessions {
		snapshot = append(snapshot, sess)
	}
	s.mu.Unlock()

	sent := 0
	for _, sess := range snapshot {
		_, err := sess.conn.Write([]byte(msg))
		if err == nil {
			sent++
		}
	}
	return sent
}

// ConnectionCount returns the current number of active connections
func (s *TcpServer) ConnectionCount() int {
	s.mu.Lock()
	defer s.mu.Unlock()
	return len(s.sessions)
}

func (s *TcpServer) acceptLoop() {
	defer s.wg.Done()

	for {
		conn, err := s.listener.Accept()
		if err != nil {
			select {
			case <-s.stopChan:
				return
			default:
				continue
			}
		}

		sess := &Session{
			conn:   conn,
			server: s,
		}

		s.mu.Lock()
		s.sessions[sess] = struct{}{}
		s.mu.Unlock()

		s.connCount.Add(1)

		if s.onOpen != nil {
			s.onOpen(conn.RemoteAddr())
		}

		s.wg.Add(1)
		go sess.handleConnection()
	}
}

func (sess *Session) handleConnection() {
	defer sess.server.wg.Done()
	defer func() {
		addr := sess.conn.RemoteAddr()
		sess.conn.Close()

		sess.server.mu.Lock()
		delete(sess.server.sessions, sess)
		sess.server.mu.Unlock()

		if sess.server.onClose != nil {
			sess.server.onClose(addr)
		}
	}()

	reader := bufio.NewReader(sess.conn)
	for {
		select {
		case <-sess.server.stopChan:
			return
		default:
		}

		line, err := reader.ReadString('\n')
		if err != nil {
			if err != io.EOF {
				// Connection error
			}
			return
		}

		// Remove the trailing newline
		if len(line) > 0 && line[len(line)-1] == '\n' {
			line = line[:len(line)-1]
		}
		if len(line) > 0 && line[len(line)-1] == '\r' {
			line = line[:len(line)-1]
		}

		if sess.server.onMsg != nil {
			sess.server.onMsg(line, sess.conn.RemoteAddr())
		}
	}
}

```

### cmd/bench_client/bench_client.go

```go
package main

import (
	"bufio"
	"context"
	"flag"
	"fmt"
	"io"
	"net"
	"os"
	"os/signal"
	"runtime"
	"sort"
	"strconv"
	"strings"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
)

type Args struct {
	PubHost  string
	PubPort  uint16
	SubHost  string
	SubPort  uint16
	Pubs     int
	Subs     int
	Rate     int
	MsgSize  int
	Duration int
}

type LatencyStats struct {
	mu        sync.Mutex
	samplesMs []float64
	cap       int
}

func NewLatencyStats() *LatencyStats {
	return &LatencyStats{
		samplesMs: make([]float64, 0, 200000),
		cap:       200000,
	}
}

func (ls *LatencyStats) Add(ms float64) {
	ls.mu.Lock()
	defer ls.mu.Unlock()
	if len(ls.samplesMs) < ls.cap {
		ls.samplesMs = append(ls.samplesMs, ms)
	}
}

func (ls *LatencyStats) Print() {
	ls.mu.Lock()
	v := make([]float64, len(ls.samplesMs))
	copy(v, ls.samplesMs)
	ls.mu.Unlock()

	if len(v) == 0 {
		fmt.Println("latency: no samples")
		return
	}

	// Calculate average
	sum := 0.0
	for _, x := range v {
		sum += x
	}
	avg := sum / float64(len(v))

	// Calculate percentiles
	sort.Float64s(v)
	p50 := percentile(v, 0.50)
	p90 := percentile(v, 0.90)
	p99 := percentile(v, 0.99)

	fmt.Printf("Latency (ms): avg=%.3f p50=%.3f p90=%.3f p99=%.3f\n", avg, p50, p90, p99)
}

func percentile(sorted []float64, p float64) float64 {
	if len(sorted) == 0 {
		return 0.0
	}
	idx := int(p * float64(len(sorted)-1))
	if idx >= len(sorted) {
		idx = len(sorted) - 1
	}
	return sorted[idx]
}

type Shared struct {
	sent     atomic.Uint64
	received atomic.Uint64
	lat      *LatencyStats
}

func nowNs() uint64 {
	return uint64(time.Now().UnixNano())
}

// Publisher sends messages at a fixed rate
type Publisher struct {
	conn    net.Conn
	id      int
	rate    int
	msgSize int
	seq     uint64
	shared  *Shared
	stop    <-chan struct{}
}

func NewPublisher(host string, port uint16, id, rate, msgSize int, shared *Shared, stop <-chan struct{}) (*Publisher, error) {
	conn, err := net.Dial("tcp", fmt.Sprintf("%s:%d", host, port))
	if err != nil {
		return nil, err
	}

	return &Publisher{
		conn:    conn,
		id:      id,
		rate:    rate,
		msgSize: msgSize,
		shared:  shared,
		stop:    stop,
	}, nil
}

func (p *Publisher) Start(ctx context.Context) {
	defer p.conn.Close()

	interval := time.Second / time.Duration(p.rate)
	if p.rate <= 0 {
		interval = 0
	}

	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	for {
		select {
		case <-p.stop:
			return
		case <-ctx.Done():
			return
		case <-ticker.C:
			p.sendOne()
		}
	}
}

func (p *Publisher) sendOne() {
	p.seq++
	ts := nowNs()

	// Format: pubId,seq,ts,<padding>\n
	msg := fmt.Sprintf("%d,%d,%d,", p.id, p.seq, ts)
	if len(msg)+1 < p.msgSize {
		padding := strings.Repeat("x", p.msgSize-len(msg)-1)
		msg += padding
	}
	msg += "\n"

	_, err := p.conn.Write([]byte(msg))
	if err != nil {
		return
	}

	p.shared.sent.Add(1)
}

// Subscriber receives messages and calculates latency
type Subscriber struct {
	conn   net.Conn
	shared *Shared
	stop   <-chan struct{}
}

func NewSubscriber(host string, port uint16, shared *Shared, stop <-chan struct{}) (*Subscriber, error) {
	conn, err := net.Dial("tcp", fmt.Sprintf("%s:%d", host, port))
	if err != nil {
		return nil, err
	}

	return &Subscriber{
		conn:   conn,
		shared: shared,
		stop:   stop,
	}, nil
}

func (s *Subscriber) Start(ctx context.Context) {
	defer s.conn.Close()

	reader := bufio.NewReader(s.conn)
	for {
		select {
		case <-s.stop:
			return
		case <-ctx.Done():
			return
		default:
		}

		line, err := reader.ReadString('\n')
		if err != nil {
			if err != io.EOF {
				// Connection error
			}
			return
		}

		// Parse: pubId,seq,ts,...
		parts := strings.Split(line, ",")
		if len(parts) >= 3 {
			if sentNs, err := strconv.ParseUint(parts[2], 10, 64); err == nil && sentNs > 0 {
				recvNs := nowNs()
				ms := float64(recvNs-sentNs) / 1e6
				s.shared.lat.Add(ms)
			}
		}

		s.shared.received.Add(1)
	}
}

func parseArgs() Args {
	args := Args{}

	flag.StringVar(&args.PubHost, "pub-host", "127.0.0.1", "RecvServer host")
	flag.Func("pub-port", "RecvServer port", func(s string) error {
		p, err := strconv.Atoi(s)
		if err != nil {
			return err
		}
		args.PubPort = uint16(p)
		return nil
	})
	flag.StringVar(&args.SubHost, "sub-host", "127.0.0.1", "SendServer host")
	flag.Func("sub-port", "SendServer port", func(s string) error {
		p, err := strconv.Atoi(s)
		if err != nil {
			return err
		}
		args.SubPort = uint16(p)
		return nil
	})
	flag.IntVar(&args.Pubs, "pub", 1, "Publishers count")
	flag.IntVar(&args.Subs, "sub", 1, "Subscribers count")
	flag.IntVar(&args.Rate, "rate", 1000, "Msgs/sec per publisher")
	flag.IntVar(&args.MsgSize, "msg-size", 64, "Bytes per message incl. newline")
	flag.IntVar(&args.Duration, "duration", 10, "Duration in seconds")

	flag.Parse()

	// Defaults
	if args.PubPort == 0 {
		args.PubPort = 7001
	}
	if args.SubPort == 0 {
		args.SubPort = 7002
	}
	if args.MsgSize < 16 {
		args.MsgSize = 16
	}

	return args
}

func main() {
	args := parseArgs()

	fmt.Printf("bench_client start\n")
	fmt.Printf("pubs=%d subs=%d rate=%d/pub msg_size=%d duration=%ds threads=%d\n",
		args.Pubs, args.Subs, args.Rate, args.MsgSize, args.Duration, runtime.NumCPU())

	shared := &Shared{
		lat: NewLatencyStats(),
	}

	stopChan := make(chan struct{})
	ctx, cancel := context.WithTimeout(context.Background(), time.Duration(args.Duration)*time.Second)
	defer cancel()

	var wg sync.WaitGroup

	// Create publishers
	for i := 0; i < args.Pubs; i++ {
		pub, err := NewPublisher(args.PubHost, args.PubPort, i, args.Rate, args.MsgSize, shared, stopChan)
		if err != nil {
			fmt.Printf("Failed to create publisher %d: %v\n", i, err)
			continue
		}
		wg.Add(1)
		go func() {
			defer wg.Done()
			pub.Start(ctx)
		}()
	}

	// Create subscribers
	for i := 0; i < args.Subs; i++ {
		sub, err := NewSubscriber(args.SubHost, args.SubPort, shared, stopChan)
		if err != nil {
			fmt.Printf("Failed to create subscriber %d: %v\n", i, err)
			continue
		}
		wg.Add(1)
		go func() {
			defer wg.Done()
			sub.Start(ctx)
		}()
	}

	// Signal handling
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	select {
	case <-ctx.Done():
		// Duration expired
	case <-sigChan:
		// Signal received
	}

	close(stopChan)
	cancel()
	wg.Wait()

	// Print statistics
	sent := shared.sent.Load()
	recv := shared.received.Load()
	secs := float64(args.Duration)
	sendRate := float64(sent) / secs
	recvRate := float64(recv) / secs

	fmt.Println("==== bench result ====")
	fmt.Printf("Sent: %d msgs (%.2f msg/s)\n", sent, sendRate)
	fmt.Printf("Recv: %d msgs (%.2f msg/s)\n", recv, recvRate)
	shared.lat.Print()
	fmt.Println("======================")
}

```

### Dockerfile

```dockerfile
# Multi-stage build for optimal image size
FROM golang:1.19-bullseye AS builder

WORKDIR /build

# Copy go mod files
COPY go.mod go.sum* ./
RUN go mod download

# Copy source code
COPY . .

# Build main application
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o tcp_forward .

# Build bench_client
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o bench_client ./cmd/bench_client

# Final stage - Ubuntu 22.04
FROM ubuntu:22.04

# Install ca-certificates for HTTPS connections (if needed)
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy binaries from builder
COPY --from=builder /build/tcp_forward .
COPY --from=builder /build/bench_client .

# Expose default ports
EXPOSE 7001 7002

# Default command runs the main tcp_forward server
CMD ["./tcp_forward"]

```

### 編譯與運行

1. 建構 Docker 映像檔

```sh
docker build -t tcp-forward-go .
```

2. 執行容器 (主伺服器)

```sh
docker run -p 7001:7001 -p 7002:7002 tcp-forward-go
```

3. 進入容器，執行效能測試客戶端

```sh
# 透過vs code 的dev container進入
cd /app
./bench_client --pub-host 0.0.0.0 --pub 1 --sub 3 --rate 10000 --msg-size 80 --duration 15

# 執行結果
bench_client start
pubs=1 subs=3 rate=10000/pub msg_size=80 duration=15s threads=16
==== bench result ====
Sent: 115613 msgs (7707.53 msg/s)
Recv: 346839 msgs (23122.60 msg/s)
Latency (ms): avg=0.171 p50=0.169 p90=0.220 p99=0.296
======================
```

### 與 C++ 版本比較

意外的發現性能竟然是 go 語言的比較出色，但這部分目前我是沒辦法解釋的。

```sh
# C++ 版本的執行結果
bench_client start
pubs=1 subs=3 rate=10000/pub msg_size=80 duration=15s threads=16
==== bench result ====
Sent: 54754 msgs (3650.27 msg/s)
Recv: 164262 msgs (10950.8 msg/s)
Latency (ms): avg=0.257809 p50=0.249847 p90=0.331499 p99=0.437018
======================
```

## 結論

有了AI工具，現在換語言的實作真的變超輕鬆，就算對go語言只是懵懵懂懂，也是能順利移植過來，雖然中間會遇到一些問題，但AI都會處理，不過用下來除非本身很懂要AI做的東西是什麼，不然在本身不太懂的情況下使用，真的會有很空虛的現象，變成完全依賴AI才能運作，我想這就是未來工程師真正的考驗吧。
