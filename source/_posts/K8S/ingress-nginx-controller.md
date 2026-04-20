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

這樣有風險因為把入口點集中在一個地方，所以通常會在 改為使用 daemonsets 的方式讓 ingress nginx controller 灑到所有的 node 上，然後再進入點放一個 nginx 或是其他的 proxy 來負責倒流進入 private k8s。

## Install K8s Dashboard


## Install Oauth2Proxy with gitlab to authorizate user

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oauth2-proxy
  template:
    metadata:
      labels:
        app: oauth2-proxy
    spec:
      containers:
      - name: oauth2-proxy
        image: quay.io/oauth2-proxy/oauth2-proxy:v7.6.0
        # 如果在公司私有環境，記得換成內部 Harbor 的鏡像路徑
        args:
        - --provider=oidc
        - --reverse-proxy=true      # <--- 必加！告訴 Proxy 它在 Ingress 後面
        - --email-domain=* # 限制特定的 email 網域可輸入公司網域
        - --upstream=file:///dev/null    # 因為我們是用 Ingress 轉發，這裡設為 null
        - --http-address=0.0.0.0:4180
        - --oidc-issuer-url=https://gitlab.com # 或是公司內部 GitLab 地址
        - --cookie-secure=false
        - --whitelist-domain=kubedashboard.localhost # 加入這行
        - --cookie-refresh=1h
        - --redirect-url=https://kubedashboard.localhost/oauth2/callback
        - --set-authorization-header=true
        - --pass-authorization-header=true
        - --scope=openid profile email
        env:
        - name: OAUTH2_PROXY_CLIENT_ID
          value: "8804a714b9f0a8ba58d0f58d22179cae34ffa2b66a7dcf4660b0a4ffe7204299"
        - name: OAUTH2_PROXY_CLIENT_SECRET
          value: "gloas-f90c1402e620e971231526a6eb0a1e126ad781d57caa9ebefeda499394f11de9"
        - name: OAUTH2_PROXY_COOKIE_SECRET
          value: "gJRFrVwfN8vJYB178rzdkw=="
          # value: "ApLd9UBOd8YwaJcXqn/DJA==" # 例如：python -c 'import os,base64; print(base64.b64encode(os.urandom(16)).decode())'

---
apiVersion: v1
kind: Service
metadata:
  name: oauth2-proxy
  namespace: kubernetes-dashboard
spec:
  ports:
  - name: http
    port: 4180
    protocol: TCP
    targetPort: 4180
  selector:
    app: oauth2-proxy
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    # 1. 外部認證檢查 URL (指向 Service 的內部 DNS)
    nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy.kubernetes-dashboard.svc.cluster.local:4180/oauth2/auth"

    # 2. 未登入時跳轉的 SSO 登入起始頁面
    nginx.ingress.kubernetes.io/auth-signin: "https://kubedashboard.localhost/oauth2/start?rd=$scheme://$host$request_uri$is_args$args"

    # 3. 如果 Dashboard 服務本身是 HTTPS (預設通常是)
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # 將 OAuth2 Proxy 驗證後的 Authorization Header 傳給後端
    nginx.ingress.kubernetes.io/auth-response-headers: "Authorization"
spec:
  ingressClassName: nginx
  rules:
  - host: kubedashboard.localhost
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard-kong-proxy
            port:
              number: 443
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: oauth2-proxy-public
  namespace: kubernetes-dashboard
  # 這裡不要放任何 auth-url 或 auth-signin！
spec:
  ingressClassName: nginx
  rules:
  - host: kubedashboard.localhost
    http:
      paths:
      - path: /oauth2
        pathType: Prefix
        backend:
          service:
            name: oauth2-proxy
            port:
              number: 4180

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sso-admin-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin # 測試用，給予最大權限
subjects:
- kind: User
  name: "poumason@live.com" # 這裡填寫你 GitLab 帳號的 Email
  apiGroup: rbac.authorization.k8s.io
```

# References
- [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Ingress Controller 101：輕鬆學會基本技巧](https://weng-albert.medium.com/ingress-controller-101-%E8%BC%95%E9%AC%86%E5%AD%B8%E6%9C%83%E5%9F%BA%E6%9C%AC%E6%8A%80%E5%B7%A7-82bb0285c382)
- [Kind, Keycloak — Securing Kubernetes api server with OIDC](https://medium.com/@charled.breteche/kind-keycloak-securing-kubernetes-api-server-with-oidc-371c5faef902)
- [Kubernetes RBAC 101: 如何通过 OIDC 强化集群安全](https://www.cloudnative101.net/posts/kubernetes-rbac-oidc-security-guide/)
- [云原生小技巧 #4：OrbStack — 本地 K8s 环境的域名映射优化，开发者的新宠](https://mp.weixin.qq.com/s/f3vFf_GkURscjOwy2vXP1Q)
- [Protect Kubernetes Dashboard using oauth2-proxy and Keycloak](https://www.enricobassetti.it/2021/04/protect-kubernetes-dashboard-using-oauth2-proxy-and-keycloak/)
- [佈署 & 存取 Kubernetes Dashboard](https://godleon.github.io/blog/Kubernetes/k8s-Deploy-and-Access-Dashboard/)
- [Configure GitLab as an OAuth 2.0 authentication identity provider](https://docs.gitlab.com/integration/oauth_provider/#view-all-authorized-applications)