# Build Service: External Secrets Migration Guide

## Context

This migration addresses a critical architectural flaw where staging and production environments shared the same base external-secrets files. This created a dangerous situation where:

1. **Changes intended for staging could break production**
2. **Version mismatches in patches caused silent failures**
3. **Production accidentally used staging secrets**

### The Incident (January 2026)

When ESO API v1beta1 → v1 migration Phase 1 Fix was deployed:
- Base external-secrets were updated to `apiVersion: v1`
- Production patches still targeted `version: v1beta1`
- **Result:** Patches failed silently, production used staging secret keys

## New Architecture

### Before (PROBLEMATIC):
```
components/build-service/
├── base/
│   ├── external-secrets/
│   │   └── pipelines-as-code-secret.yaml (key: staging/...)  ← SHARED BY BOTH!
│   ├── allow-argocd-to-manage.yaml
│   └── ...
├── staging/base/
│   └── kustomization.yaml  → references ../../base/external-secrets
└── production/base/
    ├── kustomization.yaml  → references ../../base/external-secrets
    └── pipelines-as-code-secret-patch.yaml  ← Patches staging→production
```

**Problems:**
- ❌ Shared base contains environment-specific values (staging)
- ❌ Production depends on patches that can fail silently
- ❌ No isolation between environments
- ❌ Changing base for staging affects production

### After (SAFE):
```
components/build-service/
├── common/                    ← Truly shared (no env-specific values)
│   ├── allow-argocd-to-manage.yaml
│   ├── build-pipeline-config/
│   ├── monitoring.yaml
│   ├── rbac/
│   └── kustomization.yaml
├── staging/base/
│   ├── external-secrets/      ← STAGING-SPECIFIC
│   │   ├── pipelines-as-code-secret.yaml (key: staging/...)
│   │   └── kustomization.yaml
│   └── kustomization.yaml     → references ../../common + external-secrets/
└── production/base/
    ├── external-secrets/      ← PRODUCTION-SPECIFIC
    │   ├── pipelines-as-code-secret.yaml (key: production/...)
    │   └── kustomization.yaml
    └── kustomization.yaml     → references ../../common + external-secrets/
```

**Benefits:**
- ✅ Complete isolation - staging changes never affect production
- ✅ No patches needed - values are environment-specific
- ✅ Clear ownership - each env owns its secrets
- ✅ Impossible to break production from staging changes
- ✅ Easy to understand - no hidden dependencies

## Key Changes

### 1. Created `common/` directory
- Contains truly shared resources (deployments, services, RBAC, etc.)
- **NO environment-specific values**
- Safe to change without impacting environments

### 2. Separated external-secrets by environment
- `staging/base/external-secrets/` - staging-specific
- `production/base/external-secrets/` - production-specific
- Each contains the same structure but with environment-appropriate values

### 3. Updated kustomization references
- Changed from `../../base` to `../../common`
- Changed from `../../base/external-secrets` to `external-secrets`
- Removed production patch (no longer needed!)

## Verification

### Staging builds with staging key:
```bash
$ kustomize build components/build-service/staging/base | grep -A5 "kind: ExternalSecret"
kind: ExternalSecret
spec:
  dataFrom:
  - extract:
      key: staging/pipeline-service/github-app  ✅
```

### Production builds with production key:
```bash
$ kustomize build components/build-service/production/base | grep -A5 "kind: ExternalSecret"
kind: ExternalSecret
spec:
  dataFrom:
  - extract:
      key: production/pipeline-service/github-app  ✅
```

## Migration Steps for Other Components

To migrate other components to this pattern:

### 1. Create common directory
```bash
mkdir -p components/<COMPONENT>/common
cp -r components/<COMPONENT>/base/* components/<COMPONENT>/common/
rm -rf components/<COMPONENT>/common/external-secrets
```

### 2. Create staging external-secrets
```bash
mkdir -p components/<COMPONENT>/staging/base/external-secrets
cp components/<COMPONENT>/base/external-secrets/* \
   components/<COMPONENT>/staging/base/external-secrets/
```

### 3. Create production external-secrets
```bash
mkdir -p components/<COMPONENT>/production/base/external-secrets
cp components/<COMPONENT>/base/external-secrets/* \
   components/<COMPONENT>/production/base/external-secrets/
```

### 4. Update production secret values
Edit `components/<COMPONENT>/production/base/external-secrets/*.yaml`:
- Change `staging/...` → `production/...`

### 5. Update staging/base/kustomization.yaml
```yaml
resources:
- ../../common           # was: ../../base
- external-secrets       # was: ../../base/external-secrets
```

### 6. Update production/base/kustomization.yaml
```yaml
resources:
- ../../common           # was: ../../base
- external-secrets       # was: ../../base/external-secrets

# Remove patches for external-secrets (no longer needed!)
```

### 7. Test
```bash
kustomize build components/<COMPONENT>/staging/base
kustomize build components/<COMPONENT>/production/base
```

### 8. Verify
Check that:
- Staging uses staging secret keys
- Production uses production secret keys
- No patches needed for external-secrets

## Components to Migrate

Based on the ESO v1 migration analysis, these components need migration:

**High Priority (have production patches for secrets):**
- ✅ build-service (POC completed)
- image-controller
- image-rbac-proxy
- integration
- konflux-kite
- mintmaker
- pipeline-service

**Medium Priority:**
- dora-metrics
- has
- notification-controller
- monitoring/logging
- monitoring/prometheus
- tracing/otel-collector

**Others:**
- All remaining components with external-secrets

## Rollout Strategy

### Phase 1: Pilot (build-service)
- ✅ POC completed
- Review and get approval
- Merge to main
- Monitor for 1 week

### Phase 2: Critical Components
- Migrate components with production patches (6 components)
- One PR per component
- Each component validated independently

### Phase 3: Remaining Components
- Migrate all other components
- Can batch compatible components

## Testing Checklist

For each migrated component:

- [ ] `kustomize build` succeeds for staging
- [ ] `kustomize build` succeeds for production
- [ ] Staging manifest contains staging secret keys
- [ ] Production manifest contains production secret keys
- [ ] No unexpected patches in manifests
- [ ] ArgoCD syncs successfully in staging
- [ ] ArgoCD syncs successfully in production
- [ ] ExternalSecrets reconcile successfully
- [ ] Actual secrets created with correct values

## Backward Compatibility

**Important:** This change is **NOT** backward compatible with the old structure.

- Old: `../../base/external-secrets`
- New: `external-secrets` (local to environment)

Components must be migrated completely in one PR. Partial migration will break builds.

## Future Improvements

After all components are migrated:

1. **Delete old base/external-secrets/**
   - No longer needed
   - Prevents accidental usage

2. **Add validation**
   - CI check: no `../../base/external-secrets` references
   - CI check: no external-secrets patches with version targets

3. **Documentation**
   - Update onboarding docs
   - Add to architecture decision records (ADR)

## Questions?

Contact: RelEng Team (@rhartman @bhills @pkhander @shebert @jkubica)
