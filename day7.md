# ZeroMq 挖掘 3

昨天看到使用C++語言的zeromq跑再ubuntu系統上的性能明顯優異於go語言跑再windows上，今天接續著讓go語言跑再windows上面進行測試。

## 環境準備

- 使用下面這個dockerfile來生成環境

```dockerfile
FROM ubuntu:22.04

# 確保 non-interactive 模式，避免安裝過程卡住
ENV DEBIAN_FRONTEND=noninteractive

# 更新並安裝必要工具
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        git \
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
        valgrind \
        ca-certificates && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# --------------------------
# 安裝 Go (官方 tarball)
# --------------------------
ENV GOLANG_VERSION=1.22.6
RUN curl -OL https://go.dev/dl/go${GOLANG_VERSION}.linux-amd64.tar.gz && \
    tar -C /usr/local -xzf go${GOLANG_VERSION}.linux-amd64.tar.gz && \
    rm go${GOLANG_VERSION}.linux-amd64.tar.gz

# 設定 Go 環境變數
ENV PATH=$PATH:/usr/local/go/bin \
    GOPATH=/go \
    PATH=$PATH:/go/bin

# 建立 GOPATH 目錄
RUN mkdir -p /go
WORKDIR /workspace
```

- 編譯與啟動

```sh
docker build -t go_zeromq .

docker run -itd --name go_zeromq -p 5556:5556 go_zeromq
```

## 實際運行

- 這邊就直接拿 day5 的代碼來使用

```sh

```


