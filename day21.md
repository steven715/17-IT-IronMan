# ELK 體驗

今天來體驗 ELK ，實際上ELK是由 ElasticSearch(搜尋引擎)、Logstash(資料處理管道)、Kinbana(UI介紹提供搜尋結果)三種組成而成的。

## 今日目標

透過docker方式部署ELK，並且寫一個使用spdlog 的c++ 容器，讓log能被ELK接收，並讓再kibana 上面查詢

## 環境準備

1. 目錄結構

```sh
C:\USERS\ASUS\STEVEN\ELK-LAB
│  docker-compose.yml
│  filebeat.yml
│
├─app-cpp
│  │  Dockerfile
│  │
│  └─cpp-spdlog
│          CMakeLists.txt
│          main.cpp
│
└─logs
        
```

2. docker-compose.yml

```yaml
version: "3.9"
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.14.3
    container_name: es
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false        # demo 簡化：先關掉 auth
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock: { soft: -1, hard: -1 }
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200/_cluster/health"]
      interval: 10s
      timeout: 5s
      retries: 20

  kibana:
    image: docker.elastic.co/kibana/kibana:8.14.3
    container_name: kib
    environment:
      - ELASTICSEARCH_HOSTS=http://es:9200
      - XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.14.3
    container_name: fb
    user: root
    depends_on:
      - elasticsearch
      - kibana
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /c/Users/Asus/steven/elk-lab/logs:/logs:ro
    environment:
      - ELASTICSEARCH_HOSTS=http://es:9200
    command: ["--strict.perms=false"]   # 讓掛載的檔案權限不至於擋住啟動

  cpp-app:
    build:
      context: ./app-cpp
    image: cpp-spdlog-demo:latest
    container_name: cpp-app
    depends_on:
      - filebeat
    volumes:
      - /c/Users/Asus/steven/elk-lab/logs:/logs
    environment: []
    restart: unless-stopped

volumes:
  esdata:

```

3. filebeat.yml 這邊會透過 filebeat 這個套件將日誌拉給 logstash 使用

```yaml
filebeat.inputs:
  - type: filestream
    id: cpp_app_logs
    enabled: true
    paths:
      - /logs/*.log
    # 每行都是 JSON，就用 ndjson 解析；target 設空字串代表展平成根層欄位
    parsers:
      - ndjson:
          target: ""
          overwrite_keys: true
    processors:
      # 你的 JSON 若有毫秒時間戳欄位（例如 ts），可自動轉成 @timestamp
      - timestamp:
          field: ts
          layouts:
            - UNIX_MS
          ignore_missing: true
      # 打一些固定標籤方便在 Kibana 篩選
      - add_fields:
          target: labels
          fields:
            app: cpp-spdlog
            env: dev

# 出口（用 compose 的環境變數；沒設就走 elasticsearch:9200）
output.elasticsearch:
  hosts: ["${ELASTICSEARCH_HOSTS:elasticsearch:9200}"]

#（可選）讓 setup 流程能連到 Kibana
setup.kibana:
  host: "kibana:5601"

# 記錄器
logging.to_files: false
logging.level: info

```

4. app-cpp/Dockerfile

```dockerfile
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
    apt-get install -y ca-certificates && \
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

RUN /opt/vcpkg/vcpkg install spdlog

WORKDIR /root/cpp-spdlog
COPY ./cpp-spdlog /root/cpp-spdlog
RUN mkdir build && cd build && \ 
    cmake .. -DCMAKE_TOOLCHAIN_FILE=/opt/vcpkg/scripts/buildsystems/vcpkg.cmake && \
    make

CMD ["/root/cpp-spdlog/build/app"]
```

5. app-cpp/cpp-spdlog/main.cpp


```cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/stdout_color_sinks.h>
#include <chrono>
#include <thread>
#include <string>

int main() {
    auto logger = spdlog::stdout_color_mt("console");
    // 時間用 epoch 毫秒，字段有 level/name/msg
    spdlog::set_pattern(R"({"ts":%Ems,"level":"%l","logger":"%n","msg":"%v"})");

    int i = 0;
    while (true) {
        logger->info("hello from spdlog #{}, user=alice, action=login", i++);
        logger->warn("warn event #{}, module=auth, code=1001", i);
        logger->error("error event #{}, module=payment, code=42", i);
        std::this_thread::sleep_for(std::chrono::milliseconds(250));
    }
    return 0;
}

```

6. app-cpp/cpp-spdlog/CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(grpc_cpp_demo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(spdlog CONFIG REQUIRED)
add_executable(app main.cpp)
target_link_libraries(app PRIVATE spdlog::spdlog)
```

7. 啟動

```sh
docker compose up -d --build
```

8. 確認資料

- 打開 [Kibana](http://localhost:5601)

![step1](/elastic-demo/kibana.png)

- 按左上角的選項，再按 Discover，就能看到日誌被elk輸出到畫面上了

![step2](/elastic-demo/discover.png)
![step3](/elastic-demo/log.png)

- 另外透過建立一個data view，來將應用程式的資料獨立出來

![step4](/elastic-demo/create_data_view.png)
![step5](/elastic-demo/save_data_view.png)
![step6](/elastic-demo/data_view.png)

9. 清除環境

```sh
docker compose down
```

## 結論

本次 ELK 體驗就先簡單用到這邊，這次的內容其實也是目前工作上有使用到的部分，衍伸可以探討 opentelemetry 相關的內容，但目前還尚未有動力進行，就看未來是否有機會接觸更多囉。
