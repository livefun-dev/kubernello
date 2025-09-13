# Kubernello - Kubernetes for Side Projects at 5€/month

**Kubernello** is a low-cost, self-hosted personal cloud solution designed for side projects and learning. Built on top of k3s (a lightweight Kubernetes distribution), it provides a complete container orchestration platform running on a single Hetzner VPS for approximately 5€/month.

## Why Kubernello?

This project was created to:
- **Minimize hosting costs** for personal projects compared to expensive cloud services
- **Maximize learning opportunities** with real Kubernetes experience
- **Provide easy deployment** and management of multiple applications
- **Enable HTTPS by default** with automatic SSL certificate management
- **Offer GitOps workflows** for automated deployments

## Table of Contents

- [Architecture](#architecture)
- [Cost Analysis](#cost-analysis)
- [Prerequisites](#prerequisites)
- [Server Setup](#server)
- [Installing k3s](#installing-k3s)
- [Traefik Ingress Controller](#install-ingress-controller-with-custom-settings)
- [Example Application Deployment](#deploy-example-app)
- [SSL Certificates with cert-manager](#cert-manager)
- [DNS Configuration](#setup-dns-for-custom-domain)
- [Exposing Applications](#expose-app-publicly-via-ingress)
- [TODO: Future Enhancements](#todo)

## Architecture

Kubernello uses k3s, a lightweight Kubernetes distribution that:
- Uses SQLite instead of etcd (better performance on small servers)
- Includes Traefik ingress controller configured to bind to host ports 80/443
- Provides local-path storage provisioner for persistent volumes
- Removes cloud-provider dependencies and unnecessary components

### How Traffic Routing Works

Traefik ingress controller runs with `hostPort` configuration, meaning:
1. External traffic hits your server's public IP on ports 80/443
2. Traffic is directly routed to the Traefik pod
3. Traefik proxies requests to internal services based on ingress rules
4. **No external load balancer needed** - saving ~20€/month compared to cloud providers

## Cost Analysis

| Component | Cost (monthly) |
|-----------|----------------|
| Hetzner VPS (2 vCPU, 4GB RAM) | ~5€ |
| **vs Google Cloud equivalent** | ~25€+ |
| **vs AWS equivalent** | ~30€+ |

**Total savings: ~20-25€/month** compared to major cloud providers

## Prerequisites

### Required Tools
- **helm 3** - Kubernetes package manager
- **k3sup 0.9.12+** - k3s installation tool ([GitHub](https://github.com/alexellis/k3sup))
- **kubectl v1.19.3+** - Kubernetes CLI
- **k9s** (optional) - Terminal-based Kubernetes UI

### Server Requirements
- **VPS Provider**: Hetzner Cloud (recommended for cost-effectiveness)
- **Specifications**: 2 vCPU, 4GB RAM, 40GB SSD
- **OS**: Ubuntu 20.04 LTS
- **SSH Access**: Root access with SSH key authentication

## Server Setup

### 1. Create Hetzner Cloud Server
1. Go to [Hetzner Cloud](https://hetzner.cloud/)
2. Create a new project
3. Launch a server with:
   - **Image**: Ubuntu 20.04 LTS
   - **Type**: CX21 (2 vCPU, 4GB RAM) - ~5€/month
   - **Location**: Choose closest to your users
4. **Important**: Add your SSH public key during creation
5. Note the server's public IP address

### 2. Server Preparation
Once created, your server will be accessible via SSH:
```bash
ssh root@YOUR_SERVER_IP
```

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

### Install Ingress Controller with Custom Settings

Our custom Traefik setup includes:
- **Access logs** in JSON format for monitoring
- **Prometheus metrics** for observability
- **SSL/TLS termination** support
- **Proper RBAC** permissions

```bash
kubectl apply -f ./traefik.yml
```

### Key Configuration Changes

Compared to default Traefik, our config adds:

```yaml
accessLogs:
  enabled: true
  format: "json"        # Structured logging
  fields:
    defaultMode: keep
metrics:
  prometheus:
    enabled: true         # Enable metrics collection
```

### Verify Traefik is Running

```bash
# Check Traefik pod status
kubectl get pods -n kube-system -l app=traefik

# Test ingress controller
curl -I http://YOUR_SERVER_IP/
```

**Expected result**: A `404 Not Found` response means Traefik is working correctly! 

> The 404 is normal - there are no ingress rules yet, but Traefik is receiving and handling requests.

## Deploy Example App

### 1. Deploy Nginx Application

Deploy a simple nginx app to test our setup:

```bash
kubectl apply -f app.yml
```

This creates:
- **Deployment**: nginx container with resource limits
- **Service**: ClusterIP service to expose the pod internally

### 2. Verify Deployment

```bash
# Check pod status
kubectl get pods

# Check service
kubectl get svc

# Port forward to test locally
kubectl port-forward svc/nginx 8080:80

# Test in another terminal
curl http://localhost:8080
```

**Expected**: You should see the nginx welcome page HTML.

## SSL Certificates with cert-manager

[cert-manager](https://cert-manager.io/) automatically provisions and manages TLS certificates from Let's Encrypt.

### 1. Install cert-manager

```bash
# Create namespace
kubectl create namespace cert-manager

# Add Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.1.0 \
  --set installCRDs=true
```

### 2. Create Let's Encrypt Issuer

**Important**: Update the email in `cluster-issuer.yml` before applying:

```bash
# Edit cluster-issuer.yml and replace email@email.com with your email
kubectl apply -f cluster-issuer.yml
```

### 3. How cert-manager Works

cert-manager watches for:
- **Ingress resources** with cert-manager annotations
- **Certificate requests** for domains specified in ingress TLS sections
- **Automatic renewal** of certificates before expiration

When an ingress with proper annotations is created, cert-manager:
1. Creates a **Certificate** resource
2. Initiates **ACME challenge** with Let's Encrypt
3. **Validates domain ownership** via HTTP-01 challenge
4. **Stores certificate** in Kubernetes Secret
5. **Automatically renews** certificates when needed

## Setup DNS for Custom Domain

To use custom domains with your Kubernello cluster, configure DNS records with your domain provider.

### Required DNS Records

| Type | Name | Value | Purpose |
|------|------|-------|----------|
| `A` | `yourdomain.com` | `YOUR_SERVER_IP` | Main domain |
| `CNAME` | `*.yourdomain.com` | `yourdomain.com` | Wildcard for subdomains |

### Example Configuration

For domain `demo.livefun.dev` with server IP `116.203.17.76`:

```
A     demo.livefun.dev      -> 116.203.17.76
CNAME *.demo.livefun.dev    -> demo.livefun.dev
```

### Verification

Test DNS propagation:

```bash
# Test main domain
dig yourdomain.com

# Test wildcard subdomain
dig app.yourdomain.com
```

Both should resolve to your server's IP address.

## Expose App Publicly via Ingress

### 1. Configure Domain in Ingress

**Important**: Update the domain in `app-ingress.yml`:

```yaml
spec:
  rules:
    - host: nginx.yourdomain.com  # Change this to your domain
  tls:
    - hosts:
        - nginx.yourdomain.com    # Change this too
      secretName: nginx.yourdomain.com-tls
```

### 2. Deploy Ingress Resource

```bash
kubectl apply -f app-ingress.yml
```

### 3. Critical Annotations Explained

These annotations are **required** for proper functionality:

```yaml
annotations:
  # Tells Traefik to handle this ingress
  kubernetes.io/ingress.class: 'traefik'
  
  # Triggers cert-manager to create SSL certificate
  cert-manager.io/cluster-issuer: 'letsencrypt-prod'
  
  # Redirects HTTP traffic to HTTPS automatically
  traefik.ingress.kubernetes.io/redirect-entry-point: https
```

### 4. Monitor Certificate Creation

```bash
# Watch certificate creation
kubectl get certificates -w

# Check certificate details
kubectl describe certificate nginx.yourdomain.com-tls

# View certificate logs
kubectl logs -n cert-manager -l app=cert-manager -f
```

### 5. Test Your Application

Once DNS propagates and certificates are issued:

```bash
# Test HTTP (should redirect to HTTPS)
curl -I http://nginx.yourdomain.com

# Test HTTPS
curl https://nginx.yourdomain.com
```

**Expected result**: Your nginx application accessible over HTTPS with a valid Let's Encrypt certificate!

## TODO

Future enhancements planned for Kubernello:

### DevOps & Automation
- [ ] **CI/CD Pipeline Setup**
  - GitLab CI/CD integration
  - GitHub Actions workflows
  - Automated deployment strategies

### Storage & Data
- [ ] **Persistent Storage Management**
  - Local-path provisioner configuration
  - Volume backup strategies
  - Database persistent volumes

### Monitoring & Observability
- [ ] **Centralized Logging**
  - Promtail + Loki + Grafana stack
  - Log aggregation and search
  - Application log collection

- [ ] **Metrics & Monitoring**
  - Prometheus metrics collection
  - Grafana dashboards
  - Alerting rules and notifications

- [ ] **Distributed Tracing**
  - OpenTelemetry integration
  - Jaeger for request tracing
  - Service dependency mapping

### Security & Production
- [ ] **Security Hardening**
  - Network policies
  - Pod security standards
  - Secret management best practices

- [ ] **Backup & Recovery**
  - Automated cluster backups
  - Disaster recovery procedures
  - Data retention policies

### Advanced Features
- [ ] **Multi-Application Management**
  - Helm chart deployments
  - Application versioning
  - Blue-green deployments

- [ ] **Resource Optimization**
  - Resource quotas and limits
  - Horizontal Pod Autoscaling
  - Cluster resource monitoring

## Gotchas & Common Issues

- **Certificate delays**: Let's Encrypt certificates can take 2-5 minutes to issue
- **DNS propagation**: Allow up to 24 hours for DNS changes to propagate globally
- **Resource limits**: Monitor CPU/memory usage on the 5€ server
- **Traefik annotations**: Typos in ingress annotations will silently fail
- **Context switching**: Always verify you're using the correct kubectl context

---

## Contributing

Contributions are welcome! Please feel free to:
- Report issues and bugs
- Suggest new features
- Submit pull requests
- Improve documentation

## License

This project is open source and available under the [MIT License](LICENSE).

## Acknowledgments

- **k3s** - Lightweight Kubernetes distribution
- **Traefik** - Modern reverse proxy and load balancer
- **cert-manager** - Automatic SSL certificate management
- **Hetzner** - Affordable and reliable cloud infrastructure
