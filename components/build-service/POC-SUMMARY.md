# POC: Separate Base External-Secrets Architecture

## Executive Summary

This POC demonstrates a new architecture that **completely isolates staging and production external-secrets**, eliminating the risk of staging changes affecting production.

## What Changed

### Directory Structure

**New directories created:**
- `common/` - Truly shared resources (no environment-specific values)
- `staging/base/external-secrets/` - Staging-specific external secrets
- `production/base/external-secrets/` - Production-specific external secrets

**Files modified:**
- `staging/base/kustomization.yaml` - Now references `../../common` and local `external-secrets/`
- `production/base/kustomization.yaml` - Now references `../../common` and local `external-secrets/`

**Files removed (in concept):**
- Production patch for `pipelines-as-code-secret` - No longer needed!

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

### Key Observation
**No patches needed!** Each environment has its own external-secret file with the correct value already set.

## Incident Prevention

### Before (Vulnerable):
1. Update base external-secrets `apiVersion: v1beta1` → `v1`
2. Forgot to update production patch target `version: v1beta1` → `v1`
3. **Patch fails silently**
4. **Production uses staging secret key** ❌

### After (Safe):
1. Update staging external-secrets `apiVersion: v1beta1` → `v1`
2. Update production external-secrets `apiVersion: v1beta1` → `v1`
3. **No patches involved**
4. **Production always uses production secret key** ✅

## Benefits

| Aspect | Before | After |
|--------|--------|-------|
| **Isolation** | ❌ Shared base | ✅ Completely separate |
| **Patches needed** | ✅ Yes (fragile) | ❌ No |
| **Can staging break prod?** | ✅ Yes | ❌ No |
| **Silent failures possible?** | ✅ Yes (patch mismatch) | ❌ No |
| **Easy to understand?** | ❌ Requires understanding patches | ✅ What you see is what you get |

## Trade-offs

### Pros
- ✅ **Complete isolation** - Impossible to break production from staging
- ✅ **No patches** - Simpler, less error-prone
- ✅ **Explicit** - Each env owns its config
- ✅ **Testable** - Can independently verify each env

### Cons
- ⚠️ **File duplication** - External-secrets duplicated per env
- ⚠️ **Cross-env changes** - Need to update both if structure changes

### Why Duplication Is Acceptable
1. External-secrets are small (20-30 lines)
2. They **should** be different per environment
3. Having them separate makes differences **explicit**
4. Shared app resources (deployments, services) stay in `common/`

## Next Steps

### Option 1: Approve & Roll Out
1. Review this POC
2. Merge to main
3. Monitor build-service for 1 week
4. Migrate 6 critical components (image-controller, integration, etc.)
5. Migrate remaining components

### Option 2: Request Changes
- Provide feedback on approach
- Suggest alternatives
- Discuss concerns

## Migration Effort

**Per component:**
- Time: 30-60 minutes
- Risk: Low (can test with kustomize build)
- Rollback: Simple (revert PR)

**Total (all components):**
- ~25 components with external-secrets
- Estimate: 2-3 weeks
- Can parallelize after pilot

## Files in This POC

```
components/build-service/
├── POC-SUMMARY.md                     ← This file
├── MIGRATION-GUIDE.md                 ← Detailed migration guide
├── common/                            ← NEW: Shared resources
│   ├── allow-argocd-to-manage.yaml
│   ├── build-pipeline-config/
│   ├── build-pipeline-runner-rolebinding.yaml
│   ├── kustomization.yaml
│   ├── monitoring.yaml
│   └── rbac/
├── staging/base/
│   ├── external-secrets/              ← NEW: Staging-specific
│   │   ├── kustomization.yaml
│   │   └── pipelines-as-code-secret.yaml (key: staging/...)
│   ├── kustomization.yaml             ← MODIFIED
│   └── manager_resources_patch.yaml
└── production/base/
    ├── external-secrets/              ← NEW: Production-specific
    │   ├── kustomization.yaml
    │   └── pipelines-as-code-secret.yaml (key: production/...)
    ├── kustomization.yaml             ← MODIFIED
    ├── manager_resources_patch.yaml
    └── pipelines-as-code-secret-patch.yaml  ← Can be deleted!
```

## Validation Commands

```bash
# Test staging builds
kustomize build components/build-service/staging/base > /tmp/staging.yaml
grep "staging/pipeline-service" /tmp/staging.yaml  # Should find it

# Test production builds
kustomize build components/build-service/production/base > /tmp/production.yaml
grep "production/pipeline-service" /tmp/production.yaml  # Should find it

# Verify no staging values leak to production
grep "staging/" /tmp/production.yaml  # Should be empty!
```

## Questions to Consider

1. **Is file duplication acceptable for external-secrets?**
   - Recommendation: Yes, safety > DRY principle

2. **Should we migrate all at once or incrementally?**
   - Recommendation: Incrementally, one component at a time

3. **What about other shared resources?**
   - Recommendation: Only separate what's environment-specific

4. **Timeline for full migration?**
   - Recommendation: 2-3 weeks, low priority components can take longer

## Approval Checklist

- [ ] Architecture approved by RelEng team
- [ ] POC reviewed and validated
- [ ] Migration guide is clear
- [ ] Timeline agreed upon
- [ ] Communication plan for team

---

**Status:** Awaiting approval
**Created:** 2026-01-19
**Author:** Claude Code (via automated migration analysis)
