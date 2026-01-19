# External-Secrets Architecture: Option 1 vs Option 2

## TL;DR - Which Should You Choose?

**Choose Option 1 (Separate Bases)** if you value:
- Maximum simplicity and clarity
- Easiest to understand for new team members
- Complete independence between environments

**Choose Option 2 (Kustomize Components)** if you value:
- Kustomize-native approach
- Explicit component selection
- Familiarity (similar to existing `rh-certs` component pattern)

**Both options are safe and prevent the January 2026 incident.**

---

## Side-by-Side Comparison

### Directory Structure

#### Option 1: Separate Bases
```
build-service/
├── common/                     ← Shared app resources
│   ├── deployment.yaml
│   ├── service.yaml
│   └── rbac/
├── staging/base/
│   ├── external-secrets/      ← Staging-specific
│   │   └── *.yaml
│   └── kustomization.yaml
│       resources:
│         - ../../common
│         - external-secrets
└── production/base/
    ├── external-secrets/      ← Production-specific
    │   └── *.yaml
    └── kustomization.yaml
        resources:
          - ../../common
          - external-secrets
```

#### Option 2: Kustomize Components
```
build-service/
├── base/                       ← Shared app resources
│   ├── deployment.yaml
│   ├── service.yaml
│   └── rbac/
├── components/
│   ├── staging-secrets/       ← Staging component
│   │   ├── kustomization.yaml (kind: Component)
│   │   └── external-secrets/*.yaml
│   └── production-secrets/    ← Production component
│       ├── kustomization.yaml (kind: Component)
│       └── external-secrets/*.yaml
├── staging/base/
│   └── kustomization.yaml
│       resources: [../../base]
│       components: [../../components/staging-secrets]
└── production/base/
    └── kustomization.yaml
        resources: [../../base]
        components: [../../components/production-secrets]
```

---

## Detailed Comparison Matrix

| Criterion | Option 1 (Separate Bases) | Option 2 (Components) | Winner |
|-----------|---------------------------|----------------------|--------|
| **Safety** | ✅ Complete isolation | ✅ Complete isolation | 🟰 Tie |
| **Simplicity** | ✅✅ Very simple | ✅ Simple | Option 1 |
| **Kustomize-native** | ✅ Standard resources | ✅✅ Uses Components feature | Option 2 |
| **Explicitness** | ✅ Clear directory structure | ✅✅ Explicit component selection | Option 2 |
| **Learning curve** | ✅✅ Minimal (basic kustomize) | ✅ Moderate (need to understand Components) | Option 1 |
| **File duplication** | ⚠️ External-secrets duplicated | ⚠️ External-secrets duplicated | 🟰 Tie |
| **Shared base** | ❌ Base renamed to `common/` | ✅ Base stays as `base/` | Option 2 |
| **Consistency** | ⚠️ New pattern | ✅ Similar to `rh-certs` component | Option 2 |
| **Debugging** | ✅✅ Very easy | ✅ Easy | Option 1 |
| **Adding new envs** | ✅ Add new directory | ✅ Add new component | 🟰 Tie |

---

## Code Examples

### Kustomization Files

#### Option 1 - Staging
```yaml
# staging/base/kustomization.yaml
resources:
- ../../common           # Shared resources
- external-secrets       # Local staging secrets
- https://github.com/...
```

#### Option 2 - Staging
```yaml
# staging/base/kustomization.yaml
resources:
- ../../base            # Shared resources
- https://github.com/...
components:
  - ../../components/staging-secrets  # Staging component
```

**Key Difference:**
- Option 1: External-secrets are **resources** (like any other yaml)
- Option 2: External-secrets are **components** (explicit modular selection)

---

## Mental Model

### Option 1: "Each Environment Owns Everything"
```
Staging owns:
  - Reference to common/ (shared code)
  - Its own external-secrets/ (staging-specific)

Production owns:
  - Reference to common/ (shared code)
  - Its own external-secrets/ (production-specific)
```

**Analogy:** Each environment has its own apartment with shared utilities.

### Option 2: "Environments Select Components"
```
Base provides: Shared app resources

Staging selects:
  - Base resources
  + staging-secrets component

Production selects:
  - Base resources
  + production-secrets component
```

**Analogy:** Build your environment by selecting components from a catalog.

---

## Migration Effort

| Task | Option 1 | Option 2 |
|------|----------|----------|
| Create new directories | `common/` + env external-secrets | `components/staging-secrets/` + `production-secrets/` |
| Move files | base/ → common/ | base/external-secrets/ → components/ |
| Update kustomizations | Change `../../base` → `../../common` | Add components, remove patches |
| Complexity | Low | Medium |
| Time per component | 30 min | 45 min |

---

## Real-World Scenarios

### Scenario 1: "I want to update the deployment for staging"

**Option 1:**
```bash
# Edit common/deployment.yaml
# Staging automatically picks it up (references ../../common)
✅ Simple, obvious
```

**Option 2:**
```bash
# Edit base/deployment.yaml
# Staging automatically picks it up (resources: ../../base)
✅ Simple, obvious
```

**Winner:** 🟰 Tie

---

### Scenario 2: "I want to update staging secrets only"

**Option 1:**
```bash
# Edit staging/base/external-secrets/pipelines-as-code-secret.yaml
✅ Clear location, can't affect production
```

**Option 2:**
```bash
# Edit components/staging-secrets/external-secrets/pipelines-as-code-secret.yaml
✅ Clear location, can't affect production
```

