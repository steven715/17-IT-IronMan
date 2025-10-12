# 網路協議 tcp 體驗 5

今天也是延續昨天的項目，將 rust 版本的轉發器補完。

## Rust 版本轉發器

### 專案目錄

```sh
C:\USERS\ASUS\STEVEN\TCP_FORWARD_RUST
│  Cargo.toml
│  Dockerfile
│  README.md
│
└─src
        bench_client.rs
        main.rs
        tcp_server.rs
```

### Cargo.toml

```toml
[package]
name = "tcp_forward_rust"
version = "0.1.0"
edition = "2021"

[[bin]]
name = "tcp_forwarder"
path = "src/main.rs"

[[bin]]
name = "bench_client"
path = "src/bench_client.rs"

[dependencies]
tokio = { version = "1.42", features = ["full"] }
tokio-util = { version = "0.7", features = ["codec"] }
futures = "0.3"
bytes = "1.9"
clap = { version = "4.5", features = ["derive"] }

```

### src/tcp_server.rs

```rust
use std::collections::HashSet;
use std::net::SocketAddr;
use std::sync::Arc;
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::{Mutex, RwLock};
use tokio::task::JoinHandle;

type OnOpenCallback = Arc<dyn Fn(SocketAddr) + Send + Sync>;
type OnCloseCallback = Arc<dyn Fn(SocketAddr) + Send + Sync>;
type OnMsgCallback = Arc<dyn Fn(String, SocketAddr) + Send + Sync>;

#[derive(Clone)]
struct Session {
    addr: SocketAddr,
    writer: Arc<Mutex<tokio::io::WriteHalf<TcpStream>>>,
}

pub struct TcpServer {
    port: u16,
    sessions: Arc<RwLock<HashSet<SocketAddr>>>,
    session_writers: Arc<RwLock<std::collections::HashMap<SocketAddr, Arc<Mutex<tokio::io::WriteHalf<TcpStream>>>>>>,
    on_open: Arc<Mutex<Option<OnOpenCallback>>>,
    on_close: Arc<Mutex<Option<OnCloseCallback>>>,
    on_msg: Arc<Mutex<Option<OnMsgCallback>>>,
    shutdown_tx: tokio::sync::broadcast::Sender<()>,
}

impl TcpServer {
    pub fn new(port: u16) -> Self {
        let (shutdown_tx, _) = tokio::sync::broadcast::channel(1);
        Self {
            port,
            sessions: Arc::new(RwLock::new(HashSet::new())),
            session_writers: Arc::new(RwLock::new(std::collections::HashMap::new())),
            on_open: Arc::new(Mutex::new(None)),
            on_close: Arc::new(Mutex::new(None)),
            on_msg: Arc::new(Mutex::new(None)),
            shutdown_tx,
        }
    }

    pub async fn set_on_open<F>(&self, callback: F)
    where
        F: Fn(SocketAddr) + Send + Sync + 'static,
    {
        let mut cb = self.on_open.lock().await;
        *cb = Some(Arc::new(callback));
    }

    pub async fn set_on_close<F>(&self, callback: F)
    where
        F: Fn(SocketAddr) + Send + Sync + 'static,
    {
        let mut cb = self.on_close.lock().await;
        *cb = Some(Arc::new(callback));
    }

    pub async fn set_on_msg<F>(&self, callback: F)
    where
        F: Fn(String, SocketAddr) + Send + Sync + 'static,
    {
        let mut cb = self.on_msg.lock().await;
        *cb = Some(Arc::new(callback));
    }

    pub async fn start(self: Arc<Self>) -> std::io::Result<JoinHandle<()>> {
        let listener = TcpListener::bind(format!("0.0.0.0:{}", self.port)).await?;
        println!("✅ Server listening on port {}", self.port);

        let handle = tokio::spawn(async move {
            let mut shutdown_rx = self.shutdown_tx.subscribe();
            loop {
                tokio::select! {
                    result = listener.accept() => {
                        match result {
                            Ok((socket, addr)) => {
                                let server = Arc::clone(&self);
                                tokio::spawn(async move {
                                    server.handle_connection(socket, addr).await;
                                });
                            }
                            Err(e) => {
                                eprintln!("Accept error: {}", e);
                            }
                        }
                    }
                    _ = shutdown_rx.recv() => {
                        break;
                    }
                }
            }
        });

        Ok(handle)
    }

    async fn handle_connection(&self, socket: TcpStream, addr: SocketAddr) {
        // Register session
        self.sessions.write().await.insert(addr);

        let (reader, writer) = tokio::io::split(socket);
        let writer = Arc::new(Mutex::new(writer));
        self.session_writers.write().await.insert(addr, Arc::clone(&writer));

        // Call on_open callback
        if let Some(cb) = self.on_open.lock().await.as_ref() {
            cb(addr);
        }

        // Handle reads
        let mut reader = BufReader::new(reader);
        let mut line = String::new();

        loop {
            line.clear();
            match reader.read_line(&mut line).await {
                Ok(0) => break, // EOF
                Ok(_) => {
                    let msg = line.trim_end().to_string();
                    if let Some(cb) = self.on_msg.lock().await.as_ref() {
                        cb(msg, addr);
                    }
                }
                Err(_) => break,
            }
        }

        // Cleanup
        self.sessions.write().await.remove(&addr);
        self.session_writers.write().await.remove(&addr);

        if let Some(cb) = self.on_close.lock().await.as_ref() {
            cb(addr);
        }
    }

    pub async fn broadcast(&self, msg: &str) -> usize {
        let writers = self.session_writers.read().await;
        let mut count = 0;

        for writer in writers.values() {
            let mut w = writer.lock().await;
            if w.write_all(msg.as_bytes()).await.is_ok() {
                count += 1;
            }
        }

        count
    }

    pub async fn connection_count(&self) -> usize {
        self.sessions.read().await.len()
    }

    pub fn shutdown(&self) {
        let _ = self.shutdown_tx.send(());
    }
}

```

