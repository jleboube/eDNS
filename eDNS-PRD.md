# Product Requirements Document: Kubernetes-Based Encrypted DNS Server with Ad Blocking and Cluster Management

# Author: Joe LeBoube Copyright 2025

## 1. Document Overview
### 1.1 Purpose
This Product Requirements Document (PRD) outlines the requirements for a network-wide encrypted DNS server integrated with content filtering and ad-blocking capabilities, designed for deployment in a Kubernetes environment. The solution targets home lab users with Ubiquiti UniFi Network setups, managing IoT devices, mobile phones, and other networked devices. It aims to provide secure, private DNS resolution while blocking unwanted ads, trackers, and malicious content at the DNS level.

Inspired by tools like Pi-hole and AdGuard Home, this system will combine encrypted DNS protocols (e.g., DNS over HTTPS - DoH, or DNS over TLS - DoT) with blocklist-based filtering. The deployment will leverage Kubernetes for scalability, high availability, and ease of management in a home lab setting.

To address the complexity of Kubernetes management, this updated PRD includes a web-based Kubernetes management component, allowing users to administer the cluster via a secure web console without needing deep CLI expertise.

### 1.2 Scope
- **In Scope**: 
  - Encrypted DNS resolution with upstream secure resolvers.
  - Ad blocking and content filtering using predefined blocklists.
  - Kubernetes deployment manifests (e.g., Deployments, Services, ConfigMaps).
  - Basic web-based dashboard for DNS configuration and monitoring.
  - Web-based Kubernetes cluster management console (e.g., based on Kubernetes Dashboard).
  - Integration with Ubiquiti UniFi Network for DNS redirection.
- **Out of Scope**:
  - Advanced machine learning-based filtering.
  - Custom hardware integration beyond standard Kubernetes clusters.
  - Mobile app support for management.
  - Enterprise-scale features like multi-tenancy or federated authentication.

### 1.3 Assumptions and Dependencies
- The user has a running Kubernetes cluster (e.g., via k3s, MicroK8s, or full Kubernetes) in their home lab.
- Ubiquiti UniFi Network is configured to forward DNS queries to the Kubernetes service IP/port.
- Access to blocklists from sources like StevenBlack or AdGuard DNS filter lists.
- Basic knowledge of Kubernetes YAML manifests and Docker for containerization.
- Dependencies: Docker images for DNS components (e.g., based on Unbound, Pi-hole, or AdGuard Home), Kubernetes Dashboard image, persistent storage for configurations and logs.

### 1.4 Stakeholders
- **End User**: Home lab administrator (e.g., the host managing IoT and mobile devices).
- **Developer/Implementer**: The individual building and deploying the Kubernetes resources.
- **Devices**: IoT, mobiles, and other network clients relying on the DNS server.

## 2. Functional Requirements
### 2.1 Core DNS Functionality
- The system must act as a recursive DNS resolver, caching responses to improve performance.
- Support for encrypted upstream DNS queries:
  - DNS over TLS (DoT) to providers like Cloudflare (1.1.1.1:853) or Quad9.
  - DNS over HTTPS (DoH) to providers like Google (dns.google) or Cloudflare (cloudflare-dns.com).
- Configurable upstream resolvers with fallback options for redundancy.
- Handle standard DNS queries on UDP/TCP port 53, with optional DoH/DoT listener for clients that support it.

### 2.2 Ad Blocking and Content Filtering
- Integrate with established blocklists (e.g., hosts files from Pi-hole or AdGuard repositories).
- Block domains associated with ads, trackers, malware, and phishing.
- Allow custom allowlists and blocklists via configuration.
- Query logging with filtering statistics (e.g., number of blocked queries).
- Parental control filters (optional, toggleable) for blocking adult content or social media.

### 2.3 DNS Management Interface
- Web-based dashboard accessible via Kubernetes Ingress or port-forwarding.
- Features:
  - Real-time query logs and statistics.
  - Blocklist management (add/remove/update).
  - DNS cache flushing.
  - Configuration for upstream resolvers and encryption settings.
- Authentication: Basic HTTP auth or optional OAuth integration.

### 2.4 Kubernetes Cluster Management Component
- Deploy a web-based console for managing the Kubernetes cluster to simplify operations for home lab users.
- Based on the official Kubernetes Dashboard or a similar lightweight tool.
- Features:
  - View and manage resources (Pods, Deployments, Services, ConfigMaps, etc.).
  - Monitor cluster health, node status, and resource usage.
  - Deploy and scale applications via YAML or UI forms.
  - Log viewing and basic debugging tools.
  - Role-Based Access Control (RBAC) for secure access.
