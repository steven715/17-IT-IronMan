# ZeroMq 挖掘 5

今天來補齊Zeromq 最後一個 DEALER-ROUTER的通訊模式，一樣使用C++來探討。

## 環境準備

- 直接使用 `day6` 的環境配置

## DEALER-ROUTER

整個拓樸: REQ clients => (ROUTER) BROKER (DEALER) => REP workers
- broker: 前端 ROUTER 綁一個*:7000，後端 DEALER 綁*:7001，用 zmq_proxy 轉發

```cpp
// broker.cpp
#include <zmq.hpp>
#include <iostream>

int main(int, char**){
  zmq::context_t ctx{1};
  zmq::socket_t frontend(ctx, zmq::socket_type::router);
  zmq::socket_t backend (ctx, zmq::socket_type::dealer);

  frontend.bind("tcp://*:7000"); // clients (REQ) 連到這
  backend.bind ("tcp://*:7001"); // workers (REP) 連到這
  std::cerr << "Broker: ROUTER tcp://*:7000  <->  DEALER tcp://*:7001\n";

  // 直接用內建 proxy
  zmq::proxy(static_cast<void*>(frontend), static_cast<void*>(backend), nullptr);
  return 0;
}
```

- worker: 同前面REQ-REP的Server端

```cpp
// worker.cpp
#include <zmq.hpp>
#include <iostream>
#include <thread>

int main(int argc, char** argv){
  std::string connect = "tcp://127.0.0.1:7001";
  if (argc>1) connect = argv[1];

  zmq::context_t ctx{1};
  zmq::socket_t rep(ctx, zmq::socket_type::rep);
  rep.connect(connect);
  std::cerr << "Worker REP connected " << connect << "\n";

  while(true){
    zmq::message_t msg;
    if (!rep.recv(msg, zmq::recv_flags::none)) continue;
    // 直接回覆同樣 payload（模擬處理可 sleep 幾毫秒）
    rep.send(msg, zmq::send_flags::none);
  }
}
```

- client: 同前面REQ-REP的Client端

```cpp
// dr_client.cpp
#include <zmq.hpp>
#include <iostream>
#include <thread>
#include <vector>
#include <atomic>
#include <chrono>
#include <cmath>

int main(int argc, char** argv){
  std::string connect = "tcp://127.0.0.1:7000";
  int total=1'000'000, size=100, concurrency=4;
  if (argc>1) connect = argv[1];
  if (argc>2) total   = std::stoi(argv[2]);
  if (argc>3) size    = std::stoi(argv[3]);
  if (argc>4) concurrency = std::stoi(argv[4]);

  zmq::context_t ctx{1};
  std::vector<uint8_t> payload(size, 0x33);

  // 均分任務（前 rem 個多 1）
  int base = total / concurrency, rem = total % concurrency;

  std::atomic<long long> done{0};
  auto t0 = std::chrono::steady_clock::now();

  std::vector<std::thread> ths;
  ths.reserve(concurrency);
  for (int w=0; w<concurrency; ++w){
    int n = base + (w<rem ? 1:0);
    ths.emplace_back([&, n](){
      zmq::socket_t req(ctx, zmq::socket_type::req);
      req.connect(connect);
      for (int i=0;i<n;i++){
        req.send(zmq::buffer(payload), zmq::send_flags::none);
        zmq::message_t rep;
        req.recv(rep, zmq::recv_flags::none);
        ++done;
      }
    });
  }

  // 每秒輸出一次總吞吐
  std::thread meter([&](){
    long long last = 0;
    auto last_t = std::chrono::steady_clock::now();
    while (done < total){
      std::this_thread::sleep_for(std::chrono::seconds(1));
      auto now = std::chrono::steady_clock::now();
      auto diff = done.load() - last;
      auto sec = std::chrono::duration_cast<std::chrono::seconds>(now-last_t).count();
      if (sec==0) continue;
      std::cout << "[+"<<sec<<"s] window="<<diff<<" rps  total="<<done<<"\n";
      last = done.load(); last_t = now;
    }
  });

  for (auto& t: ths) t.join();
  meter.join();

  auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::steady_clock::now()-t0).count();
  double rps = (double)total / ((double)ms/1000.0);
  double mib = ((double)total*size)/(1024.0*1024.0)/((double)ms/1000.0);
  std::cout << "Completed " << total << " req-rep via broker in "<<ms<<" ms => "
            << (long long)rps << " req/s, " << mib << " MiB/s\n";
}
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

add_executable(broker broker.cpp)
target_link_libraries(broker PRIVATE ${ZMQ_LIB} Threads::Threads)

add_executable(worker worker.cpp)
target_link_libraries(worker PRIVATE ${ZMQ_LIB} Threads::Threads)

add_executable(dr_client dr_client.cpp)
target_link_libraries(dr_client PRIVATE ${ZMQ_LIB} Threads::Threads)

```

- 編譯與運行

```sh
# 視窗A：啟動 broker
./broker

# 視窗B：啟動多個 worker（可多開終端）
./worker tcp://127.0.0.1:7001 &
./worker tcp://127.0.0.1:7001 &

# 視窗C：跑 client（可調整筆數/大小/併發）
./dr_client tcp://127.0.0.1:7000 1000000 100 8
```

## 測試結果

下面數據對照昨天用REQ-REP跑的，同樣是50000筆，每筆32 bytes，昨天是耗時12.4秒，換成DEALER-ROUTER的就只需要5秒。

```sh
./dr_client tcp://127.0.0.1:7000 50000 32 8
[+1s] window=11583 rps  total=11583
[+1s] window=11931 rps  total=23514
[+1s] window=11719 rps  total=35233
[+1s] window=11906 rps  total=47139
[+1s] window=2861 rps  total=50000
Completed 50000 req-rep via broker in 5002 ms => 9996 req/s, 0.305054 MiB/s
```

## ZeroMq 體驗後感言

這工具確實提供socket一種更簡便、高效的使用，這次只針對通訊模式做一些基本的使用，未來有機會再探討這些通訊模式實際能解決的問題，不過體驗就先在這告一段落，明天繼續體驗其他項目。