### src/main.rs

```rust
mod tcp_server;

use std::env;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::time::{Duration, Instant};
use tcp_server::TcpServer;
use tokio::signal;
use tokio::time::interval;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().collect();

    let recv_port: u16 = args.get(1).and_then(|s| s.parse().ok()).unwrap_or(7001);
    let send_port: u16 = args.get(2).and_then(|s| s.parse().ok()).unwrap_or(7002);

    let recv_count = Arc::new(AtomicU64::new(0));
    let send_count = Arc::new(AtomicU64::new(0));

    let recv_srv = Arc::new(TcpServer::new(recv_port));
    let send_srv = Arc::new(TcpServer::new(send_port));

    // Setup RecvServer callbacks
    let recv_srv_clone = Arc::clone(&recv_srv);
    recv_srv.set_on_open(move |ep| {
        println!("🔗 RecvServer OPEN: {}", ep);
    }).await;

    let recv_srv_clone = Arc::clone(&recv_srv);
    recv_srv.set_on_close(move |ep| {
        println!("❌ RecvServer CLOSE: {}", ep);
    }).await;

    let send_srv_clone = Arc::clone(&send_srv);
    let recv_count_clone = Arc::clone(&recv_count);
    let send_count_clone = Arc::clone(&send_count);
    recv_srv.set_on_msg(move |msg, _ep| {
        recv_count_clone.fetch_add(1, Ordering::Relaxed);
        let line = format!("{}\n", msg);
        let send_srv = Arc::clone(&send_srv_clone);
        let send_count = Arc::clone(&send_count_clone);
        tokio::spawn(async move {
            let sent_to = send_srv.broadcast(&line).await;
            send_count.fetch_add(sent_to as u64, Ordering::Relaxed);
        });
    }).await;

    // Setup SendServer callbacks
    send_srv.set_on_open(move |ep| {
        println!("🔗 SendServer OPEN: {}", ep);
    }).await;

    send_srv.set_on_close(move |ep| {
        println!("❌ SendServer CLOSE: {}", ep);
    }).await;

    send_srv.set_on_msg(move |msg, ep| {
        println!("📩 SendServer got (unexpected) from {}: {}", ep, msg);
    }).await;

    // Start servers
    let recv_handle = Arc::clone(&recv_srv).start().await?;
    let send_handle = Arc::clone(&send_srv).start().await?;

    // Statistics timer
    let recv_count_stats = Arc::clone(&recv_count);
    let send_count_stats = Arc::clone(&send_count);
    let send_srv_stats = Arc::clone(&send_srv);
    let recv_srv_stats = Arc::clone(&recv_srv);

    let stats_handle = tokio::spawn(async move {
        let mut ticker = interval(Duration::from_secs(60));
        let mut last = Instant::now();
        let mut last_recv = 0u64;
        let mut last_send = 0u64;

        loop {
            ticker.tick().await;

            let now = Instant::now();
            let sec = now.duration_since(last).as_secs();
            last = now;

            let cr = recv_count_stats.load(Ordering::Relaxed);
            let cs = send_count_stats.load(Ordering::Relaxed);
            let dr = cr - last_recv;
            let ds = cs - last_send;
            last_recv = cr;
            last_send = cs;

            let rps = if sec > 0 { dr as f64 / sec as f64 } else { 0.0 };
            let sps = if sec > 0 { ds as f64 / sec as f64 } else { 0.0 };

            let send_conn = send_srv_stats.connection_count().await;
            let recv_conn = recv_srv_stats.connection_count().await;

            println!(
                "⏱️  Interval {}s | Recv: {} ({:.2}/s), Sent: {} ({:.2}/s) | SendConn={} RecvConn={}",
                sec, dr, rps, ds, sps, send_conn, recv_conn
            );
        }
    });

    // Wait for Ctrl+C
    signal::ctrl_c().await?;
    println!("\nSignal received, shutting down now...");

    // Shutdown
    recv_srv.shutdown();
    send_srv.shutdown();
    stats_handle.abort();

    // Wait for server tasks to complete (with timeout)
    let _ = tokio::time::timeout(Duration::from_secs(2), async {
        let _ = recv_handle.await;
        let _ = send_handle.await;
    }).await;

    println!("Bye.");
    Ok(())
}

```

