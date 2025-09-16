# Kyverno Centralized Configuration Migration Guide

## Overview
This document outlines the migration from individual cluster configurations to a centralized Kustomize-based approach for Kyverno configurations.

## New Structure

```
components/kyverno/
├── base/                           # Common base configurations
│   ├── kustomization.yaml
│   ├── kyverno-helm-generator.yaml
│   ├── kyverno-helm-values-base.yaml
│   └── job_resources.yaml
├── overlays/                       # Environment-specific overlays
│   ├── production/
│   │   ├── kustomization.yaml
│   │   ├── kyverno-helm-generator.yaml
│   │   ├── kyverno-helm-values-production.yaml
│   │   └── clusters/              # Cluster-specific patches
│   │       ├── stone-prod-p01/
│   │       │   ├── kustomization.yaml
│   │       │   └── kyverno-helm-values-patch.yaml
│   │       ├── stone-prod-p02/
│   │       └── ...
│   ├── staging/
│   └── development/
└── production/                     # EXISTING: Keep for backward compatibility
    └── [current structure]
```

## Benefits

1. **DRY Principle**: Common configurations are defined once in the base
2. **Easy Maintenance**: Changes to common settings only need to be made in one place
3. **Clear Separation**: Environment-specific vs cluster-specific configurations are clearly separated
4. **Incremental Migration**: Can migrate clusters one by one without breaking existing deployments

## Migration Steps

### Step 1: Create Base Configuration
- ✅ Created `base/` directory with common configurations
- ✅ Moved common security contexts, metering, and feature flags to base

### Step 2: Create Production Overlay
- ✅ Created `overlays/production/` with production-specific defaults
- ✅ Moved common production settings (replicas, leaderElectionRetryPeriod, etc.)

### Step 3: Create Cluster-Specific Patches
- ✅ Created example cluster configurations that only specify differences
- ✅ stone-prod-p01: Only resource limits
- ✅ stone-prod-p02: Resource limits + rate limiting + updateRequestThreshold

### Step 4: Update ArgoCD ApplicationSet
Update the ArgoCD ApplicationSet to point to the new structure:

```yaml
# In argo-cd-apps/base/member/infra-deployments/kyverno/kyverno.yaml
spec:
  template:
    spec:
      source:
        path: '{{values.sourceRoot}}/overlays/{{values.environment}}/clusters/{{values.clusterDir}}'
        # Instead of: path: '{{values.sourceRoot}}/{{values.environment}}/{{values.clusterDir}}'
```

### Step 5: Test Migration
1. Deploy one cluster using the new structure
2. Verify it works correctly
3. Gradually migrate other clusters

## Example Cluster Configurations

### stone-prod-p01 (Minimal differences)
```yaml
# Only resource limits differ from production defaults
admissionController:
  container:
    resources:
      requests:
        cpu: 4000m
        memory: 2Gi
      limits:
        cpu: 4000m
        memory: 2Gi
```

### stone-prod-p02 (More differences)
```yaml
# Resource limits + rate limiting + updateRequestThreshold
config:
  updateRequestThreshold: 4000

backgroundController:
  extraArgs:
    clientRateLimitBurst: 2000
    clientRateLimitQPS: 2000
```

## Rollback Plan
- Keep existing `production/` directory structure intact
- Can easily revert ArgoCD ApplicationSet to point back to old structure
- No data loss or configuration changes during migration

## Future Improvements
1. Add staging and development overlays
2. Create cluster-specific resource templates
3. Add validation for required cluster-specific configurations
4. Consider using Kustomize components for even more reusability
