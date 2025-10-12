# 網路協議 tcp 體驗 3

今天依然延續 tcp socket 的探索，但今天會改用 C++ 來實作。

## 今日目標

今天目標是做一個tcp socket 消息轉發器，透過 telnet 這個工具當這個轉發器兩邊的發送者跟接收者，來確認消息轉發器的性能。

## 環境準備

今天一樣使用 day6 的 ubuntu 22.04開發環境。

容器啟動後，進容器執行下面的指令，今天會使用asio。

```sh
apt-get install pkg-config
vcpkg install asio
```

## 專案結構

```sh
tcp-forwarder/
├─ CMakeLists.txt
└─ src/
   ├─ main.cpp
   └─ TcpServer.hpp
```

### CMakeLists.txt

```camke
cmake_minimum_required(VERSION 3.16)
project(tcp_forwarder CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ---- 專案檔案 ----
add_executable(tcp_forwarder
  src/main.cpp
)

# ---- vcpkg: asio ----
find_package(asio CONFIG REQUIRED)
target_link_libraries(tcp_forwarder PRIVATE asio::asio)

# ---- Linux threads (-pthread) ----
find_package(Threads REQUIRED)
target_link_libraries(tcp_forwarder PRIVATE Threads::Threads)
```

### src/TcpServer.hpp

