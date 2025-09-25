# io_uring 體驗

今天來體驗 `io_uring`，`io_uring` 是Linux內核中(在Linux版本 5.1 以後支援)實現非同步IO的高效接口。

## 環境準備

- 使用的環境依然是 `day6` 的ubuntu系統，因為ubuntu 22.04的linux內核版本是5.15.167.4是可以使用`io_uring`

## 性能測試

- 這邊要測試的是IO，首先想到的就是網路的部分，所以一樣拿tcp socket來測試，這邊就比較傳統的epoll(使用asio)跟新的io_uring的性能比對。

## 套件安裝

```sh
# 使用vcpkg install tool
vcpkg install asio liburing
```

## 代碼實作

- asio實作

```cpp
// net_asio_echo.cpp
#include <asio.hpp>
#include <iostream>
#include <memory>
#include <array>

using asio::ip::tcp;

struct Session : std::enable_shared_from_this<Session> {
  tcp::socket sock;
  std::array<char, 4096> buf{};

  explicit Session(tcp::socket s) : sock(std::move(s)) {}

  void start() { do_read(); }

private:
  void do_read() {
    auto self = shared_from_this();
    sock.async_read_some(asio::buffer(buf),
      [this, self](std::error_code ec, std::size_t n){
        if (ec) return;  // 連線關閉或錯誤
        do_write(n);
      });
  }

  void do_write(std::size_t n) {
    auto self = shared_from_this();
    asio::async_write(sock, asio::buffer(buf.data(), n),
      [this, self](std::error_code ec, std::size_t /*written*/){
        if (ec) return;
        do_read();
      });
  }
};

int main(int argc, char** argv) {
  int port = 9009;
  if (argc > 1) port = std::atoi(argv[1]);

  try {
    asio::io_context io;

    tcp::acceptor acc(io, tcp::endpoint(tcp::v4(), static_cast<unsigned short>(port)));

    // 遞迴接受新連線
    std::function<void()> do_accept = [&]{
      acc.async_accept([&](std::error_code ec, tcp::socket s){
        if (!ec) std::make_shared<Session>(std::move(s))->start();
        do_accept();
      });
    };

    do_accept();

    std::cerr << "Standalone Asio echo listening on port " << port << " ...\n";
    io.run();
  } catch (const std::exception& ex) {
    std::cerr << "Error: " << ex.what() << "\n";
    return 1;
  }
  return 0;
}
```

- io_uring實作

