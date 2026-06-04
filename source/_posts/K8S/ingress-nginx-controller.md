---
title: 介紹如何安裝 ingress nginx controller
date: 2025-12-05 22:39:26
categories: K8S
tags:
    - ingress
    - K8S
    - ingress nginx controller
---
介紹如何使用 ingress nginx controller 與設定 ingress.
<!-- more-->

## Environment of Cluster
If you want to test ingress nginx controller in local machine. please using docker and kind to create cluster.

create the kind-confing.yaml for create cluster of kind.

```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      # 讓 Ingress Controller 知道要在哪個 node 執行
      node-labels: "ingress-ready=true"
  # 這裡最重要：將主機的 80/443 對應到節點容器內
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```
If you want to integrate OIDC, must add api server configutation with OIDC provider.
```
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        # --- OIDC 設定：讓 K8s 認識 GitLab ---
        oidc-issuer-url: "https://gitlab.com"
        oidc-client-id: "你的_GITLAB_CLIENT_ID"
        oidc-username-claim: "email"
        oidc-groups-claim: "groups"
```
Then, execute kind command to create new cluster.

```
kind create cluster --config kind-config.yaml
```

because the kind container must be public host port with container port.


## Install Ingress Nginx Controller
For local test, I using docker + kind to create cluster, and I use kind verions of ingress-nginx.

### for kind
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

### for cloud

If you want to install ingress-nginx on private k8s, please using the version:
```
kubectl apply -y https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.1/deploy/static/provider/cloud/deploy.yaml
```

based on environment to choice fit your environment.

當 ingress nginx 安裝好了後，它會產生幾個東西：
1. namespace 預設使用 ingress-nginx
1. 建立一個 ingress nginx controller 負責所有任務
1. 建立一個 預設的 ingress class 使用 nginx 為命名


預設 ingress nginx controller 利用 deployment 部署，所以通常會使用 pod 所屬的 node IP 為主。

這樣有風險因為把入口點集中在一個地方，所以通常會在改為使用 daemonsets 的方式讓 ingress nginx controller 灑到所有的 node 上，然後再進入點放一個 nginx 或是其他的 proxy 來負責倒流進入 private k8s。

### Check result



## Add Ingress
```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
  namespace: kubernetes-dashboard
spec:
  ingressClassName: nginx
  rules:
  - host: kubedashboard2.localhost
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: kubernetes-dashboard-kong-proxy
              port:
                number: 443
```

## 如何在 Mac 上搭配 kind 取得 external-ip
Kind 官方提供了一個名為 cloud-provider-kind 的工具，它專門用來在 local 環境為 LoadBalancer 類型的 Service 分配 127.0.0.1（localhost）的外部 IP。

1. 安裝 cloud-provider-kind
使用 Homebrew 即可直接安裝：

```Bash
brew install cloud-provider-kind
```

2. 啟動服務（需保持終端機開啟）
在您的電腦上打開一個獨立的終端機視窗，運行以下指令：

```Bash
cloud-provider-kind
```
這個程式會常駐並監聽您的 Kind 叢集。此時，您再去檢查 Ingress Service：

```Bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```
您會發現 EXTERNAL-IP 成功從 <pending> 變成了 127.0.0.1！


# References
- [kind]()