```cpp
#pragma once
#include <asio.hpp>
#include <atomic>
#include <functional>
#include <iostream>
#include <memory>
#include <mutex>
#include <set>
#include <string>
#include <thread>
#include <vector>

class TcpServer : public std::enable_shared_from_this<TcpServer> {
public:
    using OnOpenCB  = std::function<void(const asio::ip::tcp::endpoint&)>;
    using OnCloseCB = std::function<void(const asio::ip::tcp::endpoint&)>;
    using OnMsgCB   = std::function<void(const std::string&, const asio::ip::tcp::endpoint&)>;

    TcpServer(asio::io_context& ioc, uint16_t port, size_t worker_threads = std::thread::hardware_concurrency())
        : ioc_(ioc),
          acceptor_(ioc, asio::ip::tcp::endpoint(asio::ip::tcp::v4(), port)),
          port_(port),
          worker_threads_(worker_threads ? worker_threads : 1)
    {}

    void set_on_open(OnOpenCB cb)  { on_open_  = std::move(cb); }
    void set_on_close(OnCloseCB cb){ on_close_ = std::move(cb); }
    void set_on_msg(OnMsgCB cb)    { on_msg_   = std::move(cb); }

    void start() {
        // 啟動 worker threads（若外部沒有自行跑 run()）
        for (size_t i = 0; i < worker_threads_; ++i) {
            workers_.emplace_back([this]{ ioc_.run(); });
        }
        do_accept();
        std::cout << "✅ Server listening on port " << port_ << "\n";
    }

    void stop() {
        ioc_.post([self=shared_from_this()]{
            std::lock_guard<std::mutex> lk(self->mtx_);
            for (auto& s : self->sessions_) {
                if (s && s->socket.is_open()) {
                    asio::error_code ec;
                    s->socket.close(ec);
                }
            }
            self->sessions_.clear();
        });
        acceptor_.close();
        for (auto& t : workers_) if (t.joinable()) t.join();
    }

    // 對所有目前連線的 client 非阻塞推送
    // 回傳實際成功寫入的 client 數量（可用於統計）
    size_t broadcast(const std::string& msg) {
        std::vector<std::shared_ptr<Session>> snapshot;
        {
            std::lock_guard<std::mutex> lk(mtx_);
            snapshot.reserve(sessions_.size());
            for (auto& s : sessions_) if (auto sp = s) snapshot.push_back(sp);
        }

        size_t sent = 0;
        for (auto& s : snapshot) {
            if (!s->socket.is_open()) continue;
            auto buf = std::make_shared<std::string>(msg); // 持有直到 async_write 完成
            asio::async_write(s->socket, asio::buffer(*buf),
                [buf, s](std::error_code ec, std::size_t /*n*/) {
                    if (ec) {
                        // 寫失敗，關閉連線（稍後由讀/寫callback清理）
                        asio::error_code ignored;
                        s->socket.close(ignored);
                    }
                }
            );
            ++sent;
        }
        return sent;
    }

    size_t connection_count() const {
        std::lock_guard<std::mutex> lk(mtx_);
        return sessions_.size();
    }

private:
    struct Session : public std::enable_shared_from_this<Session> {
        explicit Session(asio::ip::tcp::socket sock) : socket(std::move(sock)) {}
        asio::ip::tcp::socket socket;
        asio::streambuf buf;  // 行分隔讀取
    };

    void do_accept() {
        acceptor_.async_accept([self = shared_from_this()](std::error_code ec, asio::ip::tcp::socket sock){
            if (!ec) {
                auto ep = sock.remote_endpoint();
                auto sess = std::make_shared<Session>(std::move(sock));

                {
                    std::lock_guard<std::mutex> lk(self->mtx_);
                    self->sessions_.insert(sess);
                }

                if (self->on_open_) self->on_open_(ep);
                self->do_read(sess); // 啟動讀取

                // 繼續接受下一個
                self->do_accept();
            } else {
                if (self->acceptor_.is_open()) {
                    // 短暫錯誤，繼續 accept
                    self->do_accept();
                }
            }
        });
    }

    void do_read(std::shared_ptr<Session> s) {
        asio::async_read_until(s->socket, s->buf, '\n',
            [self = shared_from_this(), s](std::error_code ec, std::size_t n){
                auto ep = s->socket.is_open() ? s->socket.remote_endpoint() : asio::ip::tcp::endpoint{};
                if (!ec) {
                    std::istream is(&s->buf);
                    std::string line;
                    line.reserve(n);
                    std::getline(is, line); // 移除 '\n'
                    if (self->on_msg_) self->on_msg_(line, ep);
                    self->do_read(s); // 繼續下一行
                } else {
                    // 連線關閉或錯誤，清理
                    {
                        std::lock_guard<std::mutex> lk(self->mtx_);
                        self->sessions_.erase(s);
                    }
                    if (self->on_close_) self->on_close_(ep);
                    asio::error_code ignored;
                    s->socket.close(ignored);
                }
            }
        );
    }

private:
    asio::io_context& ioc_;
    asio::ip::tcp::acceptor acceptor_;
    uint16_t port_{};
    size_t worker_threads_{1};
    mutable std::mutex mtx_;
    std::set<std::shared_ptr<Session>> sessions_;
    std::vector<std::thread> workers_;

    OnOpenCB  on_open_;
    OnCloseCB on_close_;
    OnMsgCB   on_msg_;
};
```

### src/main.cpp

