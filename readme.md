# Kubernello

## prerequisites

- helm 3

### tools

- k3sup 0.9.12
- k9s
- kubectl v1.19.3

https://github.com/alexellis/k3sup

## price

why it this possible?

### do this need a cloud load balancer?

no, due to how k3s handle the loadbalancer, it does not require an external (paid) cloud load balancer

https://github.com/k3s-io/klipper-lb

%%%%%%%%%%%%%%% spiegare come funziona host-port del traefik ingress contrioller, per dirottare tutto traffico 443 e 80 a il pod di traefik %%%%%%%%%%%%%%%

## why

## Server

https://hetzner.cloud/

## Istalling k3s

- Install [k3sup](https://github.com/alexellis/k3sup#download-k3sup-tldr)

```
export IP=116.203.17.76
k3sup install \
  --ip $IP \
  --user root \
  --merge \
  --local-path $HOME/.kube/config \
  --context my-kubernello \
  --k3s-extra-args '--no-deploy traefik'
```

```
Running: k3sup install
2020/12/20 15:17:48 116.203.17.76
Public IP: 116.203.17.76
[INFO]  Finding release for channel v1.19
[INFO]  Using v1.19.5+k3s2 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.19.5+k3s2/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.19.5+k3s2/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

### what it does?

- ssh in the machine
- install k3s
- starts it
- apply all manifests in `/var/lib/rancher/k3s/server/manifests`
- add cluster to local kubeconfig

### why --no-deploy traefik?

because it this flag is present any change to the traefik config will be overridden on restart, we will install traefik ingress controller manually

password is not required because we added the SSH key to the server

### install ingress controller with custom settings

- ssh in to the machine `ssh root@116.203.17.76`

%%%%%%%%%%%%%%% METTERE GI DIFF del file con originale %%%%%%%%%%%
access log is the only new field, is used to have log of every request to the ingress, this is a traefik config file. %%%%%%%%%%%%%%% metter link a parametri possibili da doc traefik %%%%%%%%%%%%%%%

`kubectl apply -f ./traefik.yml`

navigate to http://116.203.17.76/
a 404 page will mean that the ingress controller is working properly!

## Deploy example app

`kubectl apply -f app.yml`

deploy an nginx and service

```
port forward command
%%%%%%%%%%%%%%% oneline command to port forward and curl to check that is working property %%%%%%%%%%%%%%%
```

## cert manager

https://cert-manager.io/docs/installation/kubernetes/#installing-with-helm

`kubectl create namespace cert-manager`

`helm repo add jetstack https://charts.jetstack.io`

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.1.0 \
  --set installCRDs=true
```

deploy `cert-manager`, it will watch create/updated/delete ingress resources that match specific labels in order to trigger the creation of a tls certificate via let's encrypt

## Setup DNS for custom domain

%%%%%%%%%%%%%%% insert screenshot %%%%%%%%%%%%%%%

record `A` `demo.livefun.dev` -> 116.203.17.76
record `CNAM` `*.demo.livefun.dev` -> demo.livefun.dev

## Expose app publicly via ingress

CHANGE URL IN `app-ingress.yml`

`kubectl apply -f app-ingress.yml`

```
kubernetes.io/ingress.class: 'traefik'
cert-manager.io/cluster-issuer: 'letsencrypt-prod'
traefik.ingress.kubernetes.io/redirect-entry-point: https
```

these 3 labels instruct the cluster to redirect http traffic to https...

## CI/CD

## how does persistence storage works?

## centralized logging (promtail loki grafana)

## metrics (prometheus grafana)

## gotchas