- Accessible via a separate Ingress path or port (e.g., https://<cluster-ip>/dashboard).
- Authentication: Token-based (using Kubernetes service accounts) or integrated with external auth providers if configured.
- Integration: Deployed as part of the overall Kubernetes manifests, with optional enable/disable flag.

### 2.5 Integration with Home Network
- Configurable to work with Ubiquiti UniFi Network:
  - Set the Kubernetes service IP as the DNS server in UniFi settings.
  - Support for DHCP options to push DNS settings to clients.
- Handle high query volumes from IoT and mobile devices without performance degradation.
- The cluster management console should be accessible from the local network, with optional exposure via UniFi port forwarding.

### 2.6 Monitoring and Logging
- Export metrics to Prometheus for Kubernetes monitoring (e.g., query rates, block rates, cluster metrics).
- Persistent logging to Kubernetes volumes or external systems like ELK stack.
- Alerts for high block rates, upstream failures, or cluster issues (e.g., pod crashes).

## 3. Non-Functional Requirements
### 3.1 Performance
- Handle at least 1,000 queries per second on modest hardware (e.g., Raspberry Pi cluster).
- Low latency: <50ms average response time for cached queries.
- Scalable: Horizontal scaling via Kubernetes replicas.
- Cluster management console should load quickly and handle typical home lab workloads without significant overhead.

### 3.2 Security
- All DNS traffic encrypted where possible (DoH/DoT).
- No exposure of plain-text DNS externally; use Kubernetes Network Policies to restrict access.
- Secure the cluster management console with HTTPS, RBAC, and network policies to prevent unauthorized access.
- Regular updates to blocklists, software vulnerabilities, and Kubernetes components.
- Secrets management: Use Kubernetes Secrets for API keys, auth credentials, and dashboard tokens.

### 3.3 Reliability
- High availability: Use Kubernetes Deployments with multiple replicas and anti-affinity for DNS components.
- Automatic restarts on failure.
- Data persistence: Use PersistentVolumes for configs, logs, blocklists, and dashboard data.
- The management console should not impact DNS availability; deploy it in a separate namespace if needed.

### 3.4 Usability
- Easy deployment: Single YAML file or Helm chart for installation, including the management console.
- Configuration via environment variables or ConfigMaps.
- Documentation: Inline comments in manifests, a setup guide, and instructions for accessing the cluster console.

### 3.5 Compatibility
- Kubernetes versions: 1.20+.
- Container runtime: Docker or containerd.
- Network: Compatible with UniFi's DNS forwarding; no conflicts with existing NAT or firewall rules.
- Management Console: Compatible with the target Kubernetes version; use official Dashboard manifests.

## 4. Technical Architecture
### 4.1 High-Level Design
- **Components**:
  - **DNS Resolver Pod**: Based on a containerized DNS server (e.g., Unbound or CoreDNS) with DoH/DoT plugins.
  - **Ad Blocker Integration**: Sidecar container or integrated script to apply blocklists (e.g., using Pi-hole's gravity or AdGuard's filtering engine).
  - **DNS Dashboard Pod**: Web server (e.g., based on AdGuard Home or custom Node.js/Flask app).
  - **Kubernetes Management Pod**: Kubernetes Dashboard deployment with metrics-scraper sidecar.
  - **Storage**: PVC for persistent data across components.
- **Kubernetes Resources**:
  - Deployment: For DNS, DNS dashboard, and Kubernetes Dashboard pods.
  - Service: ClusterIP or LoadBalancer exposing port 53 (UDP/TCP) for DNS, 80/443 for DNS dashboard, and a separate port (e.g., 8443) for cluster management.
  - ConfigMap: For blocklists, configs, and dashboard settings.
  - Ingress (optional): For external access to dashboards, with paths like /dns and /cluster.
  - Namespace: Optional separate namespace for management components.
  - RBAC: ClusterRole and ServiceAccount for the Kubernetes Dashboard.

### 4.2 Data Flow
1. Client (IoT/mobile) sends DNS query to UniFi router.
2. UniFi forwards to Kubernetes Service IP:53.
3. DNS pod receives query, checks blocklist.
4. If not blocked, resolves via encrypted upstream (DoH/DoT).
5. Caches and returns response.
6. Logs query to DNS dashboard for monitoring.
7. User accesses cluster management console to view/scale DNS pods or monitor overall cluster.

### 4.3 Technology Stack
- Base Images: Alpine Linux for minimal footprint.
- DNS Software: Unbound (for encryption) + Pi-hole/AdGuard scripts for filtering.
- DNS Web Dashboard: AdGuard Home (containerized) or Pi-hole web interface.
- Cluster Management: Official Kubernetes Dashboard image.
- Kubernetes Tools: kubectl apply for deployment; optional Helm for packaging.

## 5. Deployment and Operations
### 5.1 Deployment Steps
1. Clone repository or create YAML manifests.
2. Apply Kubernetes resources: `kubectl apply -f deployment.yaml` (includes DNS and management components).
3. Configure UniFi DNS settings to point to the Service IP.
4. Access DNS dashboard at http://<service-ip>:80.
5. Access cluster management console via `kubectl proxy` initially, or set up Ingress for https://<ingress>/dashboard.
6. Generate and use a token for dashboard login (instructions in setup guide).
7. Update blocklists via DNS dashboard or ConfigMap reload.

### 5.2 Testing
- Functional: Verify DNS resolution, encryption (using tools like dig with +tls), ad blocking (test known ad domains), and cluster management (create/delete test pods via console).
- Performance: Load test with tools like dnsperf; simulate cluster operations.
- Security: Scan for vulnerabilities with Trivy; test Network Policies and RBAC.

### 5.3 Maintenance
- Blocklist updates: CronJob in Kubernetes to fetch latest lists weekly.
- Upgrades: Rolling updates via Kubernetes for all components.
- Backup: Volume snapshots for PVCs.
- Monitor console usage and update Kubernetes Dashboard version as needed.

## 6. Risks and Mitigations
- **Risk**: Kubernetes complexity for home users. **Mitigation**: Integrate web console for UI-based management; provide simplified manifests and tutorials.
- **Risk**: Blocklist false positives. **Mitigation**: Easy allowlist configuration.
- **Risk**: Upstream DNS downtime. **Mitigation**: Multiple fallback resolvers.
- **Risk**: Resource consumption. **Mitigation**: Resource requests/limits in Deployments.
- **Risk**: Security exposure of management console. **Mitigation**: Enforce RBAC, HTTPS, and local network access only.

## 7. Appendix
### 7.1 References
- Pi-hole Documentation: For blocklist inspiration.
- AdGuard Home: For integrated filtering and dashboard.
- Unbound DNS: For encrypted resolution.
- Kubernetes Dashboard: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
- Kubernetes Docs: For deployment best practices.

This PRD serves as a blueprint for implementation. Iterations may be needed based on testing in the specific home lab environment.