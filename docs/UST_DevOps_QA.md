# UST Global — Lead DevOps Engineer Interview Q&A
> Role: Lead DevOps-1 | 5-7 Years | Mumbai/Chennai/Gurgaon
> Mandatory: Linux OS, Terraform, Azure Cloud, AKS, GitHub Actions

---

## SECTION 1 — Azure Cloud 🔴 Mandatory

### Q1. Explain the difference between Azure VNet Peering, VPN Gateway, and ExpressRoute. When would you choose each?

**Hint:** Think about latency, cost, security, and whether traffic goes over the public internet.

**Model Answer:**
- **VNet Peering:** Low-latency, private connection between two VNets. Traffic stays on Azure backbone. No bandwidth limits. Best for inter-VNet communication within Azure.
- **VPN Gateway:** Encrypted tunnel over the public internet between on-premises and Azure. Site-to-Site or Point-to-Site. Cost-effective for smaller bandwidth needs.
- **ExpressRoute:** Dedicated private circuit NOT over the internet via a connectivity provider. Highest reliability, lowest latency. Best for enterprise/compliance/high-bandwidth needs. More expensive.
- **Decision rule:** Internal Azure → VNet Peering | On-prem budget → VPN Gateway | Enterprise/compliance → ExpressRoute

---

### Q2. How do you implement RBAC in Azure and what is the difference between Azure RBAC and Azure AD roles?

**Model Answer:**
- **Azure RBAC** controls access to Azure resources (VMs, AKS, Storage) at subscription/resource group/resource scope. Built-in roles: Owner, Contributor, Reader. Custom roles for least-privilege.
- **Azure AD roles** control who manages Azure AD itself (Global Administrator, User Administrator) — NOT Azure resources. Different namespaces.
- **AKS best practice:** Assign AKS RBAC Writer to managed identities. Avoid Contributor at subscription scope. Use conditional access for human identities.
- Always follow least-privilege — grant minimum role at most specific scope.

---

## SECTION 2 — AKS (Kubernetes Cluster Management) 🔴 Mandatory

### Q3. Walk me through how you upgrade an AKS cluster version with zero downtime.

**Model Answer:**
1. Check available versions: `az aks get-upgrades --resource-group <rg> --name <cluster>`
2. Set max-surge on node pool (33%) — adds extra nodes before removing old ones
3. Upgrade control plane first: `az aks upgrade ... --control-plane-only`
4. Validate control plane health
5. Upgrade each node pool: `az aks nodepool upgrade ...` — AKS cordons/drains automatically
6. Monitor: `kubectl get nodes` — verify PodDisruptionBudgets respected
7. Post-upgrade: verify workloads running, check deprecated API versions

---

### Q4. How do you configure autoscaling in AKS? Explain HPA and Cluster Autoscaler.

**Model Answer:**
- **HPA:** Scales pod replicas based on CPU/memory (or custom metrics via KEDA). Requires resource requests on pods. Triggers when avg CPU exceeds target %.
- **Cluster Autoscaler:** Scales nodes when pod is in Pending state (no capacity). Scales down underutilized nodes.
- **Together:** HPA adds pods → if no node has capacity → Cluster Autoscaler adds nodes.
- **Critical:** Always set CPU/memory requests on ALL pods — CA won't work without them.
- **AKS config:** `--enable-cluster-autoscaler --min-count 2 --max-count 5`

---

### Q5. Your AKS pod is in CrashLoopBackOff. How do you troubleshoot end-to-end?

**Model Answer:**
1. `kubectl describe pod <pod-name>` — check Events section for exact error
2. `kubectl logs <pod-name> --previous` — view logs from last crash
3. Check for OOMKilled — if yes, increase memory limits
4. Verify image tag exists in ACR
5. Check liveness/readiness probe configuration — misconfigured probes kill pods prematurely
6. Check ConfigMaps/Secrets exist: `kubectl get secret / configmap`
7. `kubectl top nodes` — verify node has enough resources

---

## SECTION 3 — Terraform (Modules from Scratch) 🔴 Mandatory

### Q6. How do you structure and write a reusable Terraform module for AKS from scratch?

**Model Answer:**
```
modules/aks/
├── main.tf        # azurerm_kubernetes_cluster resource
├── variables.tf   # Input variables with validation blocks
├── outputs.tf     # kube_config, cluster_id, kubelet_identity
└── versions.tf    # Required providers with version constraints
```
- Use `validation{}` blocks to enforce allowed values (e.g., approved VM sizes)
- Use `locals{}` for computed/derived values — keeps modules DRY
- Expose meaningful outputs for root module chaining
- Version-pin the azurerm provider
- Pass environment-specific values via `.tfvars` files (dev.tfvars, prod.tfvars)
- Never hardcode values — everything configurable must be a variable