### src/bench_client.rs

```rust
use clap::Parser;
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::sync::Arc;
use std::time::{Duration, Instant, SystemTime, UNIX_EPOCH};
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpStream;
use tokio::signal;
use tokio::sync::Mutex;
use tokio::time::interval;

#[derive(Parser, Debug)]
#[command(name = "bench_client")]
#[command(about = "TCP benchmark client")]
struct Args {
    #[arg(long, default_value = "127.0.0.1")]
    pub_host: String,

    #[arg(long, default_value_t = 7001)]
    pub_port: u16,

    #[arg(long, default_value = "127.0.0.1")]
    sub_host: String,

    #[arg(long, default_value_t = 7002)]
    sub_port: u16,

    #[arg(long, default_value_t = 1)]
    pub_: usize,

    #[arg(long, default_value_t = 1)]
    sub: usize,

    #[arg(long, default_value_t = 1000)]
    rate: u64,

    #[arg(long, default_value_t = 64)]
    msg_size: usize,

    #[arg(long, default_value_t = 10)]
    duration: u64,
}

struct LatencyStats {
    samples: Mutex<Vec<f64>>,
    cap: usize,
}

impl LatencyStats {
    fn new(cap: usize) -> Self {
        Self {
            samples: Mutex::new(Vec::new()),
            cap,
        }
    }

    async fn add(&self, ms: f64) {
        let mut samples = self.samples.lock().await;
        if samples.len() < self.cap {
            samples.push(ms);
        }
    }

    async fn print(&self) {
        let mut samples = self.samples.lock().await;
        if samples.is_empty() {
            println!("latency: no samples");
            return;
        }

        let sum: f64 = samples.iter().sum();
        let avg = sum / samples.len() as f64;

        samples.sort_by(|a, b| a.partial_cmp(b).unwrap());

        let p50_idx = (0.50 * (samples.len() - 1) as f64) as usize;
        let p90_idx = (0.90 * (samples.len() - 1) as f64) as usize;
        let p99_idx = (0.99 * (samples.len() - 1) as f64) as usize;

        let p50 = samples[p50_idx];
        let p90 = samples[p90_idx];
        let p99 = samples[p99_idx];

        println!(
            "Latency (ms): avg={:.2} p50={:.2} p90={:.2} p99={:.2}",
            avg, p50, p90, p99
        );
    }
}

struct Shared {
    sent: AtomicU64,
    received: AtomicU64,
    lat: LatencyStats,
}

fn now_ns() -> u128 {
    SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .as_nanos()
}

async fn publisher(
    host: String,
    port: u16,
    pub_id: usize,
    rate: u64,
    msg_size: usize,
    shared: Arc<Shared>,
    stop: Arc<AtomicBool>,
) {
    let mut stream = match TcpStream::connect(format!("{}:{}", host, port)).await {
        Ok(s) => s,
        Err(e) => {
            eprintln!("Publisher connect error: {}", e);
            return;
        }
    };

    let interval_ns = if rate > 0 {
        1_000_000_000 / rate
    } else {
        0
    };
    let mut ticker = interval(Duration::from_nanos(interval_ns));
    let mut seq = 0u64;

    while !stop.load(Ordering::Relaxed) {
        ticker.tick().await;

        seq += 1;
        let ts = now_ns();

        let mut msg = format!("{},{},{},", pub_id, seq, ts);
        if msg.len() + 1 < msg_size {
            msg.push_str(&"x".repeat(msg_size - msg.len() - 1));
        }
        msg.push('\n');

        match stream.write_all(msg.as_bytes()).await {
            Ok(_) => {
                shared.sent.fetch_add(1, Ordering::Relaxed);
            }
            Err(e) => {
                eprintln!("Publisher write error: {}", e);
                stop.store(true, Ordering::Relaxed);
                break;
            }
        }
    }
}

async fn subscriber(
    host: String,
    port: u16,
    shared: Arc<Shared>,
    stop: Arc<AtomicBool>,
) {
    let stream = match TcpStream::connect(format!("{}:{}", host, port)).await {
        Ok(s) => s,
        Err(e) => {
            eprintln!("Subscriber connect error: {}", e);
            return;
        }
    };

    let reader = BufReader::new(stream);
    let mut lines = reader.lines();

    while !stop.load(Ordering::Relaxed) {
        match lines.next_line().await {
            Ok(Some(line)) => {
                // Parse: pubId,seq,ts,...
                let parts: Vec<&str> = line.split(',').collect();
                if parts.len() >= 3 {
                    if let Ok(sent_ns) = parts[2].parse::<u128>() {
                        let recv_ns = now_ns();
                        let ms = (recv_ns - sent_ns) as f64 / 1_000_000.0;
                        shared.lat.add(ms).await;
                    }
                }
                shared.received.fetch_add(1, Ordering::Relaxed);
            }
            Ok(None) => break,
            Err(e) => {
                if !stop.load(Ordering::Relaxed) {
                    eprintln!("Subscriber read error: {}", e);
                }
                break;
            }
        }
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let args = Args::parse();

    println!(
        "bench_client start\npubs={} subs={} rate={}/pub msg_size={} duration={}s",
        args.pub_, args.sub, args.rate, args.msg_size, args.duration
    );

    let shared = Arc::new(Shared {
        sent: AtomicU64::new(0),
        received: AtomicU64::new(0),
        lat: LatencyStats::new(200000),
    });

    let stop = Arc::new(AtomicBool::new(false));

    let mut handles = Vec::new();

    // Start publishers
    for i in 0..args.pub_ {
        let host = args.pub_host.clone();
        let port = args.pub_port;
        let shared = Arc::clone(&shared);
        let stop = Arc::clone(&stop);
        let rate = args.rate;
        let msg_size = args.msg_size;

        handles.push(tokio::spawn(async move {
            publisher(host, port, i, rate, msg_size, shared, stop).await;
        }));
    }

    // Start subscribers
    for _ in 0..args.sub {
        let host = args.sub_host.clone();
        let port = args.sub_port;
        let shared = Arc::clone(&shared);
        let stop = Arc::clone(&stop);

        handles.push(tokio::spawn(async move {
            subscriber(host, port, shared, stop).await;
        }));
    }

    // Timer for duration
    let stop_timer = Arc::clone(&stop);
    let duration = args.duration;
    tokio::spawn(async move {
        tokio::time::sleep(Duration::from_secs(duration)).await;
        stop_timer.store(true, Ordering::Relaxed);
    });

    // Ctrl-C handler
    let stop_signal = Arc::clone(&stop);
    tokio::spawn(async move {
        signal::ctrl_c().await.ok();
        stop_signal.store(true, Ordering::Relaxed);
    });

    // Wait for all tasks
    for handle in handles {
        let _ = handle.await;
    }

    // Statistics
    let sent = shared.sent.load(Ordering::Relaxed);
    let recv = shared.received.load(Ordering::Relaxed);
    let secs = args.duration as f64;
    let send_rate = if secs > 0.0 { sent as f64 / secs } else { 0.0 };
    let recv_rate = if secs > 0.0 { recv as f64 / secs } else { 0.0 };

    println!("==== bench result ====");
    println!("Sent: {} msgs ({:.2} msg/s)", sent, send_rate);
    println!("Recv: {} msgs ({:.2} msg/s)", recv, recv_rate);
    shared.lat.print().await;
    println!("======================");

    Ok(())
}
```

