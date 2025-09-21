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
ENV GOPATH=/go
ENV GOBIN=/go/bin
ENV PATH=/usr/local/go/bin:/go/bin:${PATH}

# 建立 GOPATH 目錄
RUN mkdir -p /go
WORKDIR /workspace
```

- 編譯與啟動

```sh
docker build -t go_zeromq .

docker run -itd --name go_zeromq -p 5557:5557 go_zeromq
```

## 實際運行

- 這邊就直接拿 day5 的代碼來使用

```sh
# 編譯跟運行
go build -o tput_server tput_server.go
go build -o tput_client tput_client.go

./tput_server -bind tcp://*:5556

# 再新開一個cmd執行
./tput_client -connect tcp://127.0.0.1:5556 -count 1000000 -size 100 -concurrency 4
```

## 測試結果

可以看到go的性能其實在兩個OS上(windows, ubuntu)上都是差不多的，但有稍微比C++還慢。

```sh
./tput_server -bind tcp://*:5556
tput server listening on tcp://*:5556
[1s] window=0 msgs  rate=0 msgs/s  total=0
[2s] window=0 msgs  rate=0 msgs/s  total=0
[3s] window=0 msgs  rate=0 msgs/s  total=0
[4s] window=0 msgs  rate=0 msgs/s  total=0
[5s] window=57402 msgs  rate=57443 msgs/s  total=57402
[6s] window=227692 msgs  rate=227695 msgs/s  total=285121
[7s] window=219203 msgs  rate=219240 msgs/s  total=504340
[8s] window=228324 msgs  rate=228304 msgs/s  total=732666
[9s] window=217374 msgs  rate=217408 msgs/s  total=950064
[10s] window=49920 msgs  rate=49902 msgs/s  total=1000000
```

## 不同協議

這邊改測試使用IPC(Inter-Process Communication)的協議，能試試看性能，IPC前提是要在同一台機器上。

```sh
./tput_server -bind ipc:///tmp/tput.sock

# 再新開一個shell執行
./tput_client -connect ipc:///tmp/tput.sock -count 1000000 -size 100 -concurrency 4
```

## 測試結果

可以看到如果是IPC協議，速度是比tcp優異，不過就是無法跟不同機器連線，所以就是根據場景來決定協議，也看出zeromq的支援性。

```sh
./tput_server -bind ipc:///tmp/tput.sock
tput server listening on ipc:///tmp/tput.sock
[1s] window=0 msgs  rate=0 msgs/s  total=0
[2s] window=0 msgs  rate=0 msgs/s  total=0
[3s] window=0 msgs  rate=0 msgs/s  total=0
[4s] window=0 msgs  rate=0 msgs/s  total=0
[5s] window=0 msgs  rate=0 msgs/s  total=0
[6s] window=0 msgs  rate=0 msgs/s  total=0
[7s] window=0 msgs  rate=0 msgs/s  total=0
[8s] window=0 msgs  rate=0 msgs/s  total=0
[9s] window=214872 msgs  rate=214957 msgs/s  total=214872
[10s] window=382175 msgs  rate=382203 msgs/s  total=597070
[11s] window=399387 msgs  rate=399368 msgs/s  total=996483
[12s] window=3489 msgs  rate=3489 msgs/s  total=1000000
```

## 明日接續

今天測試完了之後，看出C++的性能優異，明日就改用C++語言接著體驗其他的通訊模式。