```cpp
#include "TcpServer.hpp"
#include <chrono>
#include <csignal>
#include <atomic>
#include <iostream>
#include <asio/signal_set.hpp>
#include <asio/steady_timer.hpp>

int main(int argc, char** argv) {
    std::cout << std::unitbuf; // 讓 log 即時輸出

    uint16_t recv_port = 7001;
    uint16_t send_port = 7002;
    if (argc >= 2) recv_port = static_cast<uint16_t>(std::stoi(argv[1]));
    if (argc >= 3) send_port = static_cast<uint16_t>(std::stoi(argv[2]));

    asio::io_context ioc;

    // ✅ 關鍵：建立 work guard，防止 ioc.run() 立刻返回
    auto guard = asio::make_work_guard(ioc);

    // 訊號攔截
    asio::signal_set signals(ioc, SIGINT, SIGTERM);

    auto recvSrv = std::make_shared<TcpServer>(ioc, recv_port);
    auto sendSrv = std::make_shared<TcpServer>(ioc, send_port);

    std::atomic<uint64_t> recv_count{0};
    std::atomic<uint64_t> send_count{0};

    recvSrv->set_on_open([&](const asio::ip::tcp::endpoint& ep){
        std::cout << "🔗 RecvServer OPEN: " << ep << "\n";
    });
    recvSrv->set_on_close([&](const asio::ip::tcp::endpoint& ep){
        std::cout << "❌ RecvServer CLOSE: " << ep << "\n";
    });
    recvSrv->set_on_msg([&](const std::string& msg, const asio::ip::tcp::endpoint&){
        ++recv_count;
        auto line = msg + "\n";
        auto sent_to = sendSrv->broadcast(line);
        send_count += sent_to;
    });

    sendSrv->set_on_open([&](const asio::ip::tcp::endpoint& ep){
        std::cout << "🔗 SendServer OPEN: " << ep << "\n";
    });
    sendSrv->set_on_close([&](const asio::ip::tcp::endpoint& ep){
        std::cout << "❌ SendServer CLOSE: " << ep << "\n";
    });
    sendSrv->set_on_msg([&](const std::string& msg, const asio::ip::tcp::endpoint& ep){
        std::cout << "📩 SendServer got (unexpected) from " << ep << ": " << msg << "\n";
    });

    recvSrv->start();
    sendSrv->start();

    // 可取消的統計計時器
    using namespace std::chrono;
    auto stats_timer = std::make_shared<asio::steady_timer>(ioc);
    auto last = steady_clock::now();
    uint64_t last_recv = 0, last_send = 0;

    std::function<void()> arm_stats = [&]() {
        stats_timer->expires_after(seconds(60));
        stats_timer->async_wait([&](const std::error_code& ec){
            if (ec == asio::error::operation_aborted) return;

            auto now = steady_clock::now();
            auto sec = duration_cast<seconds>(now - last).count();
            last = now;

            uint64_t cr = recv_count.load();
            uint64_t cs = send_count.load();
            uint64_t dr = cr - last_recv;
            uint64_t ds = cs - last_send;
            last_recv = cr; last_send = cs;

            double rps = sec > 0 ? static_cast<double>(dr) / sec : 0.0;
            double sps = sec > 0 ? static_cast<double>(ds) / sec : 0.0;

            std::cout << "⏱️  Interval " << sec << "s | "
                      << "Recv: " << dr << " (" << rps << "/s), "
                      << "Sent: " << ds << " (" << sps << "/s) | "
                      << "SendConn=" << sendSrv->connection_count()
                      << " RecvConn=" << recvSrv->connection_count()
                      << "\n";

            arm_stats();
        });
    };
    arm_stats();

    // 收到訊號 → 立即關閉（取消 timer、釋放 guard、停止 ioc）
    signals.async_wait([&](const std::error_code& ec, int sig){
        if (!ec) {
            std::cout << "\nSignal " << sig << " received, shutting down now...\n";
            stats_timer->cancel();
            guard.reset();   // 解除保活
            ioc.stop();      // 讓 run() 立即返回
        }
    });

    // 主執行緒也參與事件循環；有 guard 不會提早返回
    ioc.run();

    // 收尾
    recvSrv->stop();
    sendSrv->stop();
    std::cout << "Bye.\n";
    return 0;
}
```

## 建置與運行

```sh
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=/opt/vcpkg/scripts/buildsystems/vcpkg.cmake
make

# 啟動轉發器
./tcp_forwarder 7001 7002
```

- 安裝 telnet

```sh
apt install -y telnet
```

