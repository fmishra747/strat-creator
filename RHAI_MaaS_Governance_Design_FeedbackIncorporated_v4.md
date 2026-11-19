# MaaS Governance — Updated Design

*RHAIRFE-2235  |  RHAISTRAT-2705  |  Additive request-rate-limit proposal*

## Business Need

Extend MaaS governance to request-count quotas for model endpoints, including rerankers served at /v1/rerank, while preserving existing LLM token quotas. Request limits are generic and can be configured for chat, rerank, and other model types.

## TL;DR

Keep modelRefs[].tokenRateLimits unchanged. Add optional modelRefs[].rateLimit using the existing {limit, window} shape. Select Kuadrant policies from the presence of each field, not model modality. Reuse existing AI Gateway routing/authentication where available.

## Technical Approach

### 1. Additive MaaSSubscription schema

Retain tokenRateLimits: [{limit, window}] with no migration.

Add rateLimit: [{limit, window}] to count endpoint hits, independent of model modality. Both fields may appear on one model reference; each limit is independent.

### 2. Kuadrant policy selection

tokenRateLimits → TokenRateLimitPolicy; rateLimit → RateLimitPolicy. No rateLimit.type, quota renaming, or modality-based selection. Subscription-scoped counters must use authenticated subscription identity; requests over quota return HTTP 429.

### 3. MaaSModelRef modality (not discovery)

Use the MaaSModelRef modality enum being developed by the MaaS team for model classification, subscription validation, and dashboard telemetry filtering. Token-limit compatibility must be capability-aware: reject unsupported token limits for rerank-only endpoints, while preserving existing embedding configurations that use tokenRateLimits. Request quotas are allowed across modalities. This proposal does not specify or introduce an InferenceService annotation, automatic discovery watch, or new modality source.

**Validation Logic (MaaSSubscriptionReconciler):**

```python
if (
    model_ref.modality == "rerank"
    and subscription_model_ref.tokenRateLimits
):
    raise ValidationError(
        "tokenRateLimits is not supported for rerank models. "
        "Use rateLimit for request-based quotas."
    )
```

This ensures rerank models use request-based quotas while preserving token limits for chat and embedding models.

### 4. Gateway routing and authorization

Reuse existing AI Gateway route and Kuadrant AuthPolicy for model endpoints including /v1/rerank, subject to deployment coverage. Require MaaS API-key authentication and subscription authorization. Do not generate additional reranker-specific HTTPRoutes by default.

### 5. Telemetry and dashboard

Reuse the existing captured request-count telemetry for rerank showback rather than introducing reranker-specific collection. Retain token metrics for models supporting token accounting. Extend existing MaaS BFF/UI patterns to display both quota families and modality filtering; align dashboard presentation with the dashboard team. Document-count metrics require separate payload-aware instrumentation.

## Additional Design Clarifications (Reviewer Feedback)

### vLLM chat/rerank serving and policy coexistence

Current technical assessment suggests that a single vLLM deployment serves **either chat completion or reranking**, not both simultaneously. **Confirm this serving limitation with the Model Serving team** before treating it as a platform guarantee. The current MaaS rate-limit design does not require a single vLLM deployment to serve both chat and rerank.

This serving distinction does **not** eliminate the need for both Kuadrant policy types on one LLM model reference:

- `tokenRateLimits` configures `TokenRateLimitPolicy` (TRLP) for token-accounted traffic.
- `rateLimit` configures `RateLimitPolicy` (RLP) for endpoint request counts, regardless of modality.
- An LLM subscription may configure **both** fields to enforce independent token and request quotas on the same chat endpoint. Verify that the deployed Kuadrant configuration supports simultaneous enforcement, correct policy targeting, and subscription-isolated counters. This is distinct from a vLLM deployment serving both chat and rerank.
- A rerank-only endpoint uses request-based limits; token limits on that endpoint are rejected. Existing embedding `tokenRateLimits` configurations remain valid.

`MaaSModelRef` modality remains the MaaS source of truth for model classification. No model-discovery mechanism is proposed in this design.

### Unlimited request limits

