# eDNS: Encrypted DNS with Ad Blocking on Kubernetes

Network-wide encrypted DNS resolver with ad/malware filtering and a bundled Kubernetes Dashboard for home-lab clusters (Ubiquiti/UniFi friendly). AdGuard Home handles DNS/DoH/DoT/DoQ with curated blocklists; Kubernetes Dashboard provides UI-based cluster management.

## What’s Included
- AdGuard Home deployment with TLS-enabled DNS (DoH/DoT/DoQ) and UI on port `8445`.
- Blocklists from StevenBlack, AdGuard, AdAway, and KAD (auto-refresh every 24h).
- Persistent volumes for config/work data, optional Ingress for UI/DoH, and a LoadBalancer service for DNS.
- Kubernetes Dashboard (separate namespace) with metrics-scraper and token-based access.
- Docker Compose v2 file for local/dev runs using non-default host ports.

## Kubernetes Deployment
1. Generate/provide TLS material for DoH/DoT and dashboards:
   ```bash
   kubectl create namespace edns-system
   kubectl -n edns-system create secret tls edns-tls --cert=certs/tls.crt --key=certs/tls.key
   ```
   Reuse the same `edns-tls` secret in `edns-mgmt` if you want TLS on the dashboard ingress:
   ```bash
   kubectl create namespace edns-mgmt
   kubectl -n edns-mgmt create secret tls edns-tls --cert=certs/tls.crt --key=certs/tls.key
   ```
2. Apply everything with Kustomize:
   ```bash
   kubectl apply -k k8s
   ```
3. Set UniFi/edge DNS forwarders to the `adguard-dns` LoadBalancer IP on port 53 (UDP/TCP). For DoH clients use `https://doh.edns.local/dns-query`; for DoT use `dot` on port `853`.

### Access
- **AdGuard UI**: `https://edns.local/` (Ingress) or `http://<service-ip>:8445` (ClusterIP). Default admin user `admin` / password `changeMe!` — update immediately:
  ```bash
  htpasswd -bnBC 10 admin '<new-password>'
  ```
  Replace the bcrypt hash in `config/adguard/AdGuardHome.yaml`, then re-apply.
- **Kubernetes Dashboard**: `https://dashboard.edns.local/` via Ingress, or port-forward the service:
  ```bash
  kubectl -n edns-mgmt port-forward service/kubernetes-dashboard 9443:8443
  ```
  Get a login token:
  ```bash
  kubectl -n edns-mgmt describe secret dashboard-token
  ```

### Config/Storage
- ConfigMap is generated from `config/adguard/AdGuardHome.yaml`; PVCs `adguard-config` and `adguard-work` keep state/logs.
- Ingress hosts (`edns.local`, `doh.edns.local`, `dashboard.edns.local`) are placeholders—update them to match your DNS/TLS certs.
- The AdGuard deployment runs 2 replicas; health checks hit the UI root.

## Docker Compose (local/dev)
Spin up AdGuard locally with non-default host ports:
```bash
docker compose up -d
```
Ports: DNS on `5533` (TCP/UDP), DoH `44443`, DoT `4853`, DoQ `4784`, UI `8445`. TLS cert/key should live under `./certs` and match `config/adguard/AdGuardHome.yaml`.

## Operations
- Blocklists auto-refresh every 24 hours; edit `filters`/`filters_update_interval` in `config/adguard/AdGuardHome.yaml` to customize.
- To rotate TLS certs, update the `edns-tls` secrets and restart deployments: `kubectl rollout restart deploy/adguard -n edns-system`.
- For RBAC hardening, swap the `cluster-admin` binding in `k8s/edns-mgmt/dashboard.yaml` with a scoped `ClusterRole` as needed.

## Verification
- DNS: `dig @<LB-IP> example.com` and ensure responses resolve; use `dig @<LB-IP> cloudflare.com +tls` for DoT.
- Ad blocking: `dig @<LB-IP> doubleclick.net` should return a blocked/0.0.0.0 response.
- Dashboard: confirm login via token and view resources/pods.