```sh
# 終端1: 接收端
telnet 127.0.0.1 7002

# 終端2: 推送端 
telnet 127.0.0.1 7001

# 終端2: 推送端
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
hihi

# 終端1: 接收端
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
hihi

# 終端3: 轉發Servevr
✅ Server listening on port 7001
✅ Server listening on port 7002
🔗 SendServer OPEN: 127.0.0.1:60768
⏱️  Interval 60s | Recv: 0 (0/s), Sent: 0 (0/s) | SendConn=1 RecvConn=0
🔗 RecvServer OPEN: 127.0.0.1:52542
⏱️  Interval 60s | Recv: 1 (0.0166667/s), Sent: 1 (0.0166667/s) | SendConn=1 RecvConn=1
```

## 衍生測試

這邊我想更了解轉發器的性能指數，所以會透過下面的工具，來繼續測試

```sh
# 安裝相關的工具
apt install -y netcat-openbsd pv
```

### 測試推送吞吐量

1. 啟動轉發器

```sh
./tcp_forwarder 7001 7002
```

2. 啟動接收端

```sh
nc 127.0.0.1 7002
```

3. 啟動一個高頻發送端

```sh
yes "msg $(date +%s%N)" | pv -L 1m | nc 127.0.0.1 7001

# 這邊能看到每秒大概推送 200KB，在限速1MB/s的場景下
3.51MiB 0:00:08 [ 225KiB/s]
```

4. 從轉發器日誌看性能

```sh
🔗 SendServer OPEN: 127.0.0.1:38588
🔗 RecvServer OPEN: 127.0.0.1:53314
⏱️  Interval 60s | Recv: 51511 (858.517/s), Sent: 51510 (858.5/s) | SendConn=1 RecvConn=1
⏱️  Interval 60s | Recv: 574156 (9569.27/s), Sent: 574156 (9569.27/s) | SendConn=1 RecvConn=1
```

### 測試推送吞吐量 2

前面的測試是透過額外的工具(yes跟pv)來進行，但真實的測試還是從實際的客戶端代碼，所以在剛剛的專案中加入以下的檔案

#### /src/bench_client.cpp

