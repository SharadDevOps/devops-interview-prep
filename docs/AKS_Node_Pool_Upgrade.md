# AKS Node Pool Upgrade — Complete Guide
> Lead DevOps Engineer Interview Topic | FinTrust Bank Case Study

---

## What is a Node Pool Upgrade?

AKS clusters run on nodes (VMs). These nodes have:
- A Kubernetes version (e.g., 1.27, 1.28)
- An OS image (Ubuntu, Windows)
- A VM size (Standard_D4s_v3)

Upgrading a node pool = updating the Kubernetes version on all nodes
without losing the workloads running on them.

---

## Why Zero Downtime is Critical (especially for FinTrust Bank)

FinTrust processes 4 million transactions daily.
Any downtime = financial loss + RBI compliance violation.

A node pool upgrade done WRONG = all pods evicted at once = outage.
A node pool upgrade done RIGHT = rolling upgrade, zero downtime.

---

## The Upgrade Process Step by Step

### Step 1 — Check Available Versions
```bash
az aks get-upgrades \
  --resource-group rg-shopflow-prod \
  --name aks-shopflow-prod \
  --output table
```

Output shows:
- Current version: 1.27.9
- Available upgrades: 1.28.5, 1.28.7

### Step 2 — Set Max Surge BEFORE Upgrading
```bash
az aks nodepool update \
  --resource-group rg-shopflow-prod \
  --cluster-name aks-shopflow-prod \
  --name workloadpool \
  --max-surge 33%
```

Max surge = 33% means:
- If you have 3 nodes → AKS adds 1 extra node (surge node) FIRST
- Now you have 4 nodes temporarily
- Old node gets cordoned and drained
- Workloads move to surge node
- Old node deleted
- Repeat for each node

WITHOUT max surge:
- Node gets cordoned → workloads have nowhere to go → OUTAGE

### Step 3 — Upgrade Control Plane First
```bash
az aks upgrade \
  --resource-group rg-shopflow-prod \
  --name aks-shopflow-prod \
  --kubernetes-version 1.28.5 \
  --control-plane-only
```

ALWAYS upgrade control plane before node pools.
Control plane manages the cluster brain — API server, scheduler, etcd.
Kubernetes supports control plane 1 version ahead of nodes.

### Step 4 — Validate Control Plane Health
```bash
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running
kubectl get componentstatuses
```

Ensure:
- Control plane nodes show new version
- No pods in CrashLoopBackOff or Pending
- All system pods healthy

### Step 5 — Upgrade System Node Pool
```bash
az aks nodepool upgrade \
  --resource-group rg-shopflow-prod \
  --cluster-name aks-shopflow-prod \
  --name systempool \
  --kubernetes-version 1.28.5 \
  --no-wait
```

Monitor progress:
```bash
watch kubectl get nodes
```

### Step 6 — Upgrade Workload Node Pool
```bash
az aks nodepool upgrade \
  --resource-group rg-shopflow-prod \
  --cluster-name aks-shopflow-prod \
  --name workloadpool \
  --kubernetes-version 1.28.5
```

### Step 7 — Post Upgrade Validation
```bash
# Check all nodes on new version
kubectl get nodes -o wide

# Check all pods running
kubectl get pods --all-namespaces

# Check HPA still working
kubectl get hpa --all-namespaces

# Check PodDisruptionBudgets respected
kubectl get pdb --all-namespaces

# Test application endpoints
curl https://internetbanking.fintrust.in/health
```

---

## What Happens During Node Drain?

When AKS upgrades a node:

```
1. CORDON the node
   kubectl cordon node-1
   → Node marked Unschedulable
   → No new pods scheduled here

2. DRAIN the node
   kubectl drain node-1 --ignore-daemonsets
   → All pods gracefully evicted
   → Pods rescheduled on other nodes
   → Respects PodDisruptionBudgets

3. DELETE old node VM
4. CREATE new node VM with new K8s version
5. JOIN new node to cluster
6. UNCORDON new node
   → New node accepts workloads
```

---

## PodDisruptionBudget — Critical for Banking

Without PDB — drain evicts ALL pods at once = outage.
With PDB — drain respects minimum available pods.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: internet-banking-pdb
spec:
  minAvailable: 2        # Always keep 2 pods running
  selector:
    matchLabels:
      app: internet-banking
```

With this PDB:
- If 3 pods running → drain evicts 1 at a time
- Always 2 pods serving traffic
- Zero downtime during upgrade

---

## Max Surge Options

| Setting | Behavior | Use Case |
|---|---|---|
| 0 (default) | No surge — upgrade in place | Dev/Test (risky for prod) |
| 1 | Add 1 extra node | Small clusters |
| 33% | Add 33% extra nodes | Production (recommended) |
| 100% | Double the nodes | Maximum speed, higher cost |

---

## Interview Question & Answer

Q: "How do you upgrade an AKS cluster with zero downtime?"

A: "I follow a specific sequence. First I check available versions with
az aks get-upgrades. Before touching anything I set max-surge to 33%
on each node pool — this ensures AKS adds a new surge node before
draining the old one, so workloads always have somewhere to go.

I upgrade the control plane first using --control-plane-only flag.
Kubernetes supports control plane being 1 version ahead of nodes.
I validate control plane health before touching any node pool.

Then I upgrade the system node pool, monitor with watch kubectl get nodes,
then upgrade the workload node pool.

Throughout the upgrade PodDisruptionBudgets ensure we never drop below
minimum replicas. For FinTrust Bank we set minAvailable: 2 on all
critical services — payment gateway, internet banking — so during drain
we always have 2 pods serving live traffic.

Post upgrade I validate all nodes, pods, HPA, and run smoke tests
against the application endpoints."

---

## What-If Scenarios

Q: What if a node gets stuck draining?
A: kubectl drain can hang if a pod has no PDB and wont terminate.
   Use: kubectl drain node --force --grace-period=0
   Investigate stuck pod: kubectl describe pod <pod>
   Check for finalizers, resource locks

Q: What if you need to rollback?
A: AKS does not support in-place downgrade.
   Prevention: Keep DR cluster on old version until upgrade validated.
   Recovery: Restore from AKS backup (Velero) or redeploy from Helm.

Q: What if upgrade fails midway?
A: Some nodes upgraded, some not — mixed version cluster.
   This is supported — Kubernetes allows n-1 node versions.
   Continue upgrade or use az aks nodepool upgrade --kubernetes-version
   to retry.

---

## Terraform — Node Pool Upgrade Settings

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  default_node_pool {
    # Enable upgrade settings
    upgrade_settings {
      max_surge = "33%"    # Key setting for zero downtime
    }

    # Enable autoscaler
    enable_auto_scaling = true
    min_count           = 2    # Always minimum 2 nodes
    max_count           = 10
  }
}
```

---

## GitHub: SharadDevOps/devops-interview-prep
