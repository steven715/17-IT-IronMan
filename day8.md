# ZeroMq 挖掘 4

今天就接續著昨天的結尾，繼續看Zeromq的各種通訊模式。

## 環境準備

- 直接使用 `day6` 的環境配置

## REQ-REP

- 第一個以 Request-Reply 來測試

- Server端

```cpp
// req_rep_server.cpp
#include <zmq.hpp>
#include <iostream>
#include <string>

int main(int argc, char** argv) {
  std::string bind = "tcp://*:5557";
  if (argc > 1) bind = argv[1]; // 例如: tcp://*:5557

  zmq::context_t ctx{1};
  zmq::socket_t rep{ctx, zmq::socket_type::rep};
  rep.bind(bind);

  std::cerr << "REP listening on " << bind << "\n";
  while (true) {
    zmq::message_t req;
    if (!rep.recv(req, zmq::recv_flags::none)) continue;
    // 直接回覆同樣 payload
    rep.send(req, zmq::send_flags::none);
  }
}
```

- Client端

```cpp
// req_rep_client.cpp
#include <zmq.hpp>
#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>
#include <cstring>

using clk = std::chrono::high_resolution_clock;

static long percentile(std::vector<long>& ns, double p) {
  if (ns.empty()) return 0;
  size_t k = static_cast<size_t>((ns.size()-1)*p);
  return ns[k];
}

int main(int argc, char** argv){
  std::string connect = "tcp://127.0.0.1:5557";
  int count = 10000, size = 32, warmup = 200;
  if (argc > 1) connect = argv[1];             // tcp://127.0.0.1:5557
  if (argc > 2) count   = std::stoi(argv[2]);  // 次數
  if (argc > 3) size    = std::stoi(argv[3]);  // payload bytes
  if (argc > 4) warmup  = std::stoi(argv[4]);  // 熱身

  zmq::context_t ctx{1};
  zmq::socket_t req{ctx, zmq::socket_type::req};
  req.connect(connect);

  std::vector<uint8_t> payload(size, 0x5A);

  // warmup
  for (int i=0;i<warmup;i++){
    zmq::message_t msg(payload.data(), payload.size());
    req.send(msg, zmq::send_flags::none);
    zmq::message_t rep;
    req.recv(rep, zmq::recv_flags::none);
  }

  std::vector<long> rtts; rtts.reserve(count);
  auto t0_all = clk::now();
  for(int i=0;i<count;i++){
    auto t0 = clk::now();
    req.send(zmq::buffer(payload), zmq::send_flags::none);
    zmq::message_t rep;
    req.recv(rep, zmq::recv_flags::none);
    auto dt = std::chrono::duration_cast<std::chrono::nanoseconds>(clk::now()-t0).count();
    rtts.push_back(dt);
  }
  auto dur_all_ns = std::chrono::duration_cast<std::chrono::nanoseconds>(clk::now()-t0_all).count();

  std::sort(rtts.begin(), rtts.end());
  long sum=0; for(auto v: rtts) sum += v;
  long avg = sum / (long)rtts.size();
  std::cout << "count="<<count<<" size="<<size<<"B total="<< (dur_all_ns/1e6) <<"ms "
            << "avg="<< (avg/1e3) <<"us p50="<<(percentile(rtts,0.5)/1e3)<<"us "
            << "p95="<<(percentile(rtts,0.95)/1e3)<<"us p99="<<(percentile(rtts,0.99)/1e3)<<"us\n";
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

add_executable(req_rep_server req_rep_server.cpp)
target_link_libraries(req_rep_server PRIVATE ${ZMQ_LIB} Threads::Threads)

add_executable(req_rep_client req_rep_client.cpp)
target_link_libraries(req_rep_client PRIVATE ${ZMQ_LIB} Threads::Threads)
```

- 編譯與運行，建議直接透過vscode裡面的cmake extension打包編譯，下面就只展示運行指令

```sh
# 先到build
cd build
# 第一個shell
./req_rep_server tcp://*:5557

# 第二個shell
./req_rep_client tcp://127.0.0.1:5557 50000 32 500
```

## 測試結果

