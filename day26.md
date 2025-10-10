# 網路協議 tcp 體驗 2

今天就繼續看看 tcp 協議相關，能做什麼更多的應用。

## 今日目標

使用 rust 語言實作 tcp socket 的Server跟Client來跟昨天 GO 實作的做互聯。

## Rust 安裝

今天先以Windows環境下安裝 [rust](https://rust-lang.org/tools/install/)，Windows的環境下會需要 [Visual Studio C++ Build tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)，但我自己的電腦有安裝 `Visual Studio 2022` 裡面也有安裝VC++的build tool，所以安裝完就能開 cmd 確認是否安裝成功。

```sh
# 確認是否安裝成功
rustc --version

# Rust的套件管理工具
cargo --version
```

## Rust 新建專案

新建專案的部分，可以透過rust的cargo執行，並且開發rust我們使用 VS Code，Extension 記得安裝 `rust-analyzer`，以便於 rust 的開發。

```sh
cargo new tcp-sockets-rs --bin
cd tcp-sockets-rs
code .
```

### Server端

```rust
//  tcp-sockets-rs/src/bin/server.rs
use std::io::{BufRead, BufReader, Write};
use std::net::{TcpListener, TcpStream};
use std::thread;

fn handle_client(mut stream: TcpStream) -> std::io::Result<()> {
    let peer = stream.peer_addr().ok();
    let reader = BufReader::new(stream.try_clone()?);

    for line in reader.lines() {
        let line = line?;
        println!("📩 from {:?}: {}", peer, line);

        let reply = format!("Echo from server: {}\n", line);
        stream.write_all(reply.as_bytes())?;
        stream.flush()?;
    }

    println!("❌ client disconnected: {:?}", peer);
    Ok(())
}

fn main() -> std::io::Result<()> {
    let addr = "0.0.0.0:9000";
    let listener = TcpListener::bind(addr)?;
    println!("✅ TCP Server listening on {}", addr);

    for conn in listener.incoming() {
        match conn {
            Ok(stream) => {
                println!("🔗 new client: {:?}", stream.peer_addr().ok());
                thread::spawn(|| {
                    if let Err(e) = handle_client(stream) {
                        eprintln!("client error: {e}");
                    }
                });
            }
            Err(e) => eprintln!("accept error: {e}"),
        }
    }
    Ok(())
}

```

## Client端

```rust
// tcp-sockets-rs/src/bin/client.rs
use std::io::{self, BufRead, Read, Write};
use std::net::TcpStream;

fn main() -> std::io::Result<()> {
    let addr = "127.0.0.1:9000";
    let mut stream = TcpStream::connect(addr)?;
    println!("✅ Connected to {addr}. Type messages and press Enter.");

    let stdin = io::stdin();
    let mut input = String::new();

    loop {
        input.clear();
        print!("You: ");
        io::stdout().flush()?;
        if stdin.lock().read_line(&mut input)? == 0 {
            // EOF
            break;
        }

        stream.write_all(input.as_bytes())?;
        stream.flush()?;

        let mut buf = [0u8; 1024];
        let n = stream.read(&mut buf)?;
        if n == 0 {
            println!("\n❌ server closed connection");
            break;
        }
        print!("💬 Server reply: {}", String::from_utf8_lossy(&buf[..n]));
    }

    Ok(())
}

```

## 運行

```sh
# 視窗 A：啟動 server
cargo run --bin server

# 視窗 B：啟動 client
cargo run --bin client
```

```sh
# 這邊就會看到跟昨天一樣的結果
# 視窗 A： server
✅ TCP Server listening on 0.0.0.0:9000
🔗 new client: Some(127.0.0.1:60991)
📩 from Some(127.0.0.1:60991): aaa

# 視窗 B： client
✅ Connected to 127.0.0.1:9000. Type messages and press Enter.
You: aaa
💬 Server reply: Echo from server: aaa
```

```sh
# 這邊就使用昨天的go client來連線看看
go run client.go

# go Client
✅ Connected to server. Type messages and press Enter.
You: i'm go
💬 Server reply: Echo from server: i'm go

# rust server
🔗 new client: Some(127.0.0.1:55255)
📩 from Some(127.0.0.1:55255): i'm go
```

## 結論

今天就到體驗 tcp socket 能夠跨語言溝通的部分，明天再接續體驗其他部分囉。
