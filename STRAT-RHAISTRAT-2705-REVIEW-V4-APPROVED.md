# STRAT RHAISTRAT-2705 Review — v4 Approved Design

**Status:** ✅ **APPROVE (8/8)** — Ready for engineering

**Changes:** Updated strategy to align with v4 simplified design (Jamie, Ishita, Ryan approved)

---

## Scoring Improvement: 5/8 → 8/8

| Dimension | Old | New | Status |
|-----------|-----|-----|--------|
| **Feasibility** | 1/2 ❌ | 2/2 ✅ | v4 reduces complexity |
| **Testability** | 1/2 ❌ | 2/2 ✅ | Clearer acceptance criteria |
| **Scope** | 2/2 ✅ | 2/2 ✅ | Maintained focus |
| **Architecture** | 1/2 ❌ | 2/2 ✅ | Simpler design, fewer issues |
| **Total** | **5/8 REVISE** | **8/8 APPROVE** | **Ready for engineering** |

---

## What Changed (v4 Simplifications)

### ✅ Simpler CRD Schema
**Old:** `spec.rateLimit.type: "token" | "request"` with complex policy selection logic  
**New:** Just add `spec.rateLimit: [{limit, window}]` alongside existing `spec.tokenRateLimits`

**Why:** Policy selection by field presence is clearer and simpler than type-driven selection.

---

### ✅ No HTTPRoute Generation
**Old:** "MaaSModelRefReconciler generates new HTTPRoutes for /v1/rerank"  
**New:** "Do not generate additional reranker-specific HTTPRoutes by default"

**Why:** Existing AI Gateway already routes /v1/rerank; avoids routing duplication.

---

### ✅ No InferenceService Discovery
**Old:** InferenceService watch with model-type annotations  
**New:** "No model-discovery mechanism is proposed in this design"

**Why:** MaaSModelRef creation is assumed manual; keeps scope focused.

---

### ✅ Simplified Telemetry
**Old:** "Requires extending the telemetry pipeline to capture reranker-specific metrics"  
**New:** "Request-count information is already captured. Reuse that existing telemetry"

**Why:** Gateway already captures request counts; no new instrumentation needed.

---

### ✅ Both Quota Types Can Coexist
**Old:** Dedicated subscriptions per model type  
**New:** One MaaSSubscription can have both `tokenRateLimits` and `rateLimit`

**Why:** More flexible; supports hybrid quota models if needed.

---

## Scorer Findings (Full Details)

### Feasibility (2/2) ✅
- **v4 effort estimate (M / 3-5 sprints)** is well-matched
- Kuadrant RateLimitPolicy validation is critical path; spike identified with fallback
- Simplified design reduces implementation complexity
- No breaking changes to existing token quotas

### Testability (2/2) ✅
- All acceptance criteria are binary-verifiable
- Concrete measurements: HTTP 429, Prometheus metrics, RateLimitPolicy existence
- Happy path and failure modes covered
- No ambiguous success criteria

### Scope (2/2) ✅
- Narrowly focused on MaaS governance layer
- Single coherent deliverable
- Explicit out-of-scope list (runtime, embedding, external egress, mixed-type subscriptions)
- No scope expansion risk

### Architecture (2/2) ✅
- Components verified against RHOAI architecture context
- Integration patterns grounded in documented design
- Field-presence policy selection is simpler than modality-driven approach
- Backward compatibility preserved
- No conflicts with other strategies

---

## Key Strengths (v4 Design)

1. **Simplicity:** Field-presence policy selection instead of modality-driven logic
2. **Reuse:** Leverages existing MaaS reconciler patterns and AI Gateway telemetry
3. **Backward compatible:** No schema migration for existing subscriptions
4. **Generalized:** Extends to future non-token-based model types (embedding, etc.)
5. **Clear risks:** Kuadrant validation spike identified with credible fallback
6. **No new discovery:** Avoids InferenceService annotation/watch complexity

---

## Ready for Engineering

**Approved:** 8/8 (all dimensions)  
**Needs attention:** false  
**Next:** Sprint planning with cross-team coordination

**Key engineering work:**
- P0: MaaS CRD `rateLimit` field + RateLimitPolicy reconciliation (1-2 sprints)
- P0: Kuadrant validation spike (sprint 1, blocking)
- P1: Dashboard dual-quota display + modality filtering (~1 sprint)
- P1: Telemetry integration with showback strategies (0.5-1 sprint)

---

## Files Updated

- `artifacts/strat-tasks/RHAISTRAT-2705.md` — Updated with v4 design
- `artifacts/strat-reviews/RHAISTRAT-2705.md` — Review file (8/8 APPROVE)
- `tmp/strat-assess/review/RHAISTRAT-2705.result.md` — Full scorer assessment

---

## Status

✅ **READY FOR SIGN-OFF**

No further revisions needed. Strategy is engineering-ready.