---

### Q7. What happens when terraform apply runs against a manually changed resource? How do you handle drift?

**Model Answer:**
- `terraform plan` detects drift — shows diff between desired and actual state
- **Option 1:** Keep manual change → update Terraform code to match → apply shows no diff
- **Option 2:** Revert manual change → run `terraform apply` — restores defined config
- **Option 3:** Import manually created resource: `terraform import <address> <azure_resource_id>`
- **Option 4:** Update state only: `terraform apply -refresh-only`
- **Prevention:** Azure Policy blocks direct portal/CLI changes. Only CI/CD managed identity has Contributor. Humans get Reader only.
- Use remote state with locking (Azure Storage + state lock) to prevent concurrent applies.

---

## SECTION 4 — GitHub Actions 🔴 Mandatory

### Q8. Design a GitHub Actions workflow: build Docker image → push to ACR → deploy to AKS.

**Model Answer:**
1. **Auth:** Use OIDC with `azure/login` + Federated Identity Credential — NO static secrets in GitHub
2. **Build:** `docker build -t <acr>.azurecr.io/<image>:${{ github.sha }} .`
3. **Push to ACR:** `az acr login` + `docker push` — image tagged with commit SHA (never :latest)
4. **Get AKS credentials:** `azure/aks-set-context` action
5. **Deploy:** `kubectl set image` or `helm upgrade --install`
6. **Environment protection:** Add `environment: production` with required reviewers
7. **Reusable workflows:** Extract build and deploy as reusable .github/workflows
8. **Security:** Pin all action versions to commit SHA

---

### Q9. How do you manage secrets across dev, staging, production in GitHub Actions?

**Model Answer:**
- **GitHub Environments:** Create dev/staging/production with own secrets, protection rules, branch restrictions
- **Secret hierarchy:** Repository secrets (shared) → Environment secrets (override per env)
- **Best practice — Azure Key Vault:** Use `azure/get-keyvault-secrets` action — secrets never stored in GitHub
- **Combined with OIDC:** Fully secretless pipeline — most secure approach
- **Secret rotation:** Key Vault handles rotation — workflows always get latest version
- **Non-sensitive config:** Use GitHub Actions Variables (not secrets) for cluster names, namespaces

---

## SECTION 5 — Linux OS 🔴 Mandatory

### Q10. A Linux server is consuming 100% CPU. How do you diagnose and resolve?

**Model Answer:**
1. `top` or `htop` — identify PID of offending process
2. `ps aux --sort=-%cpu | head -10` — confirm with full command details
3. `cat /proc/<PID>/status` + `ls -l /proc/<PID>/exe` — identify the process
4. `strace -p <PID>` — see system calls (detect infinite loops)
5. `lsof -p <PID>` — check open files/sockets
6. `journalctl -u <service> -n 100` — check application logs
7. **Resolution:** `kill -15 <PID>` (graceful) or `kill -9` (force) → `systemctl restart <service>`
8. **Prevention:** CPU limits via cgroups, Prometheus alert at 80%

---

### Q11. A Linux server is running out of disk space. How do you diagnose and fix?

