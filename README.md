> [!TIP]
> <a href="https://ondrej-sika.cz/skoleni/kubernetes"><img src="https://img.shields.io/badge/Školení%20Kubernetes-by%20Ondřej%20Šika-blue?style=for-the-badge&logo=kubernetes&logoColor=white" style="width: 100%; height: auto;" alt="Školení Kubernetes by Ondřej Šika" /></a>
>
> 🚀 Naučte se, jak efektivně spravovat kontejnery s **Kubernetes**! Praktické školení vedené [Ondřejem Šikou](https://ondrej-sika.cz).
>
> 👉 [Více informací a registrace zde](https://ondrej-sika.cz/skoleni/kubernetes)

# 2025-12-09_rke2_cluster_setup

## Setup First Master

- `ssh root@m0.sikademo.com`

```bash
curl -fsSL https://raw.githubusercontent.com/sikalabs/slu/master/install.sh | sh
apt-get install -y open-iscsi
curl -sfL https://get.rke2.io | sh -
mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
token: 6Caa_W2LN_X9HG_ItNV_bmhx_4Nyl_nuAA_LzM8
tls-san:
    - m0.sikademo.com
    - m1.sikademo.com
    - m2.sikademo.com
node-taint:
    - "CriticalAddonsOnly=true:NoExecute"
disable:
    - rke2-ingress-nginx
EOF
systemctl enable rke2-server.service
systemctl start rke2-server.service
```

Validate

```bash
/var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml get nodes
```

Install kubectl and helm to default path

```bash
slu install-bin kubectl
slu install-bin helm
ln -s /usr/local/bin/kubectl /usr/local/bin/k
```

Create ~/.kube/config

```bash
mkdir -p $HOME/.kube
sudo cp /etc/rancher/rke2/rke2.yaml $HOME/.kube/config
```

## Other Masters

- `ssh root@m1.sikademo.com`
- `ssh root@m2.sikademo.com`

```bash
curl -fsSL https://raw.githubusercontent.com/sikalabs/slu/master/install.sh | sh
apt-get install -y open-iscsi
curl -sfL https://get.rke2.io | sh -
mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
server: https://m0.sikademo.com:9345
token: 6Caa_W2LN_X9HG_ItNV_bmhx_4Nyl_nuAA_LzM8
tls-san:
    - m0.sikademo.com
    - m1.sikademo.com
    - m2.sikademo.com
node-taint:
    - "CriticalAddonsOnly=true:NoExecute"
disable:
    - rke2-ingress-nginx
EOF
systemctl enable rke2-server.service
systemctl start rke2-server.service
```

## Workers

- `ssh root@w0.sikademo.com`
- `ssh root@w1.sikademo.com`
- `ssh root@w2.sikademo.com`
- `ssh root@w3.sikademo.com`

```bash
curl -fsSL https://raw.githubusercontent.com/sikalabs/slu/master/install.sh | sh
apt-get install -y open-iscsi
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
mkdir -p /etc/rancher/rke2/
cat << EOF > /etc/rancher/rke2/config.yaml
server: https://m0.sikademo.com:9345
token: 6Caa_W2LN_X9HG_ItNV_bmhx_4Nyl_nuAA_LzM8
EOF
systemctl enable rke2-agent.service
systemctl start rke2-agent.service
```

## Install NGINX Ingress Controller

- `ssh root@m0.sikademo.com`

```bash
helm upgrade --install \
ingress-nginx ingress-nginx \
--repo https://kubernetes.github.io/ingress-nginx \
--create-namespace \
--namespace ingress-nginx \
--set controller.service.type=ClusterIP \
--set controller.ingressClassResource.default=true \
--set controller.kind=DaemonSet \
--set controller.hostPort.enabled=true \
--set controller.metrics.enabled=true \
--set controller.config.use-proxy-protocol=false \
--wait
```

## Install Cert Manager

- `ssh root@m0.sikademo.com`

```bash
helm upgrade --install \
cert-manager cert-manager \
--repo https://charts.jetstack.io \
--create-namespace \
--namespace cert-manager \
--set crds.enabled=true \
--wait
```

Install ClusterIssuer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: lets-encrypt-slu@sikamail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-issuer-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

## Install Longhorn

- `ssh root@m0.sikademo.com`

```bash
helm upgrade --install \
longhorn longhorn \
--repo https://charts.longhorn.io \
--create-namespace \
--namespace longhorn-system
```
