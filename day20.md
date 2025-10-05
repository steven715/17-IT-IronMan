# Kubernetes 體驗 4

今天來到k8s體驗的最後一站，其實是要來補之前grpc遺漏的istio。

## 今日目標

建立一個k8s叢集，將之前部屬的grpc server搭配istio，做一個有service discovery跟service registry的功能體驗。

## 前置準備

之前在day13有準備一個grpc 的image，因為k3s不會直接讀取我們本機的docker image，所以要先導出來，之後再匯入k3s

```sh
docker save grpc_coredns_consul_cpp_grpc:latest -o /tmp/grpcimg.tar
```

## 建立 k8s 叢集

今天改使用 k3s (racher 公司提供的一個輕量級的k8s版本)，因為我在 kind 的使用上常遇到建完一個 cluster ，刪掉再建一個新的都會出錯，所以改用 k3s 。

```sh
# 一行裝好 k3s（關閉 traefik）
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik" sh -

# 匯入 grpc image
sudo k3s ctr images import /tmp/grpcimg.tar
# 驗證
sudo k3s kubectl get nodes
# 後續把 kubectl 指到 k3s
sudo k3s kubectl config view --raw > ~/.kube/config
kubectl get nodes
```

## 安裝Istio

```sh
# 下載 istioctl（可換版）
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.22.2 sh -
export PATH="$PATH:$(pwd)/istio-1.22.2/bin"

# 安裝
istioctl install --set profile=demo -y

# 建立 namespace 並開啟 sidecar 自動注入
kubectl create namespace grpc-mesh
kubectl label namespace grpc-mesh istio-injection=enabled
```

## 部屬 grpc

```yaml
# grpc-server.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpp-grpc-dev
  namespace: grpc-mesh
  labels:
    app: greeter
    version: v1
  annotations:
    sidecar.istio.io/inject: "true"   
spec:
  restartPolicy: Never
  containers:
  - name: cpp-grpc
    image: grpc_coredns_consul_cpp_grpc:latest
    imagePullPolicy: IfNotPresent
    command: ["/bin/bash", "-c", "--"]
    args: ["sleep infinity"]          
    tty: true
    stdin: true
    ports:
    - containerPort: 50051
      name: grpc                     
    volumeMounts:
    - name: grpc-src
      mountPath: /root/grpc-cpp       
  volumes:
  - name: grpc-src
    hostPath:
      path: /mnt/c/Users/Asus/steven/grpc-cpp
      type: DirectoryOrCreate
---
apiVersion: v1
kind: Service
metadata:
  name: greeter
  namespace: grpc-mesh
  labels:
    app: greeter
spec:
  selector:
    app: greeter
  ports:
  - name: grpc
    port: 50051
    targetPort: 50051

```

- 部屬指令

```sh
kubectl apply -f grpc-server.yaml
kubectl -n grpc-mesh get pods,svc
```

- 進入Pod裡面編譯並啟動Server，這邊會這樣做是這個Pod等等還會拿來當Client端使用

```sh
# 進入 Pod 裡面
kubectl -n grpc-mesh exec -it pod/cpp-grpc-dev -- /bin/bash

# 編譯
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=/opt/vcpkg/scripts/buildsystems/vcpkg.cmake
make

# 啟動server
./server
```

## 服務註冊/發現

```yaml
# istio-routing.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: greeter
  namespace: grpc-mesh
spec:
  host: greeter.grpc-mesh.svc.cluster.local
  trafficPolicy:
    loadBalancer: { simple: ROUND_ROBIN }
  subsets:
  - name: v1
    labels: { version: v1 }
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: greeter
  namespace: grpc-mesh
spec:
  hosts:
  - greeter.grpc-mesh.svc.cluster.local
  http:
  - route:
    - destination:
        host: greeter.grpc-mesh.svc.cluster.local
        subset: v1
      weight: 100

```

部屬指令

```sh
kubectl apply -f istio-routing.yaml
```

## 呼叫測試

```sh
# 進入 Pod 裡面啟動 client 端程式
kubectl -n grpc-mesh exec -it pod/cpp-grpc-dev -- /bin/bash
cd /root/grpc-cpp/build/
./client greeter.grpc-mesh.svc.cluster.local:50051 Alice
```

## 清理環境

```sh
kubectl delete -f istio-routing.yaml
kubectl delete -f grpc-server.yaml
istioctl uninstall --purge -y
sudo /usr/local/bin/k3s-uninstall.sh
```

## 結論

重新回來體驗了kubernetes的相關功能，也讓我有機會把以前的知識給補回來，不過這次的Istio也是很多東西沒有體驗很完整，就來日方長，以後再來體驗這塊。
