# Kubernetes 體驗 3

今天來接續體驗昨天提到的helm chart以及racher。

## 今日目標

透過 windows 的 wsl 裡面安裝的 kind 建立 k8s cluster，並透過 helm chart 安裝 racher (k8s 叢集管理GUI)。

1. 建立 Kind Cluster

```sh
kind create cluster --name rancher-demo

# 建立成功會看到 kind 提醒可以用下面指令確認 k8s
kubectl cluster-info --context kind-rancher-demo
```

2. 安裝 helm

```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

3. 加入 Racher Helm Repo

```sh
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

4. 建立 cattle-system namespace

```sh
# 這是為了要給 racher 建立與 racher 無關的Pod/Service做的隔離，這樣 cluster 裡面其他應用就不會被不相干的namespace影響
kubectl create namespace cattle-system
```

5. 安裝 Cert-Manager（Rancher 需要）

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# 確認每個 pod 都變成 Running 就安裝好了
kubectl get pods -n cert-manager
```

6. 安裝 Racher

```sh
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.localhost \
  --set bootstrapPassword=admin

# 確認每個 pod 都變成 Running 就安裝好了
kubectl -n cattle-system get pods
```

7. 提供 Racher UI 對外接口

```sh
# Racher 會強制使用 https
kubectl -n cattle-system port-forward deploy/rancher 8080:80 8443:443
```

[Racher UI](https://rancher.localhost:8443) 點開就能看到畫面了。

## 明日接續

Racher的內容其實非常豐富，但這次只是我的一個以前用過的重新體驗，所以就點到安裝為止，之後有機會再繼續深挖Racher的其他功能，明日就回到之前提到的使用Istio做grpc的service discover跟service registry。
