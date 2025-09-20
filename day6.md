# ZeroMq 挖掘 2

今天就來試試改用其他語言跑看看ZeroMq的性能。

## 環境準備

這邊用以下的dockerfile啟一個C++的環境來執行zeromq。

```dockerfile
# dockerfile 
FROM ubuntu:22.04

# 確保 non-interactive 模式，避免安裝過程卡住
ENV DEBIAN_FRONTEND=noninteractive

# 更新並安裝必要工具
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        git \
        clang-format \
        cppcheck \
        aspell \
        colordiff \
        zip unzip \
        tar \
        pkg-config \
        vim \
        curl \
        gdb \
        htop \
        iproute2 \
        iputils-ping \
        valgrind && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN apt-get update && \
    apt-get install -y curl ca-certificates && \
    cd /usr/local/src && \
    curl -LO https://github.com/Kitware/CMake/releases/download/v3.22.2/cmake-3.22.2-linux-x86_64.tar.gz && \
    tar -xvf cmake-3.22.2-linux-x86_64.tar.gz && \
    mv cmake-3.22.2-linux-x86_64 /usr/local/cmake

# 設定 CMake 到 PATH（對所有 shell 生效）
ENV PATH="/usr/local/cmake/bin:$PATH"

# 安裝 vcpkg
WORKDIR /opt
RUN git clone https://github.com/microsoft/vcpkg.git && \
    cd vcpkg && \
    ./bootstrap-vcpkg.sh -disableMetrics

# 設定環境變數（方便後續使用）
ENV VCPKG_ROOT=/opt/vcpkg
ENV PATH="${VCPKG_ROOT}:${PATH}"
```

- 編譯與啟動

```sh
docker build -t cpp_zeromq .

docker run -itd --name cpp_zeromq -p 5556:5556 cpp_zeromq
```

## 代碼實作

- 安裝 zeromq

```sh
vcpkg install zeromq cppzmq
```

- CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.15)
project(zmq_perf_cpp CXX)
set(CMAKE_CXX_STANDARD 17)

find_package(Threads REQUIRED)

# 嘗試找 cppzmq（header-only），如果找不到也不致命
find_path(CPPZMQ_INCLUDE_DIRS NAMES zmq.hpp)
if (CPPZMQ_INCLUDE_DIRS)
  include_directories(${CPPZMQ_INCLUDE_DIRS})
endif()

# 找 libzmq
find_library(ZMQ_LIB NAMES zmq libzmq)
if (NOT ZMQ_LIB)
  message(FATAL_ERROR "libzmq not found. Install zeromq / libzmq3-dev.")
endif()

add_executable(tput_server tput_server.cpp)
target_link_libraries(tput_server PRIVATE ${ZMQ_LIB} Threads::Threads)

add_executable(tput_client tput_client.cpp)
target_link_libraries(tput_client PRIVATE ${ZMQ_LIB} Threads::Threads)
```

- Server端

```cpp
#include <zmq.hpp>
#include <iostream>
#include <chrono>
#include <thread>
#include <csignal>

using Clock = std::chrono::steady_clock;

static bool g_running = true;
void on_sigint(int){ g_running = false; }

int main(int argc, char** argv) {
    std::string bind = "tcp://*:5556";
    if (argc >= 3 && std::string(argv[1]) == "--bind") bind = argv[2];

    std::signal(SIGINT, on_sigint);
#ifdef SIGTERM
    std::signal(SIGTERM, on_sigint);
#endif

    zmq::context_t ctx(1);
    zmq::socket_t pull(ctx, zmq::socket_type::pull);
    // 高吞吐可選：加大 HWM（預設 1000）
    // int hwm = 100000;
    // pull.set(zmq::sockopt::rcvhwm, hwm);

    pull.bind(bind);
    std::cout << "tput server listening on " << bind << std::endl;

    int64_t total = 0;
    int64_t window = 0;

    auto start   = Clock::now();
    auto lastLog = start;

    // 用 poll 每 1000ms 打印統計；poll 期間讀到就累計
    while (g_running) {
        zmq::pollitem_t items[] = { { static_cast<void*>(pull), 0, ZMQ_POLLIN, 0 } };
        // 最長等 1000ms，期間如果有資料就讀
        int rc = zmq::poll(items, 1, std::chrono::milliseconds(1000));

        if (rc > 0 && (items[0].revents & ZMQ_POLLIN)) {
            // 盡量把當前可讀的都讀一讀（避免每次只收一條）
            while (true) {
                zmq::message_t msg;
                zmq::recv_result_t res = pull.recv(msg, zmq::recv_flags::dontwait);
                if (!res) break; // 沒有更多了
                ++total;
                ++window;
            }
        }

        auto now = Clock::now();
        auto elapsedLog = std::chrono::duration<double>(now - lastLog).count();
        if (elapsedLog >= 1.0) {
            double rate = window / elapsedLog;
            double sec  = std::chrono::duration<double>(now - start).count();
            std::cout << "[" << static_cast<int>(sec) << "s] "
                      << "window=" << window << " msgs  "
                      << "rate=" << static_cast<long long>(rate) << " msgs/s  "
                      << "total=" << total << std::endl;
            window = 0;
            lastLog = now;
        }
    }

    std::cout << "server exit. total=" << total << std::endl;
    return 0;
}

