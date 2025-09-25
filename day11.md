# io_uring 體驗 2

今天就接著來看看為什麼io_uring比asio的性能差。

## io_uring 運作機制

- io_uring 的目的是簡化傳統I/O在使用者空間(Userspace)跟系統核心(Kernel)間過多的溝通，並提供真正的非同化的操作，以期望提供更高的性能。
- io_uring的設計中為了提升Userspace與Kernel間的溝通，所以使用共享記憶體的方式當作溝通的機制，裡面有兩個佇列能幫助整個溝通過程
  - 提交佇列 (Submission Queue，SQ): 應用程式(Userspace)作為生產者將 I/O 請求(Submission Queue Entry，SQE)填入，核心(Kernel)作為消費者讀取並執行
  - 完成佇列 (Completion Queue，CQ): 核心(Kernel)作為生產者將已完成的 I/O 操作結果(Completion Queue Entry，CQE)填入，應用程式(Userspace)作為消費者讀取結果

## 代碼調整 - asio

- 昨天的 `net_io_uring_echo.cpp` 實際上是單執行緒，今天就另外弄一個多執行緒的版本，不過這邊先從asio的多執行版本來看看效果

```cpp
//  net_asio_echo_mt.cpp
#include <asio.hpp>
#include <iostream>
#include <memory>
#include <array>
#include <thread>
#include <vector>

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
      [this,self](std::error_code ec, std::size_t n){
        if (ec) return;
        do_write(n);
      });
  }
  void do_write(std::size_t n) {
    auto self = shared_from_this();
    asio::async_write(sock, asio::buffer(buf.data(), n),
      [this,self](std::error_code ec, std::size_t){
        if (ec) return;
        do_read();
      });
  }
};

int main(int argc, char** argv) {
  int port = 9010;
  int threads = std::max(1u, std::thread::hardware_concurrency());
  if (argc > 1) port = std::atoi(argv[1]);
  if (argc > 2) threads = std::atoi(argv[2]);

  try {
    asio::io_context io;

    tcp::acceptor acc(io, tcp::endpoint(tcp::v4(), static_cast<unsigned short>(port)));
    std::function<void()> do_accept;
    do_accept = [&]{
      acc.async_accept([&](std::error_code ec, tcp::socket s){
        if (!ec) std::make_shared<Session>(std::move(s))->start();
        do_accept();
      });
    };
    do_accept();

    std::cerr << "Asio MT echo listening on port " << port
              << " with " << threads << " threads...\n";

    // 啟動 thread pool 跑 io_context
    std::vector<std::thread> pool;
    pool.reserve(threads);
    for (int i=0;i<threads;i++) pool.emplace_back([&]{ io.run(); });
    for (auto& t: pool) t.join();
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

add_executable(net_asio_echo_mt net_asio_echo_mt.cpp)
target_link_libraries(net_asio_echo_mt PRIVATE asio::asio)  # 官方 target
find_package(Threads REQUIRED)
target_link_libraries(net_asio_echo_mt PRIVATE Threads::Threads)
```

- 編譯與運行

```sh
./net_asio_echo_mt 9010 8

# 在開一個shell
./echo_client --host 127.0.0.1 --port 9010 \
  --conns 16 --requests 200000 --size 128 --inflight 8
```

- 測試結果: 跟昨天跑出來的21940.3 ms，總體大概是提升 8.8% ((19996 ms - 21940 ms) / 21940 ms)，多執行緒的這部在asio是得到驗證的

```sh
./echo_client --host 127.0.0.1 --port 9010 \
  --conns 16 --requests 200000 --size 128 --inflight 8
Target: 127.0.0.1:9010 conns=16 req/conn=200000 size=128 inflight=8
Completed 3200000 reqs in 19996.7 ms
Throughput: 160026 req/s, 19.5344 MiB/s
```

## 代碼調整 - io_uring