```cpp
// net_io_uring_echo.cpp
// net_io_uring_echo.cpp
#include <liburing.h>

#include <arpa/inet.h>
#include <netinet/tcp.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#include <fcntl.h>

#include <cerrno>
#include <cstring>
#include <cstdlib>
#include <iostream>
#include <vector>

static int set_nonblock(int fd, bool nb) {
  int flags = fcntl(fd, F_GETFL, 0);
  if (flags < 0) return -1;
  if (nb) flags |= O_NONBLOCK; else flags &= ~O_NONBLOCK;
  return fcntl(fd, F_SETFL, flags);
}

static void set_tcp_opts(int fd) {
  int yes = 1;
  ::setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes));
  ::setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &yes, sizeof(yes));
}

static int prep_listen(uint16_t port) {
  int fd = ::socket(AF_INET, SOCK_STREAM | SOCK_CLOEXEC, 0);
  if (fd < 0) {
    std::perror("socket");
    std::exit(1);
  }
  set_tcp_opts(fd);

  sockaddr_in addr{};
  addr.sin_family = AF_INET;
  addr.sin_addr.s_addr = htonl(INADDR_ANY);
  addr.sin_port = htons(port);

  if (::bind(fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0) {
    std::perror("bind");
    std::exit(1);
  }
  if (::listen(fd, 1024) < 0) {
    std::perror("listen");
    std::exit(1);
  }
  // 非阻塞不是必須，但可避免少數情況下的阻塞系統呼叫
  set_nonblock(fd, true);
  return fd;
}

// 每個連線的狀態
struct Conn {
  int fd;
  std::vector<char> buf; // 單一回合使用的收/發緩衝
  int send_off = 0;      // 已送出位移
  int send_len = 0;      // 本回合要送的總長度
};

// tag 編碼：
//  - accept 事件：tag == -1
//  - recv 事件：tag = 指向 Conn* 的純指標（最低位 0）
//  - send 事件：tag = (Conn* | 1)（最低位 1）
static inline void* make_send_tag(Conn* c) {
  return reinterpret_cast<void*>((reinterpret_cast<uintptr_t>(c) | 1ull));
}
static inline Conn* tag_to_conn(void* tag) {
  return reinterpret_cast<Conn*>(reinterpret_cast<uintptr_t>(tag) & ~1ull);
}
static inline bool is_send_tag(void* tag) {
  return (reinterpret_cast<uintptr_t>(tag) & 1ull) != 0ull;
}

int main(int argc, char** argv) {
  uint16_t port = 9009;
  if (argc > 1) port = static_cast<uint16_t>(std::atoi(argv[1]));

  int lfd = prep_listen(port);

  io_uring ring;
  // 佇列大小可依負載調整；4096 足夠一般測試
  if (io_uring_queue_init(4096, &ring, 0) < 0) {
    std::perror("io_uring_queue_init");
    return 1;
  }

  auto post_accept = [&](int lfd) {
    io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_accept(sqe, lfd, nullptr, nullptr, 0);
    io_uring_sqe_set_data(sqe, reinterpret_cast<void*>(static_cast<uintptr_t>(-1)));
  };

  auto post_recv = [&](Conn* c) {
    c->send_off = 0;
    c->send_len = 0;
    io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_recv(sqe, c->fd, c->buf.data(),
                       static_cast<unsigned>(c->buf.size()), 0);
    io_uring_sqe_set_data(sqe, c);
  };

  auto post_send_first = [&](Conn* c, int nbytes) {
    c->send_len = nbytes;
    c->send_off = 0;
    io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_send(sqe, c->fd, c->buf.data(),
                       static_cast<unsigned>(nbytes), 0);
    io_uring_sqe_set_data(sqe, make_send_tag(c));
  };

  auto post_send_more = [&](Conn* c) {
    // 補送尚未送完的部分
    int remain = c->send_len - c->send_off;
    io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_send(sqe, c->fd, c->buf.data() + c->send_off,
                       static_cast<unsigned>(remain), 0);
    io_uring_sqe_set_data(sqe, make_send_tag(c));
  };

  auto close_and_delete = [&](Conn* c) {
    if (!c) return;
    ::close(c->fd);
    delete c;
  };

  // 先掛第一個 accept
  post_accept(lfd);
  io_uring_submit(&ring);

  std::cerr << "io_uring echo listening on :" << port << "\n";

  while (true) {
    io_uring_cqe* cqe = nullptr;
    int rc = io_uring_wait_cqe(&ring, &cqe);
    if (rc < 0) {
      // 理論上不會常見；簡單忽略繼續
      continue;
    }

    void* tag = io_uring_cqe_get_data(cqe);
    int res = cqe->res;
    io_uring_cqe_seen(&ring, cqe);

    // ---- accept 完成 ----
    if (reinterpret_cast<uintptr_t>(tag) == static_cast<uintptr_t>(-1)) {
      // 無論成功與否，立即再掛下一個 accept 以持續接新連線
      post_accept(lfd);
      io_uring_submit(&ring);

      if (res < 0) {
        // accept 失敗（可能是 EAGAIN 等），略過
        continue;
      }

      int cfd = res;
      set_tcp_opts(cfd);
      set_nonblock(cfd, true);

      // 建立連線狀態並先排一個 recv
      Conn* c = new Conn{cfd, std::vector<char>(4096), 0, 0};
      post_recv(c);
      io_uring_submit(&ring);
      continue;
    }

    // ---- send 完成 ----
    if (is_send_tag(tag)) {
      Conn* c = tag_to_conn(tag);

      if (res <= 0) {
        // 對端關閉或錯誤
        close_and_delete(c);
        continue;
      }

      // 更新已送位移，若未送完則補送
      c->send_off += res;
      if (c->send_off < c->send_len) {
        post_send_more(c);
        io_uring_submit(&ring);
      } else {
        // 本回合完整 echo 完成，才再次排 recv
        post_recv(c);
        io_uring_submit(&ring);
      }
      continue;
    }

    // ---- recv 完成 ----
    Conn* c = static_cast<Conn*>(tag);
    if (res <= 0) {
      // 對端關閉或錯誤
      close_and_delete(c);
      continue;
    }

    // 把剛讀到的 res bytes 回送（可能需要多次 send）
    post_send_first(c, res);
    io_uring_submit(&ring);
  }

  //（理論上不會到這裡）
  io_uring_queue_exit(&ring);
  ::close(lfd);
  return 0;
}
```

