# gRpc 體驗 2

今天來接著體驗gRpc的衍生議題，gRpc作為一個微服務的套件或第三方庫，他提供了一個兩邊要遵守的接口(`.proto`)以及溝通的資料格式(`protobuf`)，透過`protoc`會生成出我們要使用語言的接口介面，好讓我們後續能利用該介面去實作我們的Server跟Client。

以昨天的用例來看Client要通過具體的ip/port來呼叫Server，而像我之前用過的一個微服務框架(Zeroc ICE)，對於跟服務的溝通，其實是可以透過服務註冊跟服務發現的方式，來抽象具體的連線位置的，所以今天會主要來探討這個部分。

## 微服務要素

這邊就先來看看微服務的要素會有哪些，主要是以我自己對微服務的認識去看

- 服務提供: 業務功能的服務提供，能讓其客戶端享用服務端提供的服務
- 服務註冊/服務發現: 能透過服務註冊，來抽象具體連線位置的能力，服務發現就是讓客戶端透過註冊的位置來獲取服務
- 流量治理: 接著服務發現來衍生，提供一個服務註冊，背後的服務節點其實就能根據需求做動態的增減，當服務節點一多，就能透過負載平衡的分散單節點的壓力
- 高可用性(High Availible): 服務節點如果掛掉了，會有健康檢測來判斷，掛了之後會自動將服務啟回來，保持服務的可用性
- 協議一致: 服務器與客戶端要有一致的溝通協議，資料格式等
- 安全: 支援TLS、SSL等安全協議

## 環境準備

- 研究的過程中，發現因為之前的ubuntu 是用docker啟的，走的是預設的網路模式(bridge)，會導致DNS解析上有問題，所以這邊會重啟一個新的ubuntu 作為grpc的server及client，以及除了使用`consul`外，還會使用`CoreDns`來幫忙做DNS的解析，才能再docker的環境下，做到預期的服務發現

- 所以用使用`docker-compose`的方式來讓整個環境部屬起來

```sh
# 預期的檔案結構
C:\USERS\ASUS\STEVEN\GRPC_COREDNS_CONSUL
│  docker-compose.yml
│  Dockerfile
│
├─consul
│      Consul.Dockerfile
│      greeter.json
│
└─coredns
        Corefile
```

- 最外層的`Dockerfile`，是用`day6`加上昨天使用vcpkg 安裝 gRpc的映象檔，用vcpkg安裝grpc，實測下來都需要20分鐘以上，所以索性就當環境初始的一項

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

RUN /opt/vcpkg/vcpkg install grpc protobuf
```

- 再來準備CoreDns的配置檔 Corefile

```corefile
consul:53 {
    forward . 172.30.0.11:8600
    log
    errors
}

.:53 {
    forward . 8.8.8.8
    log
    errors
}
```

- 準備consul的配置檔，也是註冊服務的配置

```json
{
  "service": {
    "name": "greeter",
    "id": "greeter-1",
    "address": "172.30.0.20",
    "port": 50051,
    "checks": [
      {
        "name": "tcp on 50051",
        "tcp": "172.30.0.20:50051",
        "interval": "5s",
        "timeout": "2s"
      }
    ]
  }
}
```

- consul也另外用dockerfile的方式建置，因為windows檔案權限的緣故

```dockerfile
FROM hashicorp/consul:latest
COPY greeter.json /consul/config/greeter.json
```

- 最後就是核心的`docker-compose.yml`

```docker
version: "3.9"

networks:
  mesh:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16

services:
  consul:
    build:
      context: ./consul
      dockerfile: Consul.Dockerfile   # 自己取的檔名
    command: agent -dev -client 0.0.0.0 -ui -config-dir=/consul/config
    networks:
      mesh:
        ipv4_address: 172.30.0.11
    ports:
      - "8500:8500"
      - "8600:8600/udp"

  coredns:
    image: coredns/coredns:latest
    command: -conf /Corefile
    networks:
      mesh:
        ipv4_address: 172.30.0.10
    volumes:
      - ./coredns/Corefile:/Corefile:ro
    # 不對外開 53，僅供 mesh 內其他容器使用

  cpp_grpc:
    build:
      context: .            # 這裡放你的 Dockerfile
      dockerfile: Dockerfile
    container_name: cpp_grpc
    networks:
      mesh:
        ipv4_address: 172.30.0.20
    # 讓這個容器的 DNS 指向 CoreDNS（它會把 .consul 轉發到 consul:8600）
    dns:
      - 172.30.0.10
    working_dir: /root/grpc-cpp
    volumes:
      # 把 Windows 專案資料夾掛進來（注意 /c/... 的寫法）
      - /c/Users/Asus/steven/grpc-cpp:/root/grpc-cpp
    tty: true
    depends_on:
      - consul
      - coredns
    # 不編譯，不自動跑；你會自己進容器操作
```

- 上述準備好後，接著就是啟動整個環境

```sh
docker-compose up -d --build
```

- 部屬完後，記得再進去`cpp_grpc`的容器裡面，編譯昨天的`server`跟`client`，另外我有把昨天的專案，先從docker裡面下載回本機，這樣就不用再重新弄一次昨天的代碼
- 啟動昨天的`server`

```sh
./server
```

- 客戶端改用註冊的服務名來啟動

```sh
./client dns:///greeter.service.consul:50051 Alice
# 預期輸出：Hello, Alice
```

## 體驗結論

雖然現在的微服務框架看起來好像很複雜，但只要搞清楚到底有哪些功能，由誰來實現哪些功能，這些功能是怎麼運作的，然後實際演練過一次，感覺上就不會有那麼複雜了，俗話說`大道至簡`，最根本的道理往往是最簡單的，理解這些微服務溝通的過程，我覺得還滿像這句話的。

最後就是本來預期要體驗的`Istio`，就放到之後體驗`k8s`的時候再來試試囉。