### Dockerfile

```dockerfile
# Build stage
FROM ubuntu:22.04 AS builder

# Install build dependencies
RUN apt-get update && apt-get install -y \
    curl \
    build-essential \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Rust
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
ENV PATH="/root/.cargo/bin:${PATH}"

# Set working directory
WORKDIR /app

# Copy project files
COPY Cargo.toml ./
COPY src ./src

# Build release binaries
RUN cargo build --release

# Runtime stage
FROM ubuntu:22.04

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Copy binaries from builder
COPY --from=builder /app/target/release/tcp_forwarder /usr/local/bin/
COPY --from=builder /app/target/release/bench_client /usr/local/bin/

# Expose default ports
EXPOSE 7001 7002

# Default command runs the tcp_forwarder
CMD ["tcp_forwarder"]
```

### 編譯與運行

1. 建構 Docker 映像檔

```sh
docker build -t tcp_forwarder_rust:latest .
```

2. 執行容器 (主伺服器)

```sh
docker run -d --name tcp_forwarder_rust -p 7001:7001 -p 7002:7002 tcp_forwarder_rust:latest

# 查看日誌
docker logs -f tcp_forwarder_rust

# 停止容器
docker stop tcp_forwarder_rust
docker rm tcp_forwarder_rust
```

3. 進入容器，執行效能測試客戶端

```sh
# 進入容器執行
cd /usr/local/bin
./bench_client --pub 1 --sub 3 --rate 10000 --msg-size 80 --duration 15

# 執行結果
bench_client start
pubs=1 subs=3 rate=10000/pub msg_size=80 duration=15s
==== bench result ====
Sent: 149987 msgs (9999.13 msg/s)
Recv: 449961 msgs (29997.40 msg/s)
Latency (ms): avg=1.56 p50=0.26 p90=0.68 p99=34.51
======================
```

可以看到 rust 性能又比前兩者(c++, go)更好，果然是新興強勢的語言。

## 結論

經歷了這一輪tcp socket以及不同語言的體驗，只能說有了AI，以往這些事花費的時間，可能現在的兩、三倍以上，但現在透過AI就能在短短的時間內體驗到，真的不得不感嘆AI的強大，但還是有一種強烈的空虛感跟不踏實的感覺，或許就是使用AI的代價，也或許是並沒有實際的付出，就得到成果的愧疚感，總之這次的體驗還算成功，接下來就剩最後一天了。