```cpp
#include <asio.hpp>
#include <asio/steady_timer.hpp>
#include <atomic>
#include <chrono>
#include <csignal>
#include <iostream>
#include <mutex>
#include <random>
#include <string>
#include <thread>
#include <vector>
#include <unordered_map>

using asio::ip::tcp;
using Clock = std::chrono::steady_clock;

struct Args {
    std::string pub_host = "127.0.0.1"; // 連 RecvServer
    uint16_t    pub_port = 7001;
    std::string sub_host = "127.0.0.1"; // 連 SendServer
    uint16_t    sub_port = 7002;
    int         pubs = 1;
    int         subs = 1;
    int         rate = 1000;        // 每條發送連線 每秒幾則
    int         msg_size = 64;      // bytes（含換行）
    int         duration = 10;      // 秒
    int         threads = (int)std::thread::hardware_concurrency();
};

static std::atomic<bool> g_stop{false};

static uint64_t now_ns() {
    return (uint64_t)std::chrono::duration_cast<std::chrono::nanoseconds>(
        Clock::now().time_since_epoch()).count();
}

static void parse_int(char* v, int& out){ out = std::max(0, std::atoi(v)); }
static void parse_u16(char* v, uint16_t& out){ out = (uint16_t)std::atoi(v); }

static Args parse_args(int argc, char** argv) {
    Args a;
    for (int i=1;i<argc;i++) {
        std::string k = argv[i];
        auto need = [&](int more){ return i+more<argc; };
        if (k=="--pub-host" && need(1)) a.pub_host = argv[++i];
        else if (k=="--pub-port" && need(1)) parse_u16(argv[++i], a.pub_port);
        else if (k=="--sub-host" && need(1)) a.sub_host = argv[++i];
        else if (k=="--sub-port" && need(1)) parse_u16(argv[++i], a.sub_port);
        else if (k=="--pub" && need(1)) parse_int(argv[++i], a.pubs);
        else if (k=="--sub" && need(1)) parse_int(argv[++i], a.subs);
        else if (k=="--rate" && need(1)) parse_int(argv[++i], a.rate);
        else if (k=="--msg-size" && need(1)) parse_int(argv[++i], a.msg_size);
        else if (k=="--duration" && need(1)) parse_int(argv[++i], a.duration);
        else if (k=="--threads" && need(1)) parse_int(argv[++i], a.threads);
        else if (k=="-h" || k=="--help") {
            std::cout <<
            "bench_client options:\n"
            "  --pub-host HOST     [default 127.0.0.1]\n"
            "  --pub-port PORT     [default 7001]  (connect RecvServer)\n"
            "  --sub-host HOST     [default 127.0.0.1]\n"
            "  --sub-port PORT     [default 7002]  (connect SendServer)\n"
            "  --pub N             publishers count [default 1]\n"
            "  --sub M             subscribers count [default 1]\n"
            "  --rate R            msgs/sec per publisher [default 1000]\n"
            "  --msg-size S        bytes per message incl. '\\n' [default 64]\n"
            "  --duration D        seconds [default 10]\n"
            "  --threads T         io_context threads [default hw]\n";
            std::exit(0);
        }
    }
    if (a.msg_size < 16) a.msg_size = 16; // 至少容納 header
    if (a.pubs < 0) a.pubs = 0;
    if (a.subs < 0) a.subs = 0;
    if (a.threads <= 0) a.threads = 1;
    return a;
}

struct LatencyStats {
    std::mutex mu;
    std::vector<double> samples_ms; // 收集樣本（上限以避免過大記憶體）
    size_t cap = 200000;

    void add(double ms) {
        std::lock_guard<std::mutex> lk(mu);
        if (samples_ms.size() < cap) samples_ms.push_back(ms);
    }
    static double perc(std::vector<double>& v, double p) {
        if (v.empty()) return 0.0;
        size_t idx = (size_t)std::clamp(p * (v.size()-1), 0.0, (double)(v.size()-1));
        std::nth_element(v.begin(), v.begin()+idx, v.end());
        return v[idx];
    }
    void print() {
        std::vector<double> v;
        {
            std::lock_guard<std::mutex> lk(mu);
            v = samples_ms;
        }
        if (v.empty()) { std::cout << "latency: no samples\n"; return; }
        // 平均
        double sum = 0; for (double x: v) sum += x;
        double avg = sum / v.size();
        // p50 p90 p99
        double p50 = perc(v, 0.50);
        double p90 = perc(v, 0.90);
        double p99 = perc(v, 0.99);
        std::cout << "Latency (ms): avg=" << avg
                  << " p50=" << p50
                  << " p90=" << p90
                  << " p99=" << p99 << "\n";
    }
};

struct Shared {
    std::atomic<uint64_t> sent{0};
    std::atomic<uint64_t> received{0};
    LatencyStats lat;
};

// Publisher：固定頻率送訊息（行分隔），訊息格式：pubId,seq,ns,<padding>\n
struct Publisher : public std::enable_shared_from_this<Publisher> {
    asio::io_context& ioc;
    tcp::socket sock;
    asio::steady_timer tick;
    std::string host; uint16_t port;
    int pub_id;
    int rate;      // msgs/sec
    int msg_size;  // bytes incl '\n'
    uint64_t seq{0};
    Shared& shared;

    Publisher(asio::io_context& io, std::string h, uint16_t p, int id, int r, int sz, Shared& g)
        : ioc(io), sock(io), tick(io), host(std::move(h)), port(p),
          pub_id(id), rate(r), msg_size(sz), shared(g) {}

    void start() {
        tcp::resolver res(ioc);
        auto eps = res.resolve(host, std::to_string(port));
        asio::async_connect(sock, eps, [self=shared_from_this()](std::error_code ec, const tcp::endpoint&){
            if (ec) { std::cerr << "Publisher connect error: " << ec.message() << "\n"; return; }
            self->schedule();
        });
    }

    void schedule() {
        if (g_stop.load()) return;
        using namespace std::chrono;
        auto interval = rate > 0 ? std::chrono::nanoseconds(1'000'000'000LL / rate)
                                 : std::chrono::nanoseconds(0);
        tick.expires_after(interval);
        tick.async_wait([self=shared_from_this()](std::error_code ec){
            if (!ec) self->send_one();
        });
    }

    void send_one() {
        if (g_stop.load()) return;
        ++seq;
        uint64_t ts = now_ns();
        // 構造內容：pubId,seq,ts,\n + padding
        std::string msg = std::to_string(pub_id) + "," + std::to_string(seq) + "," + std::to_string(ts) + ",";
        if ((int)msg.size() + 1 < msg_size) msg.resize(msg_size - 1, 'x');
        msg.push_back('\n');

        auto buf = std::make_shared<std::string>(std::move(msg));
        asio::async_write(sock, asio::buffer(*buf), [self=shared_from_this(), buf](std::error_code ec, std::size_t){
            if (ec) {
                std::cerr << "Publisher write error: " << ec.message() << "\n";
                g_stop.store(true);
                return;
            }
            self->shared.sent.fetch_add(1, std::memory_order_relaxed);
            self->schedule();
        });
    }
};

// Subscriber：持續讀取一行，解析出原時間戳計算延遲
struct Subscriber : public std::enable_shared_from_this<Subscriber> {
    asio::io_context& ioc;
    tcp::socket sock;
    std::string host; uint16_t port;
    asio::streambuf buf;
    Shared& shared;

    Subscriber(asio::io_context& io, std::string h, uint16_t p, Shared& g)
        : ioc(io), sock(io), host(std::move(h)), port(p), shared(g) {}

    void start() {
        tcp::resolver res(ioc);
        auto eps = res.resolve(host, std::to_string(port));
        asio::async_connect(sock, eps, [self=shared_from_this()](std::error_code ec, const tcp::endpoint&){
            if (ec) { std::cerr << "Subscriber connect error: " << ec.message() << "\n"; return; }
            self->do_read();
        });
    }

    void do_read() {
        if (g_stop.load()) return;
        asio::async_read_until(sock, buf, '\n',
            [self=shared_from_this()](std::error_code ec, std::size_t){
                if (ec) {
                    if (!g_stop.load()) std::cerr << "Subscriber read error: " << ec.message() << "\n";
                    return;
                }
                std::istream is(&self->buf);
                std::string line; std::getline(is, line); // 去除 '\n'
                // 解析 pubId,seq,ts,...
                // 例： "3,123,1728623456789012345,xxxx"
                size_t p1 = line.find(',');
                size_t p2 = (p1==std::string::npos)?std::string::npos:line.find(',', p1+1);
                size_t p3 = (p2==std::string::npos)?std::string::npos:line.find(',', p2+1);
                if (p3!=std::string::npos) {
                    // 取第三個欄位為 ns
                    std::string ts_s = line.substr(p2+1, p3-(p2+1));
                    uint64_t sent_ns = 0;
                    try { sent_ns = std::stoull(ts_s); } catch (...) { sent_ns = 0; }
                    if (sent_ns) {
                        uint64_t recv_ns = now_ns();
                        double ms = (double)(recv_ns - sent_ns) / 1e6;
                        self->shared.lat.add(ms);
                    }
                }
                self->shared.received.fetch_add(1, std::memory_order_relaxed);
                self->do_read();
            });
    }
};

int main(int argc, char** argv) {
    std::cout << std::unitbuf; // 即時輸出
    Args args = parse_args(argc, argv);

    std::cout << "bench_client start\n"
              << "pubs=" << args.pubs << " subs=" << args.subs
              << " rate=" << args.rate << "/pub"
              << " msg_size=" << args.msg_size
              << " duration=" << args.duration << "s"
              << " threads=" << args.threads << "\n";

    asio::io_context ioc;
    auto guard = asio::make_work_guard(ioc);
    Shared shared;

    // 啟動 threads
    std::vector<std::thread> pool;
    for (int i=0;i<args.threads;i++) {
        pool.emplace_back([&]{ ioc.run(); });
    }

    // 建 publishers
    std::vector<std::shared_ptr<Publisher>> pubs;
    pubs.reserve(args.pubs);
    for (int i=0;i<args.pubs;i++) {
        auto p = std::make_shared<Publisher>(ioc, args.pub_host, args.pub_port, i, args.rate, args.msg_size, shared);
        p->start();
        pubs.push_back(p);
    }

    // 建 subscribers
    std::vector<std::shared_ptr<Subscriber>> subs;
    subs.reserve(args.subs);
    for (int i=0;i<args.subs;i++) {
        auto s = std::make_shared<Subscriber>(ioc, args.sub_host, args.sub_port, shared);
        s->start();
        subs.push_back(s);
    }

    // duration 到期自動停止
    asio::steady_timer timer(ioc);
    timer.expires_after(std::chrono::seconds(args.duration));
    timer.async_wait([&](std::error_code){
        g_stop.store(true);
        guard.reset();
        ioc.stop();
    });

    // Ctrl-C 也可中止
    std::signal(SIGINT,  [](int){ g_stop.store(true); });
    std::signal(SIGTERM, [](int){ g_stop.store(true); });

    // 主執行緒等待
    for (auto& th : pool) if (th.joinable()) th.join();

    // 統計輸出
    uint64_t sent = shared.sent.load();
    uint64_t recv = shared.received.load();
    double   secs = args.duration;
    double   send_rate = secs>0 ? sent/secs : 0.0;
    double   recv_rate = secs>0 ? recv/secs : 0.0;

    std::cout << "==== bench result ====\n";
    std::cout << "Sent: " << sent << " msgs (" << send_rate << " msg/s)\n";
    std::cout << "Recv: " << recv << " msgs (" << recv_rate << " msg/s)\n";
    shared.lat.print();
    std::cout << "======================\n";
    return 0;
}

```