REQ-REP這邊送了50000筆請求，每筆請求的payload是32bytes，由於REQ-REP是單筆處理，所以總耗時是12.4秒左右，平均是250微秒。

```sh
./req_rep_client tcp://127.0.0.1:5557 50000 32 500
count=50000 size=32B total=12389.5ms avg=247.393us p50=230.098us p95=336.878us p99=496.848us
```

## PUB-SUB

- Server端

```cpp
// sub.cpp
#include <zmq.hpp>
#include <iostream>
#include <chrono>

int main(int argc, char** argv){
  std::string connect = "tcp://127.0.0.1:5556";
  if (argc>1) connect = argv[1];

  zmq::context_t ctx{1};
  zmq::socket_t sub{ctx, zmq::socket_type::sub};
  sub.set(zmq::sockopt::subscribe, ""); // 訂閱全部
  sub.connect(connect);

  std::cerr << "SUB connected to " << connect << "\n";
  long long total=0, window=0;
  auto last = std::chrono::steady_clock::now();

  while(true){
    zmq::message_t msg;
    if (!sub.recv(msg, zmq::recv_flags::none)) continue;
    total++; window++;

    auto now = std::chrono::steady_clock::now();
    if (std::chrono::duration_cast<std::chrono::seconds>(now-last).count()>=1){
      auto sec = std::chrono::duration_cast<std::chrono::seconds>(now-last).count();
      double rate = (double)window / (double)sec;
      std::cout << "[+"<<sec<<"s] window="<<window<<" msgs  rate="<< (long long)rate <<" msgs/s  total="<<total<<"\n";
      window=0; last=now;
    }
  }
}
```

- Client端

```cpp
// pub.cpp
#include <zmq.hpp>
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>

int main(int argc, char** argv){
  std::string bind = "tcp://*:5556";
  int count = 1'000'000, size = 100, linger_ms = 0, sleep_ms = 0;
  if (argc>1) bind = argv[1];
  if (argc>2) count = std::stoi(argv[2]);
  if (argc>3) size  = std::stoi(argv[3]);
  if (argc>4) linger_ms = std::stoi(argv[4]);  // 關閉等待
  if (argc>5) sleep_ms  = std::stoi(argv[5]);  // 每筆間隔（控制速率）

  zmq::context_t ctx{1};
  zmq::socket_t pub{ctx, zmq::socket_type::pub};
  pub.set(zmq::sockopt::sndhwm, 100000);
  pub.set(zmq::sockopt::linger, linger_ms);
  pub.bind(bind);

  std::cerr << "PUB bound at " << bind << "  waiting 500ms for subs...\n";
  std::this_thread::sleep_for(std::chrono::milliseconds(500));

  std::vector<uint8_t> payload(size, 0x42);

  auto t0 = std::chrono::steady_clock::now();
  for (int i=0;i<count;i++){
    pub.send(zmq::buffer(payload), zmq::send_flags::none);
    if (sleep_ms>0) std::this_thread::sleep_for(std::chrono::milliseconds(sleep_ms));
  }
  auto dt = std::chrono::duration_cast<std::chrono::milliseconds>(std::chrono::steady_clock::now()-t0).count();
  double mps = (double)count / ((double)dt/1000.0);
  double mib = ((double)count*size)/(1024.0*1024.0)/((double)dt/1000.0);
  std::cout << "Sent " << count << " msgs(size="<<size<<") in "<<dt<<" ms => "<<(long long)mps<<" msgs/s, "<<mib<<" MiB/s\n";
}
```

- 編譯與運行

```sh
# 先到build
cd build
# 第一個shell
./sub tcp://127.0.0.1:5556

# 第二個shell
./pub tcp://*:5556 1000000 100 0
```

## 測試結果

- 這邊看到的結果是針對pub送完1000000筆每筆100bytes大小的消息，總共花費了654毫秒。

```sh
./pub tcp://*:5556 1000000 100 0
PUB bound at tcp://*:5556  waiting 500ms for subs...
Sent 1000000 msgs(size=100) in 654 ms => 1529051 msgs/s, 145.822 MiB/s
```

## 明日接續

今天先介紹這兩個通訊模式，明天再接著介紹DEALER-ROUTER的模式。
