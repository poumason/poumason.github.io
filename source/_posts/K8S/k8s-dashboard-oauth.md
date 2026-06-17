---
title: 介紹 K8s Dashboard 如何串接 Oauth2Proxy 機制
date: 2026-04-20 22:39:26
categories: K8S
tags:
    - oidc
    - K8S
    - ingress nginx
    - oauth2proxy
---
介紹如何使用 K8s Dashboard 加 Oauth2Proxy 進行 user 身分的驗證與管理。
<!-- more-->
本片的內容會搭配 ingress-nginx 一起使用，所以可以先參考 []() 將它安裝起來。

## 準備環境

### kind + OIDC
由於測試使用的是 docker + kind，因此需要先對 kind 建立 cluster 時加入一些特殊的設定，如下：

- 宣告對應 port 號：讓連線時只能使用 80/443
- 加入 OIDC 的設定：讓 kube api server 能驗證登入的 token

建立一個 kind-config.yaml 作為示範：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        # --- OIDC 設定：讓 K8s 認識 GitLab ---
        oidc-issuer-url: "https://gitlab.com"
        oidc-client-id: ""
        oidc-username-claim: "email"
        # oidc-groups-claim: "groups"
    # kind: InitConfiguration
    # nodeRegistration:
    #   # 讓 Ingress Controller 知道要在哪個 node 執行
    #   node-labels: "ingress-ready=true"
  # 這裡最重要：將主機的 80/443 對應到節點容器內
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
```

加入完成後，執行 `kind create cluster --config kind-config.yaml` 建立起 cluster。

建立完成後，可使用 `` 檢查剛加入的 OIDC 設定是否有被寫入。

### kubeadmin + OIDC

```yaml
```


## Install K8s Dashboard
```yaml
```


## Install Oauth2Proxy with gitlab to authorizate user

```yaml
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

## 補充
建立完成。部署前需要三個步驟：

1. 產生自簽 TLS 憑證

openssl req -x509 -nodes -newkey rsa:2048 -days 3650 \
  -keyout dex.key -out dex.crt \
  -subj "/CN=dex.localhost" \
  -addext "subjectAltName=DNS:dex.localhost"

產生後把憑證填入 dex-tls Secret（或改用 cert-manager 自動管理）：
# 直接用 kubectl 建立比較省事
kubectl create secret tls dex-tls \
  --cert=dex.crt --key=dex.key \
  -n dex --dry-run=client -o yaml | kubectl apply -f -

2. 在 GitLab 建立新的 OAuth Application

GitLab → User Settings → Applications：
- Redirect URI: https://dex.localhost/callback
- Scopes: openid, profile, email

把新的 client ID / secret 更新到 dex-gitlab-secret。

3. 更新 kind-config.yaml 指向 Dex

oidc-issuer-url: "https://dex.localhost"
oidc-client-id: "kubernetes-dashboard"
oidc-username-claim: "email"

注意：kind-config.yaml 的 OIDC 設定改了需要重建 cluster（kind delete cluster && kind create cluster --config kind-config.yaml）。

4. 更新 oauth2proxy-deployment.yaml 改指向 Dex

- --oidc-issuer-url=https://dex.localhost
- --scope=openid p # Dex 支援offline_access！
- --cookie-refresh

Dex 自己發的 tokenoffline_access，oauth2-proxy 的 cookie refresh 就能正常運作了。

## References

- [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Ingress Controller 101：輕鬆學會基本技巧](https://weng-albert.medium.com/ingress-controller-101-%E8%BC%95%E9%AC%86%E5%AD%B8%E6%9C%83%E5%9F%BA%E6%9C%AC%E6%8A%80%E5%B7%A7-82bb0285c382)
- [Kind, Keycloak — Securing Kubernetes api server with OIDC](https://medium.com/@charled.breteche/kind-keycloak-securing-kubernetes-api-server-with-oidc-371c5faef902)
- [Kubernetes RBAC 101: 如何通过 OIDC 强化集群安全](https://www.cloudnative101.net/posts/kubernetes-rbac-oidc-security-guide/)
- [云原生小技巧 #4：OrbStack — 本地 K8s 环境的域名映射优化，开发者的新宠](https://mp.weixin.qq.com/s/f3vFf_GkURscjOwy2vXP1Q)
- [Protect Kubernetes Dashboard using oauth2-proxy and Keycloak](https://www.enricobassetti.it/2021/04/protect-kubernetes-dashboard-using-oauth2-proxy-and-keycloak/)
- [佈署 & 存取 Kubernetes Dashboard](https://godleon.github.io/blog/Kubernetes/k8s-Deploy-and-Access-Dashboard/)
- [Configure GitLab as an OAuth 2.0 authentication identity provider](https://docs.gitlab.com/integration/oauth_provider/#view-all-authorized-applications)