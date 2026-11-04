# Cloud, Container, and Infrastructure Security Requirements

These rules are mandatory when generating infrastructure-as-code, container configurations, CI/CD pipelines, or cloud resource definitions. They extend the root CLAUDE.md and work alongside Project CodeGuard's cloud security rules.

## Infrastructure as Code (Terraform, CloudFormation, Pulumi)

### IAM and Access Control
- Never generate wildcard IAM policies (Action: "*", Resource: "*").
- Use least-privilege. Scope permissions to specific resources and actions.
- Never attach AdministratorAccess or PowerUser managed policies to service roles.
- Require MFA for console access in IAM policies.
- Use service-linked roles and instance profiles instead of long-lived access keys.

### Storage and Data
- S3 buckets: Block public access by default. Enable versioning and server-side encryption (SSE-S3 or SSE-KMS).
- Database instances: Never set publicly_accessible = true. Enable encryption at rest. Require TLS for connections.
- EBS volumes: Always enable encryption. Never leave default unencrypted.

### Networking
- Security groups: Never allow 0.0.0.0/0 on SSH (22), RDP (3389), or database ports.
- Default VPC: Do not use. Create purpose-built VPCs with proper subnet segmentation.
- Enable VPC flow logs for audit and troubleshooting.

### State and Secrets
- Terraform state files must be stored in encrypted remote backends (S3, Azure Blob, GCS).
- Never store secrets in Terraform variables, tfvars files, or state. Use secrets manager references.
- Enable state file locking to prevent concurrent modifications. On S3, use native locking (`use_lockfile = true`, Terraform 1.10+); DynamoDB-based locking is deprecated.

## Container Security (Docker)

- Never use `FROM latest`. Pin base images by digest or specific version tag.
- Never run containers as root. Use `USER nonroot` or a specific UID.
- Use multi-stage builds to minimize attack surface.
- Do not copy secrets into images. Use runtime injection via secrets managers or mounted volumes.
- Do not install unnecessary packages, compilers, or debug tools in production images.
- Set `HEALTHCHECK` instructions for production containers.
- Scan images for vulnerabilities before pushing to registry.

```
# WRONG
FROM node:latest
COPY . /app
RUN npm install

# CORRECT
# Pin to a current LTS (Node 20 reached end of life April 2026); pin by digest in production
FROM node:24-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
USER node
# -slim images do not include curl; use the runtime itself for the health check
HEALTHCHECK CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
```

## Kubernetes Security

### Pod Security
- Never set `privileged: true` on containers.
- Always define a securityContext with `runAsNonRoot: true` and `readOnlyRootFilesystem: true`.
- Drop all capabilities, then add only what is needed: `capabilities: { drop: ["ALL"], add: ["NET_BIND_SERVICE"] }`.
- Set resource limits (CPU, memory) on every container to prevent unbounded consumption.

### Network
- Apply NetworkPolicies with default-deny ingress and egress. Explicitly allow only required traffic.
- Use TLS/mTLS for service-to-service communication.

### Secrets
- Kubernetes Secrets are base64-encoded, not encrypted. Enable encryption at rest for etcd.
- Prefer external secrets operators (AWS Secrets Manager CSI driver, HashiCorp Vault) over native K8s Secrets for sensitive data.
- Never mount ServiceAccount tokens unless required. Set `automountServiceAccountToken: false`.

### RBAC
- Never grant cluster-admin to applications or CI/CD service accounts.
- Use namespace-scoped Roles, not ClusterRoles, when possible.
- Audit RBAC bindings regularly. Remove unused permissions.

## CI/CD Pipeline Security

### GitHub Actions / Pipeline Hardening
- Pin action versions by SHA, not tag: `uses: actions/checkout@abcdef123` not `uses: actions/checkout@v4`.
- Set `permissions` to minimum needed at workflow and job level. Default to `contents: read`.
- Never pass secrets as command-line arguments. Use environment variables or the secrets context.
- Do not allow `pull_request_target` triggers with checkout of PR code (injection risk).

### Artifact Security
- Sign build artifacts and container images (Sigstore/cosign).
- Generate and attach SBOM to every release.
- Store artifacts in authenticated, access-controlled registries.

### Secrets in CI/CD
- Use the platform's secrets manager (GitHub Secrets, GitLab CI Variables, etc.).
- Never echo or print secrets in build logs.
- Rotate CI/CD credentials on a defined schedule.
- Restrict secret access to specific environments and branches.