Consider an `unlimited` option for request-based RLP quotas, consistent with existing MaaS semantics. Reuse the established unlimited representation if one exists; do not introduce a new sentinel without checking the contract. A possible implementation is to omit the request-limit policy while retaining authentication, authorization, and usage telemetry. Exact behavior is a product/API decision, not yet confirmed.

### Token-limit validation: avoid embedding regression

Do **not** implement a blanket `modality != chat` rejection for `tokenRateLimits`. The shipped embedding sample uses token limits and must remain valid. Reject unsupported token-based limits on **rerank-only** models or endpoints, while preserving token-accounted chat and embedding configurations. Validation should reflect actual supported capabilities and endpoint behavior, not a broad non-chat check.

### Reranker showback

Request-count information is **already captured**. Reuse the existing telemetry and show it through the MaaS dashboard rather than adding reranker-specific instrumentation. Align with the dashboard team on how to surface existing request counts, including model/subscription breakdowns and relevant permissions. This is a dashboard presentation/integration enhancement, not a new telemetry collection requirement. Document-count metrics would require separate payload-aware instrumentation; showback does not imply billing or chargeback.

## Proposed Subscription Examples

### Existing LLM — unchanged

```yaml
modelRefs:
  - name: free-model
    namespace: llm
    tokenRateLimits:
      - limit: 100
        window: 1m
```

### LLM — token and request limits

```yaml
modelRefs:
  - name: free-model
    namespace: llm
    tokenRateLimits:
      - limit: 10000
        window: 1m
    rateLimit:
      - limit: 100
        window: 1m
```

### Reranker — request limit

```yaml
modelRefs:
  - name: reranker-model
    namespace: llm
    rateLimit:
      - limit: 100
        window: 1m
```

## Requirements and Acceptance Criteria

- P0: Existing tokenRateLimits subscriptions continue without schema migration or behavioral regression.
- P0: Valid subscribed API keys reach authorized endpoints; missing/invalid credentials are rejected.
- P0: rateLimit enforces subscription-isolated endpoint-hit quotas on any supported model type, returning HTTP 429 on excess.
- P0: Configuring both fields on one model reference supports independent token and request limits, subject to policy coexistence verification.
- P1: Dashboard and showback distinguish token and request counts; surface existing reranker request-count telemetry through the shared dashboard with capability/modality filtering, with dashboard presentation aligned across teams.
- No P2 model-discovery work is proposed in this design.
## Affected Components

MaaS controller/CRD: additive rateLimit schema, reconciliation and compatibility validation. AI Gateway/Kuadrant: routing, authentication, subscription-scoped policy enforcement, telemetry. odh-dashboard: quota display, usage and modality filtering. MaaSModelRef modality enum is aligned with the separate ongoing model-classification work.

## Dependencies and Non-Goals

Dependencies: Kuadrant, Gateway API, existing MaaS API-key/subscription authorization, vLLM rerank serving, admin showback RHAISTRAT-1235 and non-admin dashboards RHAISTRAT-1378. Non-goals: new model discovery, KServe model-type annotations, additional reranker HTTPRoutes by default, document-weighted quota enforcement, runtime changes, external reranker egress, and embedding-specific governance features beyond generic endpoint request limits.

## Risks and Engineering Validation

Verify that the installed Kuadrant version supports simultaneous RateLimitPolicy and TokenRateLimitPolicy targeting with correct per-subscription isolation and multiple API keys. Confirm with Model Serving whether vLLM deployments serve chat or rerank exclusively. Verify /v1/rerank route/auth coverage, CRD schema compatibility, dual-policy enforcement on chat endpoints, existing embedding token-limit behavior, unlimited quota semantics, and metric cardinality. Align on reusing captured request counts in the dashboard. These are validation items, not new discovery requirements.

## Supporting Documents

RHAI_MaaS_Governance_Aligned_Samples.yaml (legacy LLM, dual-limit LLM, reranker, generic non-reranker examples); RHAI_MaaS_Major_Architectural_Changes_No_Discovery.md (architecture comparison); RHAI_MaaS_Validation_and_Coordination_Next_Steps.md (separate follow-ups). Sources: RHAIRFE-2235 and RHAISTRAT-2705.
