# Kubernetes 體驗 2

今天來接著體驗k8s的套件管理器helm，並配合helm chart，讓我們可以透過模板化來部屬我們的服務，也體驗一下Racher，一個管理k8s的GUI工具。

## 安裝 helm

- 一樣我們先進入wsl的ubuntu裡面

```sh
wsl -d Ubuntu

curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

- 新增與更新 Helm 倉庫

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

## 建立 k8s 叢集

- 跟昨天一樣的方式

```sh
# 在wsl ubuntu裡面，我會習慣移動到/home/user 的下面可以進行
# 建立叢集
kind create cluster --name demo

kubectl get nodes
```

## 使用 helm

- 透過 helm 安裝一個nginx

```sh
helm install my-nginx bitnami/nginx

# 驗證
helm list
kubectl get pods
kubectl get svc my-nginx

kubectl port-forward svc/my-nginx 8080:80
# http://localhost:8080 這樣就可以看到nginx了
```

## 明日接續

今天上班下來感到十分疲憊，所以今天的內容就先到 helm 的部分，明天再來補齊 helm chart 跟 racher 的部分。