## Serverless Security (Lambda, Cloud Functions, Azure Functions)

- Apply least-privilege IAM roles to each function. Never share a single overpermissioned role across functions.
- Set memory limits, timeout limits, and concurrency limits to prevent resource exhaustion and runaway costs.
- Retrieve secrets from the platform's secrets manager at runtime. Never embed secrets in environment variables configured in the console or deployment manifests.
- Attach functions to a VPC when they need to access private resources. Restrict outbound access with security groups.
- Enable function-level logging and tracing (CloudWatch, Cloud Logging, Application Insights).
- Pin runtime versions. Do not use `latest` or auto-updating runtimes in production.
- Validate all event payloads (API Gateway, S3, SQS, etc.) — treat as untrusted input.

## Web Application Firewall and DDoS Protection

- Require a WAF (AWS WAF, Cloudflare WAF, Azure Front Door, GCP Cloud Armor) for all public-facing web applications and APIs.
- Configure WAF rules to block OWASP Top 10 attack patterns (SQLi, XSS, path traversal, etc.).
- Enable DDoS protection (AWS Shield, Azure DDoS Protection, GCP Cloud Armor) for production workloads.
- Set up rate limiting at the edge (WAF/CDN layer) in addition to application-level rate limiting.

## Backup and Disaster Recovery

- Encrypt all backups at rest using KMS-managed keys.
- Store backups in a separate region or account from the primary workload.
- Test restore procedures on a regular schedule. Untested backups are not backups.
- Set retention policies aligned with compliance requirements.
- Enable cross-region replication for critical data stores.
- Use immutable backups (S3 Object Lock, Azure Immutable Blob Storage) to protect against ransomware.

## Multi-Account and Environment Isolation

- Use separate cloud accounts or projects per environment (dev, staging, production).
- Never allow dev/staging workloads to access production data stores or secrets.
- Use AWS Organizations, Azure Management Groups, or GCP Organization Policies for centralized governance.
- Apply Service Control Policies (SCPs) or Organization Policies to enforce guardrails across accounts.

## DNS Security

- Enable DNSSEC for public-facing domains.
- Use private DNS zones (Route 53 Private Hosted Zones, Azure Private DNS, GCP Cloud DNS Private Zones) for internal service discovery.
- Do not expose internal service hostnames in public DNS records.
- Use CAA records to restrict which CAs can issue certificates for your domains.
- **Subdomain takeover prevention**: Audit DNS records for dangling CNAMEs pointing to deprovisioned cloud resources (S3 buckets, Azure Blob, GitHub Pages, Heroku, etc.). Remove DNS records before deprovisioning the target resource, not after.
- Include DNS record cleanup in infrastructure teardown procedures and IaC destroy workflows.
- Periodically scan for unclaimed subdomains using tools like `subjack`, `nuclei`, or cloud-native inventory APIs.

## Egress Filtering

- Restrict outbound traffic from containers, VMs, and serverless functions to known-good destinations.
- Use VPC egress rules, security groups, or network policies to block unnecessary outbound access.
- Route outbound internet traffic through a NAT gateway or forward proxy for logging and inspection.
- Alert on unexpected outbound connections, especially to unusual ports or IP ranges.

## Secure Logging and Monitoring

For application-level audit logging requirements (which events to log, structured logging format, never logging secrets/PII), see root `CLAUDE.md` § Audit Logging. The rules below extend those for cloud and platform layers.

- Send logs to centralized, immutable storage (CloudWatch Logs, Datadog, Splunk, Azure Monitor, GCP Cloud Logging). Use S3 Object Lock or Azure Immutable Blob Storage for tamper-evident long-term retention.
- Sanitize log inputs at ingest (WAF, ALB access logs, API Gateway) to prevent log injection from reaching downstream parsers and SIEM rules.
- Configure log retention policies in the platform itself (CloudWatch Logs retention, Log Analytics workspace retention) — do not rely on application-side deletion alone.
- Enable VPC Flow Logs, CloudTrail / Azure Activity Log / GCP Audit Logs for control-plane events. These are distinct from application audit logs and are required for forensics.
- Forward platform logs to the SIEM via native integrations (Kinesis → Splunk HEC, EventHub → Sentinel, Pub/Sub → Chronicle) rather than agent-based shipping where possible.