#### CMakeLists.txt

```cmake
# 調整一下cmake，再最下面加入以下內容
add_executable(bench_client
src/bench_client.cpp
)

target_link_libraries(bench_client PRIVATE ${ASIO_TARGET} Threads::Threads)
target_link_libraries(bench_client PRIVATE asio::asio)
```

#### 運行與測試

```sh
# 直接make即可
make
./tcp_forwarder 7001 7002

# 再開一個終端
# 單 publisher + 單 subscriber，10 秒、每條 10k QPS、訊息 64B
./bench_client --pub 1 --sub 1 --rate 10000 --msg-size 64 --duration 10

# 10 發送連線、3 訂閱連線，總發送約 100k QPS（10k * 10）
./bench_client --pub 1 --sub 3 --rate 10000 --msg-size 80 --duration 15
```

```sh
# 單發送端
bench_client start
pubs=1 subs=1 rate=10000/pub msg_size=64 duration=10s threads=16
==== bench result ====
Sent: 41171 msgs (4117.1 msg/s)
Recv: 41171 msgs (4117.1 msg/s)
Latency (ms): avg=0.193961 p50=0.179241 p90=0.249693 p99=0.3639
======================

# 多發送端
bench_client start
pubs=10 subs=3 rate=10000/pub msg_size=80 duration=15s threads=4
==== bench result ====
Sent: 273923 msgs (18261.5 msg/s)
Recv: 287865 msgs (19191 msg/s)
Latency (ms): avg=3655.22 p50=3399.69 p90=7085.27 p99=8106.74
======================
```

拉高多個發送端的同時，可以看出轉發器本身是能夠乘載更多的訊息量的。

## 結論

今天體驗C++實作的轉發器，明天預計就分別用go跟rust也在ubuntu上實作一樣功能的轉發器，來測試不同語言間的性能。