**Model Answer:**
1. `df -h` — find which mount is full
2. `du -sh /*` — find largest top-level directories
3. `du -sh /var/log/*` — logs are usually the culprit
4. `lsof | grep deleted` — find deleted files still held open (won't free space until process closes)
5. `docker system prune -f` — remove dangling images/containers
6. Configure `logrotate` in `/etc/logrotate.d/` for automatic log rotation
7. **Prevention:** Prometheus disk usage alerts at 75% and 85%

---

## SECTION 6 — Monitoring 🟢 Good to Have

### Q12. How do you set up observability for AKS using Prometheus and Grafana?

**Model Answer:**
- Install `kube-prometheus-stack` via Helm — includes Prometheus, Alertmanager, Grafana
- ServiceMonitor/PodMonitor CRDs for auto-discovery of scrape targets
- Key metrics: node CPU/memory, pod restarts, PVC usage, API server latency
- Import Grafana dashboard ID 315 (Kubernetes cluster monitoring)
- PrometheusRule CRDs for alerts — route to Slack/PagerDuty via Alertmanager
- Also enable Azure Monitor Container Insights for platform-level metrics

---

### Q13. How do you debug production issues using ELK stack in Kibana?

**Model Answer:**
- Filebeat/Fluentd collects pod logs → Elasticsearch → Kibana
- In Kibana: Discover → select index pattern (e.g., `k8s-logs-*`)
- KQL: `kubernetes.namespace: production AND log.level: error`
- Use time filter to narrow to incident window
- Correlate across pods using request/trace ID
- **Pro tip:** Always use structured (JSON) logging — makes field filtering far more powerful

---

## SECTION 7 — Leadership & Scenario

### Q14. How do you enforce infra standards and prevent manual changes to Azure?

**Model Answer:**
- **RBAC:** Developers get Reader on production. Only CI/CD managed identity has Contributor
- **Azure Policy:** Enforce mandatory tags, approved VM SKUs, no public IPs — block non-compliant at creation
- **Terraform GitOps:** All changes via Git PR → terraform plan in CI → apply on merge
- **Two approval gates:** Team lead + manager for production deployments
- **Kubernetes GitOps:** Flux/ArgoCD — manual kubectl changes auto-reverted
- **Audit:** Azure Activity Log + Microsoft Defender for Cloud

---

### Q15. Production is down at 2am. Walk through your incident response.

**Model Answer:**
1. Acknowledge alert (PagerDuty/New Relic) — open incident Slack channel
2. Assess scope: pod issue, service issue, or full outage?
3. Check recent changes — last 60 minutes in GitHub Actions history
4. If bad deployment → rollback immediately, investigate after
5. Communicate to stakeholders every 15-30 minutes
6. Loop in DB, backend, network teams via distribution list + Teams bridge
7. **Blameless postmortem within 48 hours** — timeline, root cause, action items with owners

---

## SECTION 8 — Branching Strategy

### Q16. What branching strategy do you follow and how does it integrate with CI/CD?

**Model Answer:**
- **GitFlow:** main → develop → feature/* → release/* → hotfix/*. Good for scheduled releases.
- **Trunk-Based:** All devs commit to main via short-lived feature branches (<2 days). Best for microservices.
- **Our approach:** feature/* → dev (auto-deploy) → staging (auto-deploy) → main (prod with approval)
- **Branch protection:** Require 2 PR reviews, status checks must pass, no direct pushes
- **CI/CD mapping:** feature/* = build+test only | dev = deploy to dev | main = full pipeline with approval gate
- **Conventional commits:** feat:, fix:, chore: for automated changelog + semantic versioning
- **Rule:** Never commit directly to main — all changes via Pull Requests

---

## SECTION 9 — Securing Terraform State File

### Q17. How do you secure the Terraform state file in Azure?

**Model Answer:**
- **Why it matters:** tfstate contains sensitive data in plain text — never commit to Git
- **Remote backend:** Azure Blob Storage — configure `backend "azurerm"` with storage account
- **Encryption at rest:** AES-256 by default. Use Customer-Managed Keys (CMK) via Key Vault for extra security
- **Encryption in transit:** Azure Storage enforces HTTPS
- **State locking:** Blob lease mechanism — prevents concurrent applies
- **RBAC:** Only CI/CD managed identity gets Storage Blob Data Contributor. Devs get Reader.
- **Separate state per env:** Different containers for dev/staging/prod — never share state
- **Private endpoint:** Storage behind Private Endpoint — not accessible from public internet
- **Blob versioning + soft delete:** Rollback if state gets corrupted
- **Audit:** Storage diagnostic logs → Log Analytics
- **Sensitive outputs:** Use `sensitive = true` to prevent secrets in CI/CD logs
- **Rule:** Never store secrets as Terraform resources — use Key Vault data sources instead

---

## SECTION 10 — GitHub Actions Caching

### Q18. How do you handle caching in GitHub Actions? (Maven + Docker)

**Model Answer:**
```yaml
# Maven cache
- uses: actions/cache@v3
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      ${{ runner.os }}-maven-

# Docker layer cache
- uses: docker/build-push-action@v4
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```
- **key:** OS + hash of dependency file — changes when pom.xml changes → new cache
- **restore-keys:** Fallback partial match if exact key not found
- **Cache expires:** 7 days of non-use
- **Impact:** Reduced pipeline time from 12 min → 3 min

---

## SECTION 11 — Reusable Workflows

### Q19. What are reusable workflows in GitHub Actions and how do you implement them?

**Model Answer:**
- Reusable workflows allow multiple repos/jobs to call a shared workflow — DRY principle for CI/CD
- Define with `on: workflow_call:` trigger and declare `inputs` and `secrets`
- Called using `uses: ./.github/workflows/reusable-build.yml` or from another repo

```yaml
# reusable-build.yml
on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
    secrets:
      AZURE_CLIENT_ID:
        required: true

# Caller workflow
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      service-name: order-service
    secrets: inherit
```
- **Benefits:** Single place to update pipeline logic, consistent standards across all services
- **Our project:** One reusable-build.yml + reusable-deploy.yml called by all 3 microservices
