# Workshop Admin Setup Guide

Configuration and preparation steps the cluster admin must complete before each workshop. This guide covers infrastructure prerequisites, RBAC provisioning, and per-user onboarding.

Created: 2026-07-10
Modified: 2026-07-13

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Infrastructure Checklist](#step-1-infrastructure-checklist)
- [Step 2: Create RBAC Roles (one-time)](#step-2-create-rbac-roles-one-time)
- [Step 3: Create User Groups](#step-3-create-user-groups)
- [Step 4: Onboard Users](#step-4-onboard-users)
- [Step 5: Verify Permissions](#step-5-verify-permissions)
- [Cleanup After Workshop](#cleanup-after-workshop)
- [RBAC Reference](#rbac-reference)
- [File Reference](#file-reference)

---

## Prerequisites

- Logged in as `cluster-admin`:

```bash
oc whoami
```

- RHOAI operator installed and `DataScienceCluster` is ready
- MaaS infrastructure deployed (`models-as-a-service` namespace exists):

```bash
oc get namespace models-as-a-service
```

- GPU worker nodes provisioned and labeled:

```bash
oc get nodes -l nvidia.com/gpu.present=true
```

- NVIDIA GPU Operator and Node Feature Discovery operators installed
- HTPasswd identity provider configured with workshop user accounts
- MaaS gateway is ready:

```bash
oc get gateway maas-default-gateway -n openshift-ingress
```

---

## Step 1: Infrastructure Checklist

Run this block to verify all infrastructure components are in place:

```bash
echo "=== RHOAI Operator ==="
oc get csv -n redhat-ods-operator | grep -i rhods

echo ""
echo "=== MaaS Namespace ==="
oc get namespace models-as-a-service

echo ""
echo "=== MaaS Gateway ==="
oc get gateway maas-default-gateway -n openshift-ingress

echo ""
echo "=== GPU Nodes ==="
oc get nodes -l nvidia.com/gpu.present=true --no-headers

echo ""
echo "=== Kuadrant (RHCL) ==="
oc get kuadrant -n redhat-ods-applications
```

All checks should return valid resources. If any fail, the MaaS setup script may need to be re-run (`./scripts/setup-maas.sh --with-observability`).

---

## Step 2: Create RBAC Roles (one-time)

Apply the shared RBAC definitions. This only needs to be done once per cluster, not per workshop session.

```bash
oc apply -f admin_manifests/00-admin-rbac.yaml
```

This creates three roles:

| Role | Kind | Purpose |
|------|------|---------|
| `workshop-namespace-labeler` | ClusterRole | Allows users to patch/label namespaces (for MaaS gateway access label) |
| `workshop-model-deployer` | ClusterRole | Allows CRUD on `LLMInferenceService` and `MaaSModelRef` |
| `workshop-subscription-creator` | Role (in `models-as-a-service`) | Allows CRUD on `MaaSSubscription` |

---

## Step 3: Create User Groups

Create an OpenShift group per user so that MaaS Subscriptions can be scoped to specific groups via the Dashboard UI.

### Single user

```bash
oc adm groups new <username> <username>
```

### Batch group creation

```bash
for user in usera userb userc; do
  oc adm groups new $user $user
  echo "--- group $user created ---"
done
```

Verify:

```bash
oc get groups
```

These groups will appear as selectable options when creating a MaaS Subscription in the RHOAI Dashboard (under "Owner > Groups").

---

## Step 4: Onboard Users

### Single user

Replace `<username>` with the actual OpenShift username (e.g., `usera`):

```bash
export WORKSHOP_USER=<username>
```

**Create the namespace:**

```bash
oc new-project workshop-$WORKSHOP_USER --display-name="Workshop - $WORKSHOP_USER"
```

**Apply the RBAC bindings:**

```bash
sed "s/REPLACE_USERNAME/$WORKSHOP_USER/g" admin_manifests/00-admin-rbac-binding-template.yaml | oc apply -f -
```

This creates three bindings for the user:
- `ClusterRoleBinding` for `workshop-namespace-labeler` (so the user can label their namespace)
- `RoleBinding` for `workshop-model-deployer` in `workshop-<username>` (so the user can deploy models)
- `RoleBinding` for `workshop-subscription-creator` in `models-as-a-service` (so the user can create subscriptions)

### Batch onboarding (multiple users)

Edit the user list and run:

```bash
for user in usera userb userc; do
  oc new-project workshop-$user --display-name="Workshop - $user" 2>/dev/null
  sed "s/REPLACE_USERNAME/$user/g" admin_manifests/00-admin-rbac-binding-template.yaml | oc apply -f -
  echo "--- $user onboarded ---"
done
```

---

## Step 5: Verify Permissions

For each onboarded user, verify they have the required permissions:

```bash
export WORKSHOP_USER=<username>

echo "=== Can label namespaces? ==="
oc auth can-i patch namespaces --as=$WORKSHOP_USER

echo "=== Can create LLMInferenceService? ==="
oc auth can-i create llminferenceservices.serving.kserve.io \
  -n workshop-$WORKSHOP_USER --as=$WORKSHOP_USER

echo "=== Can create MaaSModelRef? ==="
oc auth can-i create maasmodelrefs.maas.opendatahub.io \
  -n workshop-$WORKSHOP_USER --as=$WORKSHOP_USER

echo "=== Can create MaaSSubscription? ==="
oc auth can-i create maassubscriptions.maas.opendatahub.io \
  -n models-as-a-service --as=$WORKSHOP_USER
```

All four should return `yes`.

---

## Cleanup After Workshop

To remove all workshop resources after the session:

### Remove per-user resources

```bash
for user in usera userb userc; do
  oc delete project workshop-$user 2>/dev/null
  oc delete clusterrolebinding workshop-namespace-labeler-$user 2>/dev/null
  oc delete rolebinding workshop-subscription-$user -n models-as-a-service 2>/dev/null
  echo "--- $user cleaned up ---"
done
```

### Remove shared roles (optional, only if no future workshops)

```bash
oc delete clusterrole workshop-model-deployer workshop-namespace-labeler
oc delete role workshop-subscription-creator -n models-as-a-service
```

---

## RBAC Reference

### Roles created

| Role | Kind | Scope | API Groups | Resources | Verbs |
|------|------|-------|------------|-----------|-------|
| `workshop-namespace-labeler` | ClusterRole | Cluster | `""` | `namespaces` | get, patch, update |
| `workshop-model-deployer` | ClusterRole | Per-user namespace | `serving.kserve.io` | `llminferenceservices` | get, list, watch, create, update, patch, delete |
| | | | `maas.opendatahub.io` | `maasmodelrefs` | get, list, watch, create, update, patch, delete |
| | | | `serving.kserve.io` | `inferenceservices` | get, list, watch |
| `workshop-subscription-creator` | Role | `models-as-a-service` | `maas.opendatahub.io` | `maassubscriptions` | get, list, watch, create, update, patch, delete |

### Bindings per user

| Binding | Kind | Namespace | Role referenced |
|---------|------|-----------|-----------------|
| `workshop-namespace-labeler-<user>` | ClusterRoleBinding | (cluster) | `workshop-namespace-labeler` |
| `workshop-model-deployer-binding` | RoleBinding | `workshop-<user>` | `workshop-model-deployer` |
| `workshop-subscription-<user>` | RoleBinding | `models-as-a-service` | `workshop-subscription-creator` |

### Security note

The `workshop-namespace-labeler` ClusterRole grants get/patch/update on **all** namespaces cluster-wide, not just the user's own. Kubernetes RBAC cannot scope namespace access by name in a ClusterRoleBinding. This is acceptable for a workshop environment. For production, have the admin label namespaces instead.

---

## File Reference

| File | Description |
|------|-------------|
| `admin_manifests/00-admin-rbac.yaml` | ClusterRole and Role definitions (apply once per cluster) |
| `admin_manifests/00-admin-rbac-binding-template.yaml` | Per-user binding template (substitute `REPLACE_USERNAME`) |
| `README.md` | User-facing workshop guide |