```cpp
// net_io_uring_echo_mt.cpp
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
#include <thread>
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
static int prep_listen_reuseport(uint16_t port) {
  int fd = ::socket(AF_INET, SOCK_STREAM | SOCK_CLOEXEC, 0);
  if (fd < 0) { std::perror("socket"); std::exit(1); }
  set_tcp_opts(fd);
  int yes = 1;
  if (::setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &yes, sizeof(yes)) < 0) {
    std::perror("setsockopt SO_REUSEPORT"); std::exit(1);
  }
  sockaddr_in addr{}; addr.sin_family=AF_INET;
  addr.sin_addr.s_addr=htonl(INADDR_ANY);
  addr.sin_port=htons(port);
  if (::bind(fd, (sockaddr*)&addr, sizeof(addr)) < 0) { std::perror("bind"); std::exit(1); }
  if (::listen(fd, 1024) < 0) { std::perror("listen"); std::exit(1); }
  set_nonblock(fd, true);
  return fd;
}

struct Conn {
  int fd;
  std::vector<char> buf;
  int send_off=0, send_len=0;
};

static inline void* send_tag(Conn* c){ return (void*)((uintptr_t)c | 1ull); }
static inline Conn* tag_conn(void* tag){ return (Conn*)((uintptr_t)tag & ~1ull); }
static inline bool is_send(void* tag){ return ((uintptr_t)tag & 1ull)!=0ull; }

void worker_loop(uint16_t port, int buf_sz) {
  int lfd = prep_listen_reuseport(port);

  io_uring ring;
  if (io_uring_queue_init(4096, &ring, 0) < 0) { std::perror("io_uring_queue_init"); std::exit(1); }

  auto post_accept = [&](){
    io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_accept(sqe, lfd, nullptr, nullptr, 0);
    io_uring_sqe_set_data(sqe, (void*)-1);
  };

  auto post_recv = [&](Conn* c){
    c->send_off = c->send_len = 0;
    io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_recv(sqe, c->fd, c->buf.data(), (unsigned)c->buf.size(), 0);
    io_uring_sqe_set_data(sqe, c);
  };

  auto post_send_first = [&](Conn* c, int n){
    c->send_len = n; c->send_off = 0;
    io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_send(sqe, c->fd, c->buf.data(), (unsigned)n, 0);
    io_uring_sqe_set_data(sqe, send_tag(c));
  };
  auto post_send_more = [&](Conn* c){
    int remain = c->send_len - c->send_off;
    io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_send(sqe, c->fd, c->buf.data()+c->send_off, (unsigned)remain, 0);
    io_uring_sqe_set_data(sqe, send_tag(c));
  };

  auto close_conn = [&](Conn* c){ if(!c) return; ::close(c->fd); delete c; };

  // 初始先掛 1 個 accept
  post_accept();
  io_uring_submit(&ring);

  // 每個 worker thread 獨立跑自己的事件迴圈
  while (true) {
    // 批次處理完成（比單一 wait 更省 syscall）
    io_uring_cqe* cqes[64];
    unsigned got = io_uring_peek_batch_cqe(&ring, cqes, 64);
    if (got == 0) {
      io_uring_submit_and_wait(&ring, 1);
      got = io_uring_peek_batch_cqe(&ring, cqes, 64);
    }
    for (unsigned i=0;i<got;i++){
      io_uring_cqe* cqe = cqes[i];
      void* tag = io_uring_cqe_get_data(cqe);
      int res = cqe->res;
      io_uring_cqe_seen(&ring, cqe);

      if (tag == (void*)-1) {  // accept 完成
        // 立刻再掛下一個 accept（保持接力）
        post_accept();

        if (res >= 0) {
          int cfd = res;
          set_tcp_opts(cfd);
          set_nonblock(cfd, true);
          Conn* c = new Conn{cfd, std::vector<char>(buf_sz), 0, 0};
          post_recv(c);
        }
        continue;
      }

      if (is_send(tag)) {      // send 完成
        Conn* c = tag_conn(tag);
        if (res <= 0) { close_conn(c); continue; }
        c->send_off += res;
        if (c->send_off < c->send_len) {
          post_send_more(c);
        } else {
          post_recv(c);
        }
        continue;
      }

      // recv 完成
      Conn* c = (Conn*)tag;
      if (res <= 0) { close_conn(c); continue; }
      post_send_first(c, res);
    }

    // 把這一輪產生的 SQE 一次送出
    io_uring_submit(&ring);
  }

  //（正常不會走到）
  io_uring_queue_exit(&ring);
  ::close(lfd);
}

int main(int argc, char** argv) {
  int port = 9009;
  int threads = std::max(1u, std::thread::hardware_concurrency());
  int buf_sz = 4096;
  if (argc > 1) port = std::atoi(argv[1]);
  if (argc > 2) threads = std::atoi(argv[2]);
  if (argc > 3) buf_sz = std::atoi(argv[3]);

  std::cerr << "io_uring MT echo listening on port " << port
            << " with " << threads << " threads (SO_REUSEPORT)...\n";

  std::vector<std::thread> pool;
  pool.reserve(threads);
  for (int i=0;i<threads;i++) pool.emplace_back(worker_loop, port, buf_sz);
  for (auto& t: pool) t.join();
  return 0;
}
```

- CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(io_bench LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

# ---- liburing（建議走 pkg-config）----
include(FindPkgConfig)
pkg_check_modules(liburing REQUIRED IMPORTED_TARGET GLOBAL liburing>=2.0)
add_executable(net_io_uring_echo_mt net_io_uring_echo_mt.cpp)
target_link_libraries(net_io_uring_echo_mt PRIVATE PkgConfig::liburing)
target_link_libraries(net_io_uring_echo_mt PRIVATE Threads::Threads)
```

- 編譯與運行

```sh
./net_io_uring_echo_mt 9009 8 4096

# 在開一個shell
./echo_client --host 127.0.0.1 --port 9009 \
  --conns 16 --requests 200000 --size 128 --inflight 8
```

- 測試結果: 多執行緒版本的io_uring反而比單執行版本還久

```sh
./echo_client --host 127.0.0.1 --port 9009 \
  --conns 16 --requests 200000 --size 128 --inflight 8
Target: 127.0.0.1:9009 conns=16 req/conn=200000 size=128 inflight=8
Completed 3200000 reqs in 26816.7 ms
Throughput: 119328 req/s, 14.5665 MiB/s
```

## 今日結論

- io_uring 目前使用上有些步驟比asio的部分要繁瑣的多，以及 io_uring還有很多機制，我也尚未搞明白，這導致目前無法測試出其性能的潛力，那這次的體驗就先在這告一段落了，之後再來好好更深入了解這個套件。
