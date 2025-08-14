# Kubernetes PVC/PV + SMB CSI Troubleshooting Guide

## 📌 Quick Facts

- PVC fields like `volumeName`, `accessModes`, `storageClassName`, `volumeMode` are **immutable** after binding.
- Only `spec.resources.requests.storage` can be patched.
- **PVC delete + recreate** is the only way to modify immutable spec fields.
- `storageClassName` mismatch is the #1 reason PVCs remain in `Pending`.
- CSI volumeHandle **must be unique per PV**. Sharing leads to endless mount loops.
- `PodInitializing` usually means a mount, init container, or image pull is still in progress.

---

## 🛠️ Common Scenarios and Fixes

### ❌ Trying to Patch Immutable Fields

**Error**: Cannot change `volumeName` or `storageClassName` on bound PVC.

**Fix Options**:

| Keep Data | Steps |
|-----------|-------|
| No | 1. Stop workload.<br>2. `kubectl delete pvc <pvc>`<br>3. Reapply manifest. |
| Yes | 1. Stop workload.<br>2. `kubectl delete pvc <pvc> --cascade=orphan`<br>3. Edit PVC YAML: remove `status`, `resourceVersion`, `uid`, and `claimRef`. <br>4. `kubectl apply -f pvc-fixed.yaml` |

---

### 📦 PVC Pending (Unbound)

**Symptoms**: PVC stuck in `Pending`.

**Checklist**:
- PVC and PV must have:
  - Same `storageClassName` or both blank (static provisioning)
  - Compatible `accessModes`
  - Request size ≤ PV capacity

**Fix**:

```bash
# Match the PV's class to the PVC
kubectl patch pv <pv-name> --type=merge -p '{"spec":{"storageClassName":"vsphere-csi"}}'

# OR

# Clear PVC's class (if it’s still unbound)
kubectl patch pvc <pvc-name> -n <ns> --type=merge -p '{"spec":{"storageClassName":""}}'
```

---

### 🔁 PV in "Released" or "Terminating"

**Steps to rebind or clean:**

```bash
# 1. Remove claimRef
kubectl patch pv <pv-name> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'

# 2. Delete VolumeAttachment if stuck
kubectl get volumeattachment | grep <pv-name>
kubectl delete volumeattachment <name>

# 3. Remove finalizers (if Terminating)
kubectl patch pv <pv-name> --type=merge -p '{"metadata":{"finalizers":null}}'
```

---

### 🧱 Pod Hangs During Startup

**Root causes**:
- PVC(s) still `Pending`
- SMB mount failure (13, 32, 110 errors)
- Init container (e.g., `istio-init`) fails
- Image pull issues

**Troubleshooting**:

```bash
kubectl describe pod <pod> -n <ns>
kubectl logs -n <ns> <pod> -c <container>
kubectl logs -n csi-smb <driver-pod> -c smb --tail=100
```

**Common Error Codes**:

| Code | Meaning                    | Fix                                      |
|------|----------------------------|------------------------------------------|
| 13   | Permission denied          | Check secret + share ACL                 |
| 32   | Conflicting mount options  | Make PVs use same `mountOptions`         |
| 110  | Timeout                    | Check UNC path, DNS, firewall            |

---

### 🧩 Multiple SMB Mounts in a Pod

- Each PV must have a unique `.spec.csi.volumeHandle`.
- Sharing one handle across multiple shares causes conflict and infinite remount loops.

**Fix**:

```yaml
# Example static PV with unique handle
spec:
  csi:
    driver: smb.csi.k8s.io
    volumeHandle: xyz-smb  # must be unique
    volumeAttributes:
      source: \\host\Budget\
```

---

## 📋 Sanity Commands

```bash
# Check PVC/PV status
kubectl get pvc -n <ns>
kubectl get pv

# Find claimRef
kubectl get pv <pv> -o yaml | grep claimRef

# View volumeHandle across all PVs
kubectl get pv -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{"  "}{.spec.csi.volumeHandle}{end}'
```

---

## ✅ Best Practices

- Always define `persistentVolumeReclaimPolicy: Retain` for critical PVs.
- Scale down workloads before deleting PVCs.
- Avoid reusing `volumeHandle` for different SMB paths.
- Don’t rely on `kubectl apply` to update existing PVs—use `patch` or recreate instead.

### Takeaways

- Finalizer from the PVC (__kubernetes.io/pvc-protection__) and CSI controllers (providsioner/attacher/snapshotter) stick when cleanup can't finish (driver down, VolumeAttachement stuck, node gone, RBAC/API errors)
- PV __.spec.claimRef__: with __Retain__ reclaim policy, it is `expected` to remain; admin must clea up the backend volume and clear __claimRef__ to reuse. Wtih __Delete__, a broken CSI often leaves PV/PVC stuck

### Safe cleanup (automate with tool)

1. Guardrail
  * PVC has deletionTimestamp and finalizer for > N minutes
  * No Pods reference the PVC
  * No VolumeAttachment exists for the PV
2. If reclaimPolicy=Delete and backend gone: patch PVC to drop finalizers

   ```bash
   kubectl patch pvc <name> -n <ns> -p '{"metadata":{"finalizers":[]}}'
   ```
3. If reclaimPolicy=Retain and you cleaned storage: clear PV claimRef to reuse

   ```bash
   kubectl patch pv <pv-name> --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'
   ```
4. handy queries

```bash
   # Find stuck PVCs (pending deletion with finalizers)
   kubectl get pvc -A -o json | jq -r '
 .items[] | select(.metadata.deletionTimestamp and (.metadata.finalizers|length>0)) |
 "\(.metadata.namespace)/\(.metadata.name) finalizers=\(.metadata.finalizers)"'

   # Check VolumeAttachments for a PV
   kubectl get volumeattachment -A --field-selector spec.source.persistentVolumeName=<pv>
```