```

- Client端

```cpp
#include <zmq.hpp>
#include <iostream>
#include <vector>
#include <thread>
#include <atomic>
#include <chrono>
#include <cmath>

using Clock = std::chrono::steady_clock;

static void send_worker(const std::string& addr, int n, const std::vector<uint8_t>& payload,
                        std::atomic<long long>& sent_total) {
    zmq::context_t ctx(1);
    zmq::socket_t push(ctx, zmq::socket_type::push);
    // 高吞吐可選：加大 SNDHWM
    // int hwm = 100000;
    // push.set(zmq::sockopt::sndhwm, hwm);

    push.connect(addr);

    zmq::message_t msg(payload.size());
    std::memcpy(msg.data(), payload.data(), payload.size());

    for (int i = 0; i < n; ++i) {
        // 複用 message_t，避免重複分配
        zmq::message_t m(payload.size());
        std::memcpy(m.data(), payload.data(), payload.size());
        if (!push.send(m, zmq::send_flags::none)) {
            // 送失敗（罕見，除非被打斷），可重試；這裡直接略過
        }
    }
    sent_total += n;
}

int main(int argc, char** argv) {
    std::string connect = "tcp://127.0.0.1:5556";
    int count = 1'000'000;
    int size  = 100;
    int concurrency = 4;

    for (int i = 1; i + 1 < argc; ++i) {
        std::string k = argv[i];
        if (k == "--connect") connect = argv[++i];
        else if (k == "--count") count = std::stoi(argv[++i]);
        else if (k == "--size") size = std::stoi(argv[++i]);
        else if (k == "--concurrency") concurrency = std::stoi(argv[++i]);
    }

    std::vector<uint8_t> payload(size);
    for (int i = 0; i < size; ++i) payload[i] = static_cast<uint8_t>(i);

    // 平均分配 + 餘數前面幾個多送 1
    int base = count / concurrency;
    int rem  = count % concurrency;

    std::atomic<long long> sent_total{0};
    std::vector<std::thread> ths;
    ths.reserve(concurrency);

    auto t0 = Clock::now();
    for (int w = 0; w < concurrency; ++w) {
        int n = base + (w < rem ? 1 : 0);
        ths.emplace_back(send_worker, std::cref(connect), n, std::cref(payload), std::ref(sent_total));
    }
    for (auto& th : ths) th.join();
    auto t1 = Clock::now();

    double secs = std::chrono::duration<double>(t1 - t0).count();
    long long sent = sent_total.load();
    double mps  = sent / secs;
    double mib  = (static_cast<double>(sent) * size) / (1024.0 * 1024.0);
    double mibps = mib / secs;

    std::cout << "Sent " << sent << " msgs (size=" << size << "B) in "
              << secs << "s => " << static_cast<long long>(mps)
              << " msgs/s, " << mibps << " MiB/s" << std::endl;

    return 0;
}

```

- 編譯與運行

```sh
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=/opt/vcpkg/scripts/buildsystems/vcpkg.cmake
make

./tput_server --bind tcp://*:5556
# 再開一個terminal
./tput_client --connect tcp://127.0.0.1:5556 --count 1000000 --size 100 --concurrency 4
```

## 測試結果

從結果可以發現c++性能更優秀，只花了2秒的時間，不過由於作業系統的不同，C++使用ubuntu 22.04，go使用windwos，所以實際上還是要讓go也在ubuntu也執行才合理。

```sh
# Server紀錄
./tput_server --bind tcp://*:5556
tput server listening on tcp://*:5556
[1s] window=0 msgs  rate=0 msgs/s  total=0
[2s] window=0 msgs  rate=0 msgs/s  total=0
[3s] window=0 msgs  rate=0 msgs/s  total=0
[4s] window=0 msgs  rate=0 msgs/s  total=0
[5s] window=0 msgs  rate=0 msgs/s  total=0
[6s] window=398113 msgs  rate=398107 msgs/s  total=398113
[7s] window=601887 msgs  rate=491424 msgs/s  total=1000000
[8s] window=0 msgs  rate=0 msgs/s  total=1000000
```

## 明日接續

明天再透過ubuntu上執行go的程序來看性能差距，以及開始測試其他的通訊模式，目前兩個語言版本都是用PUSH/PULL 任務分派的模式，其他還有REQ/REP 請求-回復、PUB/SUB 發布-訂閱、DEALER/ROUTER 非同步的請求-回覆等不同模式可以探討，另外zeromq也支援不同傳輸層，有tcp, ipc, inproc等，也是一個可以接續探討的項目，那就明日再見。