- Client端實作

```cpp
// echo_client.cpp
#include <asio.hpp>
#include <iostream>
#include <vector>
#include <string>
#include <chrono>
#include <atomic>
#include <memory>
#include <random>
#include <mutex>
#include <algorithm>

using asio::ip::tcp;
using clk = std::chrono::steady_clock;

struct Args {
  std::string host = "127.0.0.1";
  int port = 9009;
  int connections = 1;
  int requests_per_conn = 10000;
  int size = 128;
  int inflight = 1;        // 每連線同時在飛請求數（管線深度）
  bool latency = false;    // 若 inflight==1 量測延遲分佈
};

Args parse(int argc, char**argv){
  Args a;
  for (int i=1;i<argc;i++){
    std::string s(argv[i]);
    auto next = [&]{ return std::string(argv[++i]); };
    if (s=="--host") a.host = next();
    else if (s=="--port") a.port = std::stoi(next());
    else if (s=="--conns") a.connections = std::stoi(next());
    else if (s=="--requests") a.requests_per_conn = std::stoi(next());
    else if (s=="--size") a.size = std::stoi(next());
    else if (s=="--inflight") a.inflight = std::stoi(next());
    else if (s=="--latency") a.latency = true;
    else if (s=="-h" || s=="--help"){
      std::cout <<
      "Usage: echo_client --host 127.0.0.1 --port 9009 "
      "--conns 8 --requests 100000 --size 128 --inflight 8 [--latency]\n";
      std::exit(0);
    }
  }
  return a;
}

struct LatencyStats {
  std::vector<double> samples_us;
  std::mutex m;
  void add(double us) {
    std::lock_guard<std::mutex> g(m);
    samples_us.push_back(us);
  }
  void report() {
    if (samples_us.empty()) return;
    std::sort(samples_us.begin(), samples_us.end());
    auto pct = [&](double p){
      size_t k = (size_t)((samples_us.size()-1) * p);
      return samples_us[k];
    };
    double sum=0; for (auto v: samples_us) sum += v;
    double avg = sum / samples_us.size();
    std::cout << "latency(us): avg=" << avg
              << " p50=" << pct(0.50)
              << " p95=" << pct(0.95)
              << " p99=" << pct(0.99) << "\n";
  }
};

struct Session : public std::enable_shared_from_this<Session> {
  asio::io_context& io;
  tcp::socket sock;
  std::vector<char> tx, rx;
  int remaining;           // 還要完成多少次 echo（每次 = 一次 send+recv）
  int inflight_target;     // 目標管線深度
  int inflight_now = 0;    // 目前在飛
  std::atomic<long long>& global_done;
  LatencyStats* lat;       // inflight==1 時才記錄
  std::vector<clk::time_point> t0q; // 每個inflight槽位的時間戳
  int conn_id;

  Session(asio::io_context& io, int id,
          const Args& a, std::atomic<long long>& done, LatencyStats* lat)
  : io(io),
    sock(io),
    tx(a.size, 0x5A),
    rx(a.size),
    remaining(a.requests_per_conn),
    inflight_target(std::max(1, a.inflight)),
    global_done(done),
    lat(lat),
    t0q(inflight_target),
    conn_id(id)
  {
    // 填入些隨機內容（非必要）
    std::mt19937 rng(id*1337u);
    for (int i=0;i<a.size;i++) tx[i] = char(rng()%256);
  }

  void start(const tcp::endpoint& ep) {
    auto self = shared_from_this();
    sock.async_connect(ep, [this,self](std::error_code ec){
      if (ec) { std::cerr << "connect error: " << ec.message() << "\n"; return; }
      // 啟動到目標管線深度
      int n = std::min(inflight_target, remaining);
      for (int i=0;i<n;i++) issue_one(i);
    });
  }

  void issue_one(int slot) {
    if (remaining <= 0) return;
    remaining--; inflight_now++;
    t0q[slot] = clk::now();

    auto self = shared_from_this();
    asio::async_write(sock, asio::buffer(tx),
      [this,self,slot](std::error_code ec, std::size_t){
        if (ec) { close(); return; }
        // 等 echo 回來
        asio::async_read(sock, asio::buffer(rx),
          [this,self,slot](std::error_code ec, std::size_t n){
            if (ec || n != rx.size()) { close(); return; }
            auto t1 = clk::now();
            inflight_now--;
            global_done++;

            if (lat && inflight_target==1) {
              double us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0q[slot]).count();
              lat->add(us);
            }

            // 繼續補到目標管線
            if (remaining > 0) {
              issue_one(slot);
            } else if (inflight_now==0) {
              // 本連線完成
              close();
            }
          });
      });
  }

  void close() {
    std::error_code ec;
    sock.shutdown(tcp::socket::shutdown_both, ec);
    sock.close(ec);
  }
};

int main(int argc, char** argv) {
  Args a = parse(argc, argv);
  std::cout << "Target: " << a.host << ":" << a.port
            << " conns=" << a.connections
            << " req/conn=" << a.requests_per_conn
            << " size=" << a.size
            << " inflight=" << a.inflight
            << (a.latency ? " (latency on)" : "") << "\n";

  try{
    asio::io_context io;
    tcp::resolver res(io);
    auto eps = res.resolve(a.host, std::to_string(a.port));
    tcp::endpoint ep = *eps.begin();

    std::atomic<long long> done{0};
    LatencyStats lat;
    LatencyStats* latp = (a.inflight==1 && a.latency) ? &lat : nullptr;

    auto t0 = clk::now();
    std::vector<std::shared_ptr<Session>> sessions;
    sessions.reserve(a.connections);
    for (int i=0;i<a.connections;i++){
      auto s = std::make_shared<Session>(io, i, a, done, latp);
      s->start(ep);
      sessions.push_back(std::move(s));
    }

    // 跑到所有請求完成
    // 用額外 thread 跑 io 也行，這裡用 run_for loop 防止卡死
    while (done < 1LL * a.connections * a.requests_per_conn) {
      io.run_for(std::chrono::milliseconds(50));
    }
    auto t1 = clk::now();

    double secs = std::chrono::duration<double>(t1 - t0).count();
    long long total = 1LL * a.connections * a.requests_per_conn;
    double rps = total / secs;
    double mibps = (double)total * a.size / (1024.0*1024.0) / secs;

    std::cout << "Completed " << total << " reqs in " << secs*1000.0 << " ms\n";
    std::cout << "Throughput: " << (long long)rps << " req/s, "
              << mibps << " MiB/s\n";

    if (latp) latp->report();
  } catch (const std::exception& ex) {
    std::cerr << "Error: " << ex.what() << "\n";
    return 1;
  }
  return 0;
}
```

- CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(io_bench LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

# ---- Asio（Standalone）----
find_package(asio CONFIG REQUIRED)     # 由 vcpkg 提供
add_executable(net_asio_echo net_asio_echo.cpp)
target_link_libraries(net_asio_echo PRIVATE asio::asio)  # 官方 target

# ---- liburing（建議走 pkg-config）----
include(FindPkgConfig)
pkg_check_modules(liburing REQUIRED IMPORTED_TARGET GLOBAL liburing>=2.0)
add_executable(net_io_uring_echo net_io_uring_echo.cpp)
target_link_libraries(net_io_uring_echo PRIVATE PkgConfig::liburing)

add_executable(echo_client echo_client.cpp)
target_link_libraries(echo_client PRIVATE asio::asio)
```

- 編譯及運行，編譯一樣透過vscode上的cmake來建置(F7)

```sh
./net_io_uring_echo 9009

# 在開一個shell
./echo_client --host 127.0.0.1 --port 9009 \
  --conns 16 --requests 200000 --size 128 --inflight 8

./net_asio_echo 9010

# 在開一個shell
./echo_client --host 127.0.0.1 --port 9010 \
  --conns 16 --requests 200000 --size 128 --inflight 8
```

## 測試結果

從下面的結果看到其實傳統的epoll性能竟然還比新的io_uring還好。

```sh
# asio 的結果
./echo_client --host 127.0.0.1 --port 9010 \
  --conns 16 --requests 200000 --size 128 --inflight 8
Target: 127.0.0.1:9010 conns=16 req/conn=200000 size=128 inflight=8
Completed 3200000 reqs in 21940.3 ms
Throughput: 145850 req/s, 17.804 MiB/s

# io_uring 的結果
./echo_client --host 127.0.0.1 --port 9009   --conns 16 --requests 200000 --size 128 --inflight 8
Target: 127.0.0.1:9009 conns=16 req/conn=200000 size=128 inflight=8
Completed 3200000 reqs in 25203.3 ms
Throughput: 126967 req/s, 15.499 MiB/s
```

## 明日接續

明天來繼續看看為什麼實際上io_uring跑出來的性能比epoll還差。
