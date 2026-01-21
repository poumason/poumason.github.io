---
title: 學習如何使用 dnsmasq 在 local 建立 DNS server
date: 2026-01-21 22:39:26
categories: K8S
tags:
    - dnsmasq
    - DNS
    - K8S
---
由於在自己的 Macbook 建立了 K8s，搭配 Ingress nginx Controller 為每個 Pod 建立對應的 ingress host，發現每次都要修改 `/etc/hosts` 非常辛苦，因此問了 Gemini 它推薦了可以使用 dnsmasq 解決問題，馬上就紀錄下來。
<!-- more-->

預期使用 `*.localhost` 為我的 domain，那如果 k8s 裡面有多個 AP，我就可以設定成： kubedash.localhost 或是 myservice.localhost，那搭配 dnsmasq 要怎麼做呢？

## 步驟

### 1.利用 brew 安裝 dnsmasq
```
brew install dnsmasq
```

### 2.建立 Dnsmasq 的設定檔 (例如 dnsmasq.conf)，加入監聽的 address 與解析本地的 domain
```
echo "address=/.localhost/127.0.0.1" >> $(brew --prefix)/etc/dnsmasq.conf
```

### 3. 建立 DNS 配置文件，將指定 domain 指向本地的 127.0.0.1
```
sudo mkdir -p /etc/resolver
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/localhost'
```
可利用 ping 的方式確定設定是否生效，例如：
```
ping hello.localhost
```

### 4. 啟動 dnsmasq
```
sudo brew services start dnsmasq
```

### 5. 設定 ingress
以下是範例
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vscode-ingress
  annotations:
    # 這是重點：讓 code-server 的 Websocket 能正常運作
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  rules:
  - host: vscode.localhost
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: vscode-svc
              port:
                number: 80
```

以上是簡單的筆記，相關 dnsmasq 與 local DNS server 相關設定，之後再補上。🙏

# References
- [使用 dnsmasq 配置内部 DNS server](https://tomme.me/use-dnsmasq-as-internal-dsn-server/)