**Winner:** 🟰 Tie

---

### Scenario 3: "I want to add a new environment (e.g., dev)"

**Option 1:**
```bash
mkdir -p dev/base/external-secrets
# Create kustomization.yaml referencing ../../common
✅ Straightforward
```

**Option 2:**
```bash
mkdir -p components/dev-secrets
# Create component kustomization (kind: Component)
# Update dev/base/kustomization.yaml to reference component
✅ Straightforward
```

**Winner:** 🟰 Tie

---

### Scenario 4: "New team member needs to understand the structure"

**Option 1:**
```
"Here's how it works:
- common/ has shared stuff
- staging/base/external-secrets/ has staging secrets
- production/base/external-secrets/ has production secrets"

✅ Extremely clear
```

**Option 2:**
```
"Here's how it works:
- base/ has shared stuff
- components/staging-secrets is a Component that adds staging secrets
- staging kustomization includes the staging-secrets component"

⚠️ Requires explaining kustomize Components
```

**Winner:** Option 1

---

### Scenario 5: "I want to see all staging-specific config"

**Option 1:**
```bash
ls staging/base/external-secrets/
✅ All in one place
```

**Option 2:**
```bash
ls components/staging-secrets/
✅ All in one place, but different location
```

**Winner:** 🟰 Tie (just different locations)

---

## Team Considerations

### If Your Team...

**...is new to kustomize:**
→ **Option 1** (simpler mental model)

**...already uses kustomize Components extensively:**
→ **Option 2** (consistent with existing patterns)

**...values simplicity over "proper" patterns:**
→ **Option 1**

**...values kustomize-native approaches:**
→ **Option 2**

**...wants maximum explicitness:**
→ **Option 2** (components make selection explicit)

**...wants easiest debugging:**
→ **Option 1** (everything visible in directory structure)

---

## Testing & Validation

Both options validate exactly the same way:

```bash
# Test staging
kustomize build components/build-service/staging/base | grep "key:"
# Should show: staging/pipeline-service/github-app

# Test production
kustomize build components/build-service/production/base | grep "key:"
# Should show: production/pipeline-service/github-app
```

✅ Both pass validation identically

---

## Incident Prevention

**The January 2026 Incident:**
- Base external-secrets updated v1beta1 → v1
- Production patches still targeted v1beta1
- Patch failed silently
- Production used staging secrets ❌

### How Each Option Prevents This

**Option 1:**
```
✅ No shared base for external-secrets
✅ No patches needed
✅ Impossible to have version mismatch
✅ Staging changes can't affect production files
```

**Option 2:**
```
✅ No shared base for external-secrets
✅ No patches needed
✅ Impossible to have version mismatch
✅ Staging component can't affect production component
```

**Winner:** 🟰 Tie (both completely solve the problem)

---

## Long-Term Maintenance

### Option 1
**Pros:**
- Easy to understand months/years later
- New team members onboard quickly
- Simple to debug issues

**Cons:**
- Changes to external-secret structure need updating in 2 places
- Slightly more file duplication

### Option 2
**Pros:**
- Uses kustomize features "properly"
- Consistent with existing component patterns
- May be more familiar to kustomize experts

**Cons:**
- Requires team to understand Components
- Slightly more complex to debug
- Changes to external-secret structure need updating in 2 components

---

## Recommendation Framework

### Answer These Questions:

1. **Does your team already use kustomize Components?**
   - Yes → Option 2 fits existing patterns
   - No → Option 1 is simpler

2. **What does your team value more?**
   - Simplicity → Option 1
   - Kustomize-native patterns → Option 2

3. **How experienced is your team with kustomize?**
   - Beginners → Option 1
   - Experts → Either option works

4. **Do you plan to add more environments (dev, qa, perf)?**
   - Yes → Option 2 scales with components
   - No → Option 1 is sufficient

5. **What's your priority?**
   - Easiest to understand → Option 1
   - Most "correct" kustomize usage → Option 2

---

## Final Recommendation

**For Most Teams: Choose Option 1**

Reasons:
- ✅ Simpler to understand and maintain
- ✅ Easier onboarding for new team members
- ✅ Clearer directory structure
- ✅ Easier debugging
- ✅ Both options are equally safe

**When to Choose Option 2:**
- Team already heavily uses kustomize Components
- Strong preference for kustomize-native patterns
- Team has deep kustomize expertise

---

## Migration Timeline (Both Options)

**Week 1:** POC review & approval, merge build-service
**Week 2:** Monitor build-service, migrate 3 critical components
**Week 3:** Migrate 3 more critical components
**Week 4+:** Migrate remaining ~18 components

**Total:** 2-3 weeks for full migration (same for both options)

---

## Decision Checklist

- [ ] Team has reviewed both POCs
- [ ] Tested both approaches with kustomize build
- [ ] Discussed trade-offs
- [ ] Consensus on preferred approach
- [ ] Approved for rollout

---

## Getting Help

**Questions about Option 1:** See `POC-SUMMARY.md`
**Questions about Option 2:** See `OPTION2-POC-SUMMARY.md`
**Migration steps:** See `MIGRATION-GUIDE.md`

**Contact:** RelEng Team (@rhartman @bhills @pkhander @shebert @jkubica)

---

**Document Version:** 1.0
**Last Updated:** 2026-01-19
**Authors:** Claude Code (automated analysis)
