# gRpc 體驗

今天來體驗gRpc，不過是用C++來體驗，因為本人比較擅長這個語言。

## 套件安裝

- 這邊使用 `vcpkg` 來安裝

```sh
vcpkg install grpc protobuf
```

## 接口文檔

- gRpc的RPC接口都是定義在`.proto`的檔案裏面

```proto
// helloworld.proto
syntax = "proto3";

package demo;

// 請求與回應
message HelloRequest { string name = 1; }
message HelloReply   { string message = 1; }

// 服務定義
service Greeter {
  rpc SayHello(HelloRequest) returns (HelloReply);
}
```

- 將接口文檔生產出實際語言的Server與Client的介面代碼，實際指令透過 `cmake`裡面配合完成

```cmake
cmake_minimum_required(VERSION 3.20)
project(grpc_cpp_demo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ====== 1) 直接指定 protoc 與 grpc_cpp_plugin 的絕對路徑 ======
set(PROTOC_ABS "/opt/vcpkg/installed/x64-linux/tools/protobuf/protoc")
set(GRPC_CPP_PLUGIN_ABS "/opt/vcpkg/installed/x64-linux/tools/grpc/grpc_cpp_plugin")

#（可選）簡單防呆
if (NOT EXISTS "${PROTOC_ABS}")
  message(FATAL_ERROR "protoc not found at: ${PROTOC_ABS}")
endif()
if (NOT EXISTS "${GRPC_CPP_PLUGIN_ABS}")
  message(FATAL_ERROR "grpc_cpp_plugin not found at: ${GRPC_CPP_PLUGIN_ABS}")
endif()

# ====== 2) 指定 .proto 與輸出位置 ======
set(PROTO_DIR ${CMAKE_SOURCE_DIR}/proto)
set(GEN_DIR   ${CMAKE_BINARY_DIR}/generated)
file(MAKE_DIRECTORY ${GEN_DIR})

# 明確列出要產生的 .proto（你也可加更多檔）
set(PROTO_FILES
  ${PROTO_DIR}/helloworld.proto
)

# 對每個 .proto 生成對應的輸出清單
set(GENERATED_SRCS "")
set(GENERATED_HDRS "")
foreach(PROTO ${PROTO_FILES})
  get_filename_component(BASENAME "${PROTO}" NAME_WE)
  list(APPEND GENERATED_SRCS
    ${GEN_DIR}/${BASENAME}.pb.cc
    ${GEN_DIR}/${BASENAME}.grpc.pb.cc
  )
  list(APPEND GENERATED_HDRS
    ${GEN_DIR}/${BASENAME}.pb.h
    ${GEN_DIR}/${BASENAME}.grpc.pb.h
  )
endforeach()

# ====== 3) 在 CMake 裡直接呼叫 protoc 產生 pb 檔 ======
add_custom_command(
  OUTPUT ${GENERATED_SRCS} ${GENERATED_HDRS}
  COMMAND ${PROTOC_ABS}
          --grpc_out=${GEN_DIR}
          --cpp_out=${GEN_DIR}
          --plugin=protoc-gen-grpc=${GRPC_CPP_PLUGIN_ABS}
          -I ${PROTO_DIR}
          ${PROTO_FILES}
  DEPENDS ${PROTO_FILES}
  COMMENT "Generating *.pb.cc/*.grpc.pb.cc via protoc"
  VERBATIM
)

add_custom_target(gen_protos DEPENDS ${GENERATED_SRCS} ${GENERATED_HDRS})

# ====== 4) 建 lib 讓 server/client 連結（仍需連到 protobuf/grpc）======
# 這裡示範最簡做法：仍透過 vcpkg 的 find_package 取得 link targets
find_package(protobuf CONFIG REQUIRED)
find_package(gRPC CONFIG REQUIRED)

add_library(demo_protos STATIC ${GENERATED_SRCS})
add_dependencies(demo_protos gen_protos)
target_include_directories(demo_protos PUBLIC ${GEN_DIR})
target_link_libraries(demo_protos PUBLIC protobuf::libprotobuf gRPC::grpc++)

# ====== 5) 編 server / client ======
add_executable(server src/server.cpp)
target_link_libraries(server PRIVATE demo_protos gRPC::grpc++)

add_executable(client src/client.cpp)
target_link_libraries(client PRIVATE demo_protos gRPC::grpc++)

```

## 代碼實作

- Server端

```cpp
// server.cpp
#include <grpcpp/grpcpp.h>
#include <iostream>
#include <memory>
#include <string>

// 產生的標頭（由 CMake 產生在 build/generated/）
#include "helloworld.grpc.pb.h"
#include "helloworld.pb.h"

using grpc::Server;
using grpc::ServerBuilder;
using grpc::ServerContext;
using grpc::Status;

using demo::Greeter;
using demo::HelloReply;
using demo::HelloRequest;

class GreeterServiceImpl final : public Greeter::Service {
public:
  Status SayHello(ServerContext* context,
                  const HelloRequest* request,
                  HelloReply* reply) override {
    (void)context;
    std::string msg = "Hello, " + request->name();
    reply->set_message(msg);
    std::cout << "[server] SayHello -> " << msg << std::endl;
    return Status::OK;
  }
};

int main(int argc, char** argv) {
  std::string addr = "0.0.0.0:50051";
  if (argc > 1) addr = argv[1];

  GreeterServiceImpl service;

  ServerBuilder builder;
  builder.AddListeningPort(addr, grpc::InsecureServerCredentials());
  builder.RegisterService(&service);
  std::unique_ptr<Server> server(builder.BuildAndStart());

  std::cout << "gRPC server listening on " << addr << std::endl;
  server->Wait();
  return 0;
}

```

- Client端

```cpp
// client.cpp
#include <grpcpp/grpcpp.h>
#include <iostream>
#include <memory>
#include <string>

#include "helloworld.grpc.pb.h"
#include "helloworld.pb.h"

using grpc::Channel;
using grpc::ClientContext;
using grpc::Status;

using demo::Greeter;
using demo::HelloReply;
using demo::HelloRequest;

class GreeterClient {
public:
  explicit GreeterClient(std::shared_ptr<Channel> channel)
    : stub_(Greeter::NewStub(channel)) {}

  std::string SayHello(const std::string& name) {
    HelloRequest req; req.set_name(name);
    HelloReply   rep;
    ClientContext ctx;

    Status status = stub_->SayHello(&ctx, req, &rep);
    if (!status.ok()) {
      return "[client] RPC failed: " + status.error_message();
    }
    return rep.message();
  }

private:
  std::unique_ptr<Greeter::Stub> stub_;
};

int main(int argc, char** argv) {
  std::string target = "127.0.0.1:50051";
  std::string name   = "world";
  if (argc > 1) target = argv[1];
  if (argc > 2) name   = argv[2];

  auto channel = grpc::CreateChannel(target, grpc::InsecureChannelCredentials());
  GreeterClient client(channel);

  std::string res = client.SayHello(name);
  std::cout << res << std::endl;
  return 0;
}

```

- 編譯與運行

```sh
./server

# 另開一個shell
./client 127.0.0.1:50051 Alice
# 會看到以下結果
Hello, Alice
```

## 明日接續

明天接著探索如何搭配Istio，做到微服務的discover(服務發現)的功能。
