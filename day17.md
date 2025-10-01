# Kubernetes 體驗

今天就來接著體驗管理容器的系統，k8s Kubernetes。

## Kubernetes 安裝

在研究了一下不同k8s的安裝後，決定使用kind (kubernetes in docker)，來安裝k8s。

1. 我是使用windows作業系統，因為這邊會使用wsl，windows裡面的linux系統來進行安裝，所以要先安裝 WSL2 + Ubuntu

```powershell
# 會安裝Ubuntu
wsl --install

# 進入Ubuntu
wsl -d Ubuntu
```

2. 安裝kubectl，kubectl是跟k8s溝通的cli工具

```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# 測試是否安裝成功
kubectl version --client
```

3. 安裝kind

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/

# 測試是否安裝成功
kind version
```

4. 建立一個單節點k8s cluster

```sh
kind create cluster --name demo

# 確認是否建立成功
kubectl get nodes
```

5. 測試部屬一個服務

```sh
kubectl create deployment web --image=nginx:1.25
kubectl expose deployment web --port=80 --type=NodePort
kubectl get svc web

# 對外提供8080
kubectl port-forward deploy/web 8080:80

# http://localhost:8080/ 用瀏覽器打開，就會看到nginx的頁面
```

6. 刪除cluster

```sh
kind delete cluster --name demo
```

## 結論

今天就簡單體驗一下kubernete的基礎安裝與使用，明日接續。
