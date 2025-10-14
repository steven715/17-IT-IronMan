# 網路協議 udp 體驗 

今天來體驗看看由 google 推出的基於 udp 的 [quic](https://zh.wikipedia.org/zh-tw/QUIC) 協議。 

## 環境準備

這邊一樣延續 day6 的 ubuntu 22.04 C++開發環境。

## 套件安裝

這邊依然是使用 vcpkg 進行安裝

```sh
vcpkg install msquic
```

## Server端

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
project(quic_echo_server C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)


add_executable(quic_echo_server
src/quic_echo_server.c
)

# vcpkg msquic
find_package(msquic CONFIG REQUIRED)
target_link_libraries(quic_echo_server PRIVATE msquic)

# pthread（Linux）
find_package(Threads REQUIRED)
target_link_libraries(quic_echo_server PRIVATE Threads::Threads)

# 針對 Release 最佳化
if (CMAKE_BUILD_TYPE STREQUAL "Release")
  target_compile_options(quic_echo_server PRIVATE -O3)
endif()
```

### src/quic_echo_server.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <inttypes.h>
#include <msquic.h>

// ==== config ====
// 監聽 0.0.0.0:4443 (UDP)
#define LISTEN_ADDR "0.0.0.0"
#define LISTEN_PORT 4443
// 自訂 ALPN（不是 HTTP/3）
static const char* kAlpn = "qchat";
// 憑證檔案
static const char* kCertFile = "server.crt";
static const char* kKeyFile  = "server.key";

// ==== MsQuic 全域 ====
const QUIC_API_TABLE* MsQuic = NULL;
HQUIC Registration = NULL;
HQUIC Configuration = NULL;
HQUIC Listener = NULL;

// 前置宣告
_IRQL_requires_max_(PASSIVE_LEVEL)
_Function_class_(QUIC_CONNECTION_CALLBACK)
QUIC_STATUS QUIC_API
ConnectionCallback(HQUIC Connection, void* Context, QUIC_CONNECTION_EVENT* Event);

_IRQL_requires_max_(PASSIVE_LEVEL)
_Function_class_(QUIC_STREAM_CALLBACK)
QUIC_STATUS QUIC_API
StreamCallback(HQUIC Stream, void* Context, QUIC_STREAM_EVENT* Event);

// 工具：印錯並退出
static void die(const char* msg, QUIC_STATUS st) {
    fprintf(stderr, "%s: 0x%x\n", msg, st);
    exit(1);
}

// ==== Stream 回呼：收到資料就 echo 回去 ====
QUIC_STATUS QUIC_API
StreamCallback(HQUIC Stream, void* Context, QUIC_STREAM_EVENT* Event) {
    (void)Context;

    switch (Event->Type) {
    case QUIC_STREAM_EVENT_RECEIVE: {
        // 將收到的每個片段複製到自有記憶體，送出去（echo）
        for (uint32_t i = 0; i < Event->RECEIVE.BufferCount; ++i) {
            const QUIC_BUFFER* r = &Event->RECEIVE.Buffers[i];

            // 1) 配置送出緩衝（你的擁有權）
            QUIC_BUFFER* sbuf = (QUIC_BUFFER*)malloc(sizeof(QUIC_BUFFER));
            if (!sbuf) { MsQuic->StreamShutdown(Stream, QUIC_STREAM_SHUTDOWN_FLAG_ABORT, 0); return QUIC_STATUS_OUT_OF_MEMORY; }

            sbuf->Length = r->Length;
            sbuf->Buffer = (uint8_t*)malloc(r->Length);
            if (!sbuf->Buffer) { free(sbuf); MsQuic->StreamShutdown(Stream, QUIC_STREAM_SHUTDOWN_FLAG_ABORT, 0); return QUIC_STATUS_OUT_OF_MEMORY; }

            memcpy(sbuf->Buffer, r->Buffer, r->Length);

            // 2) 送出；把 sbuf 當作 ClientContext 傳入，待 SEND_COMPLETE 時釋放
            QUIC_STATUS st = MsQuic->StreamSend(Stream, sbuf, 1,
                                                QUIC_SEND_FLAG_NONE, /* or ALLOW_0_RTT */
                                                sbuf /* ClientContext */);
            if (QUIC_FAILED(st)) {
                free(sbuf->Buffer);
                free(sbuf);
                fprintf(stderr, "StreamSend failed: 0x%x\n", st);
                MsQuic->StreamShutdown(Stream, QUIC_STREAM_SHUTDOWN_FLAG_ABORT | QUIC_STREAM_SHUTDOWN_FLAG_IMMEDIATE, 0);
                return st;
            }
        }
        // 回傳 SUCCESS 表示你已處理完接收緩衝 → MsQuic 可回收它們
        return QUIC_STATUS_SUCCESS;
    }

    case QUIC_STREAM_EVENT_SEND_COMPLETE: {
        // 收回你送出的緩衝
        QUIC_BUFFER* sbuf = (QUIC_BUFFER*)Event->SEND_COMPLETE.ClientContext;
        if (sbuf) { free(sbuf->Buffer); free(sbuf); }
        break;
    }

    case QUIC_STREAM_EVENT_PEER_SEND_SHUTDOWN:
        // 對端半關閉發送；這裡回優雅關閉本端發送
        MsQuic->StreamShutdown(Stream, QUIC_STREAM_SHUTDOWN_FLAG_GRACEFUL, 0);
        break;

    case QUIC_STREAM_EVENT_SHUTDOWN_COMPLETE:
        MsQuic->StreamClose(Stream);
        break;

    default:
        break;
    }
    return QUIC_STATUS_SUCCESS;
}

// ==== Connection 回呼：接受對方開的雙向 stream ====
QUIC_STATUS QUIC_API
ConnectionCallback(HQUIC Connection, void* Context, QUIC_CONNECTION_EVENT* Event) {
    (void)Context;
    switch (Event->Type) {
    case QUIC_CONNECTION_EVENT_CONNECTED:
        printf("🔗 QUIC CONNECTED\n");
        break;
    case QUIC_CONNECTION_EVENT_PEER_STREAM_STARTED: {
        // 對方（client）開了一條 stream
        HQUIC stream = Event->PEER_STREAM_STARTED.Stream;
        MsQuic->SetCallbackHandler(stream, (void*)StreamCallback, NULL);
        printf("📥 New stream started\n");
        break;
    }
    case QUIC_CONNECTION_EVENT_SHUTDOWN_INITIATED_BY_PEER:
    case QUIC_CONNECTION_EVENT_SHUTDOWN_INITIATED_BY_TRANSPORT:
        printf("⏹️  Connection shutting down\n");
        break;
    case QUIC_CONNECTION_EVENT_SHUTDOWN_COMPLETE:
        MsQuic->ConnectionClose(Connection);
        printf("👋 Connection closed\n");
        break;
    default:
        break;
    }
    return QUIC_STATUS_SUCCESS;
}

// ==== Listener 回呼：有新連線就接受 ====
_IRQL_requires_max_(PASSIVE_LEVEL)
_Function_class_(QUIC_LISTENER_CALLBACK)
QUIC_STATUS QUIC_API
ListenerCallback(HQUIC ListenerHandle, void* Context, QUIC_LISTENER_EVENT* Event) {
    (void)ListenerHandle; (void)Context;
    switch (Event->Type) {
    case QUIC_LISTENER_EVENT_NEW_CONNECTION: {
        HQUIC conn = Event->NEW_CONNECTION.Connection;
        MsQuic->SetCallbackHandler(conn, (void*)ConnectionCallback, NULL);
        QUIC_STATUS st = MsQuic->ConnectionSetConfiguration(conn, Configuration);
        if (QUIC_FAILED(st)) {
            MsQuic->ConnectionClose(conn);
            return st;
        }
        printf("🛎️  Accept connection\n");
        break;
    }
    default:
        break;
    }
    return QUIC_STATUS_SUCCESS;
}

int main() {
    QUIC_STATUS st = MsQuicOpen2(&MsQuic);
    if (QUIC_FAILED(st)) die("MsQuicOpen2 failed", st);

    // 註冊（可視為全域實例）
    QUIC_REGISTRATION_CONFIG regCfg = { "qchat-srv", QUIC_EXECUTION_PROFILE_LOW_LATENCY };
    st = MsQuic->RegistrationOpen(&regCfg, &Registration);
    if (QUIC_FAILED(st)) die("RegistrationOpen failed", st);

    // 設定 ALPN
    QUIC_BUFFER alpn;
    alpn.Buffer = (uint8_t*)kAlpn;
    alpn.Length = (uint32_t)strlen(kAlpn);

    // 伺服端憑證
    QUIC_CREDENTIAL_CONFIG credCfg;
    memset(&credCfg, 0, sizeof(credCfg));
    credCfg.Type = QUIC_CREDENTIAL_TYPE_CERTIFICATE_FILE;
    credCfg.CertificateFile = &(QUIC_CERTIFICATE_FILE){
        .CertificateFile = kCertFile,
        .PrivateKeyFile  = kKeyFile
    };
    credCfg.Flags = QUIC_CREDENTIAL_FLAG_NONE; // 正常驗證
    // 測試時若客戶端不驗證 CA，可接受自簽即可（client 端要關閉驗證或信任 crt）

    // 建立配置（TLS + ALPN + 傳輸設定）
    QUIC_SETTINGS settings;
    memset(&settings, 0, sizeof(settings));
    settings.IdleTimeoutMs             = 30 * 1000; // 30s idle
    settings.IsSet.IdleTimeoutMs       = TRUE;
    settings.PeerBidiStreamCount       = 100;       // 允許同時多條雙向 stream
    settings.IsSet.PeerBidiStreamCount = TRUE;

    st = MsQuic->ConfigurationOpen(Registration, &alpn, 1, &settings, sizeof(settings), NULL, &Configuration);
    if (QUIC_FAILED(st)) die("ConfigurationOpen failed", st);

    st = MsQuic->ConfigurationLoadCredential(Configuration, &credCfg);
    if (QUIC_FAILED(st)) die("ConfigurationLoadCredential failed", st);

    // 開 listener
    st = MsQuic->ListenerOpen(Registration, ListenerCallback, NULL, &Listener);
    if (QUIC_FAILED(st)) die("ListenerOpen failed", st);

    // 綁定位址與埠（0.0.0.0:4443/udp）
    QUIC_ADDR addr;
    memset(&addr, 0, sizeof(addr));
    addr.Ipv4.sin_family = AF_INET;
    addr.Ipv4.sin_port   = htons(LISTEN_PORT);
    addr.Ipv4.sin_addr.s_addr = htonl(INADDR_ANY); // 0.0.0.0
    // QuicAddrSetFamily(&addr, QUIC_ADDRESS_FAMILY_INET);
    // QuicAddrSetPort(&addr, LISTEN_PORT);
    // QuicAddrSetIPv4(&addr, 0); // 0.0.0.0
    st = MsQuic->ListenerStart(Listener, &alpn, 1, &addr);
    if (QUIC_FAILED(st)) die("ListenerStart failed", st);

    printf("✅ QUIC echo server listening on udp/%s:%d (ALPN=%s)\n", LISTEN_ADDR, LISTEN_PORT, kAlpn);
    printf("   Press Ctrl-C to quit.\n");

    // 簡單阻塞（用 getchar 等待 Ctrl-C 被 docker/前景中斷）
    // 真實專案可用 signalfd/epoll 或 thread 等等
    for (;;) { fgetc(stdin); break; }

    // 收尾
    if (Listener)     MsQuic->ListenerClose(Listener);
    if (Configuration)MsQuic->ConfigurationClose(Configuration);
    if (Registration) MsQuic->RegistrationClose(Registration);
    MsQuicClose(MsQuic);
    return 0;
}

```

### 編譯與運行

```sh
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=/opt/vcpkg/scripts/buildsystems/vcpkg.cmake
make
```

運行之前要透過 openssl (裝 msquic 的時候會同步裝)產一個自簽憑證，因為 QUIC 協議是內嵌TLS 1.3，所以會需要憑證

```sh
openssl req -x509 -newkey rsa:2048 -keyout server.key -out server.crt -days 365 -nodes -subj "/CN=localhost"
```

- 運行

```sh
./quic_echo_server
``` 

## Client端

這邊就一樣在本來的專案中添加

### CMakeLists.txt

```cmake
# 在原先的CMakeLists.txt 下面再補這幾段
add_executable(quic_echo_client
  src/quic_echo_client.c
)

find_package(msquic CONFIG REQUIRED)
target_link_libraries(quic_echo_client PRIVATE msquic)

target_link_libraries(quic_echo_client PRIVATE Threads::Threads)
```

### src/quic_echo_client.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <inttypes.h>
#include <stdbool.h>
#include <pthread.h>
#include <msquic.h>

static const char *kAlpn = "qchat";

const QUIC_API_TABLE *MsQuic = NULL;
HQUIC Registration = NULL;
HQUIC Configuration = NULL;

typedef struct
{
    pthread_mutex_t mu;
    pthread_cond_t cv;
    bool connected;
    bool stream_started;
    bool recv_done;
    bool shutdown_done;
    size_t expected; // 預期收到的 bytes（等同發送長度）
    size_t received; // 累積收到
} Sync;

static void sync_init(Sync *s)
{
    pthread_mutex_init(&s->mu, NULL);
    pthread_cond_init(&s->cv, NULL);
    s->connected = s->stream_started = s->recv_done = s->shutdown_done = false;
    s->expected = s->received = 0;
}
static void sync_destroy(Sync *s)
{
    pthread_mutex_destroy(&s->mu);
    pthread_cond_destroy(&s->cv);
}

_IRQL_requires_max_(PASSIVE_LEVEL)
    _Function_class_(QUIC_STREAM_CALLBACK)
        QUIC_STATUS QUIC_API
    StreamCallback(HQUIC Stream, void *Context, QUIC_STREAM_EVENT *Event)
{
    Sync *sync = (Sync *)Context;
    switch (Event->Type)
    {
    case QUIC_STREAM_EVENT_START_COMPLETE:
        pthread_mutex_lock(&sync->mu);
        sync->stream_started = true;
        pthread_cond_broadcast(&sync->cv);
        pthread_mutex_unlock(&sync->mu);
        break;
    case QUIC_STREAM_EVENT_RECEIVE:
    {
        // 收到 echo
        size_t got = 0;
        for (uint32_t i = 0; i < Event->RECEIVE.BufferCount; ++i)
        {
            const QUIC_BUFFER *b = &Event->RECEIVE.Buffers[i];
            fwrite(b->Buffer, 1, b->Length, stdout); // 顯示回應
            got += b->Length;
        }
        fflush(stdout);

        pthread_mutex_lock(&sync->mu);
        sync->received += got;
        if (sync->expected > 0 && sync->received >= sync->expected)
        {
            sync->recv_done = true;
            pthread_cond_broadcast(&sync->cv);
        }
        pthread_mutex_unlock(&sync->mu);
        return QUIC_STATUS_SUCCESS;
    }
    case QUIC_STREAM_EVENT_SEND_COMPLETE:
        // 可在此了解發送完成
        break;
    case QUIC_STREAM_EVENT_PEER_SEND_SHUTDOWN:
        // 對端關閉了發送方向
        break;
    case QUIC_STREAM_EVENT_SHUTDOWN_COMPLETE:
        MsQuic->StreamClose(Stream);
        break;
    default:
        break;
    }
    return QUIC_STATUS_SUCCESS;
}

_IRQL_requires_max_(PASSIVE_LEVEL)
    _Function_class_(QUIC_CONNECTION_CALLBACK)
        QUIC_STATUS QUIC_API
    ConnectionCallback(HQUIC Connection, void *Context, QUIC_CONNECTION_EVENT *Event)
{
    Sync *sync = (Sync *)Context;
    switch (Event->Type)
    {
    case QUIC_CONNECTION_EVENT_CONNECTED:
        pthread_mutex_lock(&sync->mu);
        sync->connected = true;
        pthread_cond_broadcast(&sync->cv);
        pthread_mutex_unlock(&sync->mu);
        printf("🔗 QUIC connected\n");
        break;
    case QUIC_CONNECTION_EVENT_SHUTDOWN_INITIATED_BY_PEER:
    case QUIC_CONNECTION_EVENT_SHUTDOWN_INITIATED_BY_TRANSPORT:
        printf("⏹️  connection shutting down\n");
        break;
    case QUIC_CONNECTION_EVENT_SHUTDOWN_COMPLETE:
        pthread_mutex_lock(&sync->mu);
        sync->shutdown_done = true;
        pthread_cond_broadcast(&sync->cv);
        pthread_mutex_unlock(&sync->mu);
        MsQuic->ConnectionClose(Connection);
        printf("👋 connection closed\n");
        break;
    default:
        break;
    }
    return QUIC_STATUS_SUCCESS;
}

static void die(const char *msg, QUIC_STATUS st)
{
    fprintf(stderr, "%s: 0x%x\n", msg, st);
    exit(1);
}

int main(int argc, char **argv)
{
    const char *host = "127.0.0.1";
    uint16_t port = 4443;
    const char *msg = "hello over quic\n";

    for (int i = 1; i < argc; i++)
    {
        if (!strcmp(argv[i], "--host") && i + 1 < argc)
            host = argv[++i];
        else if (!strcmp(argv[i], "--port") && i + 1 < argc)
            port = (uint16_t)atoi(argv[++i]);
        else if (!strcmp(argv[i], "--msg") && i + 1 < argc)
            msg = argv[++i];
        else if (!strcmp(argv[i], "-h") || !strcmp(argv[i], "--help"))
        {
            printf("Usage: %s [--host HOST] [--port PORT] [--msg STR]\n", argv[0]);
            return 0;
        }
    }

    QUIC_STATUS st = MsQuicOpen2(&MsQuic);
    if (QUIC_FAILED(st))
        die("MsQuicOpen2", st);

    QUIC_REGISTRATION_CONFIG regCfg = {"qchat-cli", QUIC_EXECUTION_PROFILE_LOW_LATENCY};
    st = MsQuic->RegistrationOpen(&regCfg, &Registration);
    if (QUIC_FAILED(st))
        die("RegistrationOpen", st);

    QUIC_SETTINGS settings;
    memset(&settings, 0, sizeof(settings));
    // 可視需要調整設定

    QUIC_BUFFER alpn;
    alpn.Buffer = (uint8_t *)kAlpn;
    alpn.Length = (uint32_t)strlen(kAlpn);

    st = MsQuic->ConfigurationOpen(Registration, &alpn, 1, &settings, sizeof(settings), NULL, &Configuration);
    if (QUIC_FAILED(st))
        die("ConfigurationOpen", st);

    // 客戶端憑證設定：不帶客戶端證書，並關閉伺服端驗證（僅限開發測試）
    QUIC_CREDENTIAL_CONFIG cred;
    memset(&cred, 0, sizeof(cred));
    cred.Type = QUIC_CREDENTIAL_TYPE_NONE;
    cred.Flags = QUIC_CREDENTIAL_FLAG_CLIENT | QUIC_CREDENTIAL_FLAG_NO_CERTIFICATE_VALIDATION;

    st = MsQuic->ConfigurationLoadCredential(Configuration, &cred);
    if (QUIC_FAILED(st))
        die("ConfigurationLoadCredential", st);

    // 建立連線
    HQUIC conn = NULL;
    Sync sync;
    sync_init(&sync);

    st = MsQuic->ConnectionOpen(Registration, ConnectionCallback, &sync, &conn);
    if (QUIC_FAILED(st))
        die("ConnectionOpen", st);

    st = MsQuic->ConnectionStart(
        conn,
        Configuration,              // v2 需帶 Configuration
        QUIC_ADDRESS_FAMILY_UNSPEC, // 或 QUIC_ADDRESS_FAMILY_INET
        host,
        port // uint16_t, host byte order
    );
    if (QUIC_FAILED(st))
        die("ConnectionStart", st);

    // 等待連上
    pthread_mutex_lock(&sync.mu);
    while (!sync.connected)
        pthread_cond_wait(&sync.cv, &sync.mu);
    pthread_mutex_unlock(&sync.mu);

    // 開一條雙向 stream，送資料
    HQUIC stream = NULL;
    st = MsQuic->StreamOpen(conn, QUIC_STREAM_OPEN_FLAG_NONE, StreamCallback, &sync, &stream);
    if (QUIC_FAILED(st))
        die("StreamOpen", st);

    st = MsQuic->StreamStart(stream, QUIC_STREAM_START_FLAG_NONE);
    if (QUIC_FAILED(st))
        die("StreamStart", st);

    // 等 stream 開好
    pthread_mutex_lock(&sync.mu);
    while (!sync.stream_started)
        pthread_cond_wait(&sync.cv, &sync.mu);
    pthread_mutex_unlock(&sync.mu);

    // 準備送資料
    QUIC_BUFFER sbuf;
    sbuf.Length = (uint32_t)strlen(msg);
    sbuf.Buffer = (uint8_t *)msg;

    pthread_mutex_lock(&sync.mu);
    sync.expected = sbuf.Length; // 等同要收到的 echo bytes
    pthread_mutex_unlock(&sync.mu);

    st = MsQuic->StreamSend(stream, &sbuf, 1, QUIC_SEND_FLAG_ALLOW_0_RTT, NULL);
    if (QUIC_FAILED(st))
        die("StreamSend", st);

    // 等收到完整 echo 或 3 秒超時
    struct timespec ts;
    clock_gettime(CLOCK_REALTIME, &ts);
    ts.tv_sec += 3;

    pthread_mutex_lock(&sync.mu);
    while (!sync.recv_done)
    {
        if (pthread_cond_timedwait(&sync.cv, &sync.mu, &ts) == ETIMEDOUT)
            break;
    }
    pthread_mutex_unlock(&sync.mu);

    // 優雅關閉 stream 與連線
    MsQuic->StreamShutdown(stream, QUIC_STREAM_SHUTDOWN_FLAG_GRACEFUL, 0);
    MsQuic->ConnectionShutdown(conn, QUIC_CONNECTION_SHUTDOWN_FLAG_NONE, 0);

    // 等連線完全關閉
    pthread_mutex_lock(&sync.mu);
    while (!sync.shutdown_done)
        pthread_cond_wait(&sync.cv, &sync.mu);
    pthread_mutex_unlock(&sync.mu);

    sync_destroy(&sync);
    MsQuic->ConfigurationClose(Configuration);
    MsQuic->RegistrationClose(Registration);
    MsQuicClose(MsQuic);
    return 0;
}
```

### 編譯與運行

```sh
# 在打包一次
make

./quic_echo_client --host 127.0.0.1 --port 4443 --msg "hello over quic\n"
```

## 結論

今年的鐵人賽就到這邊告一段落，不得不說有AI的幫助，整個情況都不太一樣，成果是可以比較快看到，但相對的應該要了解的東西，並不會因AI的幫忙而更輕鬆，反而還要去習慣他給予的東西，雖然可以透過提示，讓他說話傾向於能使用者更好吸收，但對於需要理解跟內化的部分，還是回歸到自己本身，這才是有沒有AI都需要重視的核心部分，以及這次的參賽雖說是志在參加，但不少文章這次確實是流於表面，雖然有個基本Sample，但我認為有價值的還是衍生的部分，也是激發我有興趣做的一塊，但這次也是給自己找了又累又懶又怕超時的藉口，自己是有察覺這一點，但這也是我未來可以更進步的地方，總之這次的鐵人賽是有收穫的，完賽啦。
