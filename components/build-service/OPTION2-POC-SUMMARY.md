# POC: Kustomize Components for External-Secrets (Option 2)

## Executive Summary

This POC demonstrates using **Kustomize Components** to isolate staging and production external-secrets. Components provide explicit environment selection while keeping shared application resources in a single base.

## What Changed

### Directory Structure

**New directories created:**
- `components/staging-secrets/` - Kustomize Component with staging external-secrets
- `components/production-secrets/` - Kustomize Component with production external-secrets

**Files modified:**
- `staging/base/kustomization.yaml` - Added `components/staging-secrets`
- `production/base/kustomization.yaml` - Added `components/production-secrets`, removed patch

**Key difference from Option 1:**
- Base resources (deployments, services, RBAC) stay in `base/`
- External-secrets are in separate components (not in base/)

## How It Works

### Kustomize Components Explained

A **Component** is a special type of kustomization that:
- Uses `kind: Component` instead of `kind: Kustomization`
- Cannot be built standalone (must be included by a kustomization)
- Provides modular, reusable pieces of configuration
- Perfect for environment-specific variants

### Structure

```
components/build-service/
├── base/                           ← Shared app resources (NO external-secrets)
│   ├── allow-argocd-to-manage.yaml
│   ├── build-pipeline-config/
│   ├── monitoring.yaml
│   └── rbac/
├── components/
│   ├── rh-certs/                   ← Existing component
│   ├── staging-secrets/            ← NEW: Staging external-secrets component
│   │   ├── kustomization.yaml      (kind: Component)
│   │   └── external-secrets/
│   │       └── pipelines-as-code-secret.yaml (key: staging/...)
│   └── production-secrets/         ← NEW: Production external-secrets component
│       ├── kustomization.yaml      (kind: Component)
│       └── external-secrets/
│           └── pipelines-as-code-secret.yaml (key: production/...)
├── staging/base/
│   └── kustomization.yaml
│       resources: [../../base, ...]
│       components: [../../components/staging-secrets]
└── production/base/
    └── kustomization.yaml
        resources: [../../base, ...]
        components: [../../components/production-secrets]
```

### Kustomization Files

**Staging (`staging/base/kustomization.yaml`):**
```yaml
resources:
- ../../base
components:
  - ../../components/rh-certs
  - ../../components/staging-secrets  ← Adds staging external-secrets
```

**Production (`production/base/kustomization.yaml`):**
```yaml
resources:
- ../../base
components:
  - ../../components/rh-certs
  - ../../components/production-secrets  ← Adds production external-secrets
```

**Staging Secrets Component (`components/staging-secrets/kustomization.yaml`):**
```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component  ← Not a Kustomization!
resources:
- external-secrets/pipelines-as-code-secret.yaml
```

## Proof It Works

### Test: Staging Build
```bash
$ kustomize build components/build-service/staging/base | grep -A6 "dataFrom:"
  dataFrom:
  - extract:
      key: staging/pipeline-service/github-app  ← Correct!
```

### Test: Production Build
```bash
$ kustomize build components/build-service/production/base | grep -A6 "dataFrom:"
  dataFrom:
  - extract:
      key: production/pipeline-service/github-app  ← Correct!
```

## Benefits

### ✅ Isolation
- Staging and production external-secrets are completely separate
- Changing staging-secrets component cannot affect production
- Each environment explicitly selects its component

### ✅ Explicit Environment Selection
```yaml
# Staging explicitly chooses staging-secrets:
components: [../../components/staging-secrets]

# Production explicitly chooses production-secrets:
components: [../../components/production-secrets]
```

### ✅ Shared Base for App Resources
- Deployments, services, RBAC stay in single `base/`
- Only external-secrets (environment-specific) are separated
- DRY principle for application code

### ✅ Kustomize-Native Pattern
- Uses official kustomize feature (Components)
- Well-documented in kustomize docs
- Same pattern already used for `rh-certs` component

### ✅ No Patches Needed
- Components add resources directly
- No fragile patch targeting
- No version mismatch possible

## Trade-offs

### Pros
- ✅ **Clean separation** - External-secrets isolated per environment
- ✅ **Explicit** - Clear which component each env uses
- ✅ **Kustomize-native** - Uses official Components feature
- ✅ **DRY for app code** - Shared base for deployments/services
- ✅ **Scalable** - Easy to add more environments (dev, qa, etc.)

### Cons
- ⚠️ **Requires understanding Components** - Less common than basic kustomize
- ⚠️ **File duplication** - External-secrets duplicated per component
- ⚠️ **Extra directory level** - `components/` adds nesting

## Comparison with Current (Broken) Approach

| Aspect | Current (Broken) | Option 2 (Components) |
|--------|------------------|----------------------|
| **Isolation** | ❌ Shared base | ✅ Separate components |
| **Patches needed** | ✅ Yes (fragile) | ❌ No |
| **Can staging break prod?** | ✅ Yes | ❌ No |
| **Explicit env selection?** | ❌ Implicit via patches | ✅ Explicit via components |
| **Kustomize-native?** | ⚠️ Patches (basic) | ✅ Components (advanced) |

## Migration Steps

For each component:

### 1. Create Component Directories
```bash
mkdir -p components/<COMPONENT>/components/staging-secrets/external-secrets
mkdir -p components/<COMPONENT>/components/production-secrets/external-secrets
```

### 2. Create Staging Secrets Component
```bash
cp components/<COMPONENT>/base/external-secrets/* \
   components/<COMPONENT>/components/staging-secrets/external-secrets/
```

Create `components/<COMPONENT>/components/staging-secrets/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
- external-secrets/pipelines-as-code-secret.yaml
```

### 3. Create Production Secrets Component
Similar to staging, but update secret values to production paths.

### 4. Update Staging Kustomization
```yaml
# Remove: ../../base/external-secrets from resources
# Add: ../../components/staging-secrets to components
```

### 5. Update Production Kustomization
```yaml
# Remove: ../../base/external-secrets from resources
# Remove: patches for external-secrets
# Add: ../../components/production-secrets to components
```

### 6. Test
```bash
kustomize build components/<COMPONENT>/staging/base
kustomize build components/<COMPONENT>/production/base
```

## When to Use Option 2

**Use Option 2 if:**
- ✅ You want to leverage kustomize Components
- ✅ You prefer keeping shared app code in one base
- ✅ Team is comfortable with advanced kustomize features
- ✅ You want explicit component selection

**Consider Option 1 if:**
- ✅ You prefer simpler directory structure
- ✅ Team is less familiar with kustomize Components
- ✅ You want maximum isolation (separate bases entirely)

## Files Changed

```
components/build-service/
├── components/
│   ├── staging-secrets/              ← NEW
│   │   ├── kustomization.yaml        (kind: Component)
│   │   └── external-secrets/
│   │       └── pipelines-as-code-secret.yaml
│   └── production-secrets/           ← NEW
│       ├── kustomization.yaml        (kind: Component)
│       └── external-secrets/
│           └── pipelines-as-code-secret.yaml
├── staging/base/
│   └── kustomization.yaml            ← MODIFIED (added component)
└── production/base/
    ├── kustomization.yaml            ← MODIFIED (added component, removed patch)
    └── pipelines-as-code-secret-patch.yaml  ← Can be deleted!
```

## Next Steps

1. Review this POC
2. Compare with Option 1 (see COMPARISON.md)
3. Decide which approach fits better
4. Approve and roll out

---

**Status:** Awaiting review
**Created:** 2026-01-19
**Branch:** poc-kustomize-components-external-secrets
