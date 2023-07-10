---
title: "K3s traefik expose k8s dashboard"
date: 2023-07-10T15:34:20+08:00
---
#### 安装dashboard
参考https://docs.k3s.io/installation/kube-dashboard。如果获取k8s dashboard yaml出现问题，可能是这个版本本身还不存在，可直接看k8s官网使用的是哪个版本的dashboard yaml。`k3s kubectl -n kubernetes-dashboard create token admin-user`生成的token在登录时需要用到

#### 准备证书
1. 部署完毕后，生成证书，这里生成了泛域名证书，方便使用
```bash
openssl req -newkey rsa:2048 -nodes -keyout root.key -x509 -days 365 -out root.crt -subj "/CN=Root CA"
openssl req -newkey rsa:2048 -nodes -keyout domain.key -out domain.csr -subj "/CN=home.site" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:home.site,DNS:*.home.site"))
openssl x509 -req -in domain.csr -CA root.crt -CAkey root.key -CAcreateserial -out domain.crt -days 365 -extensions SAN -extfile <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:home.site,DNS:*.home.site"))
```
2. 将公私钥添加至k8s secret `k3s kubectl create secret tls dashboard-ingress-certs --key domain.key --cert domain.crt -n kubernetes-dashboard`

#### 使用traefik暴露k8s dashboard service
ingress objects是k8s native objects，ingressRoutes是traefik自己定义的一类资源，有些功能用起来更简单。因此这里我直接使用ingressRoutes,`k3s kubectl apply -f ingressroute.yaml`
```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: ServersTransport
metadata:
  name: traefik-servers-transport
  namespace: kubernetes-dashboard
spec:
  serverName: "test"
  insecureSkipVerify: true
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: kubernetes-dashboard-route
  namespace: kubernetes-dashboard
spec:
  entryPoints:
    - websecure
  tls:
    secretName: dashboard-ingress-certs
  routes:
  - match: Host(`dashboard.home.site`)
    kind: Rule
    services:
      - name: kubernetes-dashboard
        namespace: kubernetes-dashboard
        port: 443
        serversTransport: traefik-servers-transport
```
在dns服务器或者在hosts配置解析后，即可访问

#### 参考
1. [使用traefik2暴露k8s-dashboard](https://blog.zeromake.com/pages/traefik2-configure-dashboard/)
2. [customize taefik2 config](https://github.com/k3s-io/k3s/issues/4589)
3. [difference between ingress and ingressRoute](https://community.traefik.io/t/what-is-the-difference-between-ingress-and-ingressroute/10864)
4. [ingressRoute文档](https://doc.traefik.io/traefik/providers/kubernetes-crd/)
