---
layout: default
title: limitabl
---

# limitabl

**rate limits, quotas, and spend controls for llm and agentic apps**

once a request is **identified**, **transformed**, and **authorized**, the next governance question is:

**how much can this user (or org, or agent) consume?**

limitabl answers that question.

it enforces per-user, per-tenant, per-model, and per-provider limits — ensuring predictable costs, fair usage, and controlled spend across your entire llm ecosystem.

## at a glance

limitabl is the **usage, rate-limit, and spend-governance layer** of gatewaystack.  
it lets you:

- enforce per-user and per-org rate limits  
- apply quotas and usage ceilings  
- enforce budget caps by model, provider, or tenant  
- detect anomalies in spend or usage  
- prevent runaway agents or infinite tool loops  

> 📦 **implementation**: [`ai-rate-limit-gateway`](https://github.com/davidcrowe/gatewaystack) + [`ai-cost-gateway`](https://github.com/davidcrowe/gatewaystack) (roadmap)

## why now?

as llm adoption increases, so do cost overruns, unexpected spikes, and unbounded agent behavior.

organizations need:

- predictable spend  
- tenant-aware usage controls  
- protection against recursive tool loops  
- guardrails for internal and external users  
- the ability to stop unexpected cost explosions in real time  

limitabl is the answer — granular control over usage and spend.

## designing the usage & spend governance layer

### within the shared requestcontext

all gatewaystack modules operate on a shared [`RequestContext`](https://github.com/davidcrowe/gatewaystack/blob/main/docs/reference/interfaces.md) object.

**limitabl operates in two phases**:

**phase 1 (pre-flight)**:
- **reading**: `identity`, `modelRequest`, `policyDecision` (optional)
- **writing**: `limitsDecision` — constraints and effects before execution (`ok` | `throttle` | `deny` | `fallback` | `degrade`)

**phase 2 (post-execution)**:
- **reading**: provider response (tokens, cost, latency)
- **writing**: `usage` — actual tokens, cost, and latency for the call; updates quota/budget stores; emits usage events

limitabl runs immediately **before** llm execution (pre-flight checks) and again **after** the provider responds (usage accounting).

### pre-flight evaluation

in the pre-flight phase, limitabl evaluates:

- rate-limit windows  
- usage quotas  
- global and tenant-specific budgets  
- model-level cost ceilings  
- risk-aware spend (for example, expensive models restricted for low-trust contexts)

if a limit is exceeded, limitabl returns a structured governance decision (`deny`, `throttle`, `fallback`, or `degrade`).

### post-execution accounting

in the post-execution phase, limitabl records actual usage and updates internal counters and budgets.

## the core functions

**1. `checkRateLimit` — enforce per-user and per-tenant rates**  
requests per second/minute/hour, sliding windows, token buckets.

**2. `checkQuota` — validate remaining quota**  
daily, weekly, monthly usage ceilings.

**3. `checkBudget` — enforce cost ceilings**  
budget ceilings per user, tenant, environment, or provider.

**4. `checkRiskCost` — risk-based spend rules**  
for example, only certain scopes can call expensive reasoning models.

**5. `applyThrottling` — delay or reject when limits are hit**  
adaptive or static throttling policies.

**6. `fallbackProvider` — reroute to cheaper models**  
automatic degradation modes when quotas or budgets are reached.

**7. `emitUsageEvents` — log usage for observability**  
produces structured records for `explicabl`'s audit pipeline.

## what limitabl does

- enforces usage and cost controls  
- protects tenants from overuse or over-spend  
- throttles or blocks excessive request patterns  
- shifts traffic to alternative providers when needed  
- provides the spend-governance layer enterprises require  
- populates the `limitsDecision` and `usage` fields in `RequestContext`  

## what limitabl does not do

- it does not authenticate identity (see `identifiabl`)  
- it does not preprocess content (see `transformabl`)  
- it does not evaluate policies (see `validatabl`)  
- it does not perform routing (see `proxyabl`)  
- it does not store audit logs (see `explicabl`)  

## two-phase limit enforcement

limitabl's two-phase design ensures both preventive controls and accurate accounting:

**phase 1: pre-flight checks (before routing)**  
- verify rate limits not exceeded  
- check quota availability  
- estimate budget impact  
- return constraints to `proxyabl` via `limitsDecision`  

**phase 2: usage accounting (after execution)**  
- record actual tokens used  
- deduct from quotas and budgets  
- update rate-limit counters  
- emit usage events for `explicabl`  
- update `usage` in `RequestContext`  

this ensures both preventive controls and accurate accounting.

## limit configuration

limits are defined hierarchically:
```yaml
limits:
  global:
    rate: 10000/min
    budget: $5000/day

  organizations:
    org_healthcare:
      rate: 1000/min
      budget: $500/day
      models:
        gpt-4:
          quota: 100000 tokens/day
          budget: $200/day

  users:
    user_doctor_123:
      rate: 100/min
      budget: $50/day
```

**precedence**: user limits override org limits, which override global limits.  
**enforcement**: the most restrictive limit applies.

## agent loop protection

limitabl prevents runaway agentic behavior:
```yaml
agent_protection:
  max_tool_calls_per_workflow: 20
  max_recursion_depth: 5
  max_workflow_cost: $2.00
  max_workflow_duration: 120s
  duplicate_tool_detection:
    enabled: true
    threshold: 3  # same tool with same params
```

**example**: an agent enters an infinite `web_search` loop. after 10 identical calls, limitabl terminates the workflow and returns an error.

## distributed rate limiting

in multi-instance deployments, limitabl uses shared state:
```yaml
storage:
  backend: "redis"
  cluster:
    - "redis://primary:6379"
    - "redis://replica-1:6379"
  consistency: "strong"
  ttl: "3600s"
```

rate limits are enforced across all gatewaystack instances with strong consistency guarantees.

## end to end flow
```text
user
   → identifiabl       (who is calling?)
   → transformabl      (prepare, clean, classify, anonymize)
   → validatabl        (is this allowed?)
   → limitabl          (how much can they use? pre-flight constraints)
   → proxyabl          (where does it go? execute)
   → llm provider      (model call)
   → [limitabl]        (deduct actual usage, update quotas/budgets)
   → explicabl         (what happened?)
   → response
```

limitabl enforces **predictable usage, stable cost, and controlled access**, preventing abuse, overspend, and runaway agents.

## integrates with your existing stack

limitabl plugs into gatewaystack and your existing llm stack without requiring application-level changes. it exposes http middleware and sdk hooks for:

- chatgpt apps sdk  
- model context protocol (mcp)  
- oauth2 / oidc identity providers  
- any llm provider (openai, anthropic, google, internal models)  

## getting started

**for limit configuration examples**:  
→ [rate limit patterns](https://github.com/davidcrowe/gatewaystack/blob/main/docs/examples/rate-limit-examples.md)  
→ [budget configuration guide](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/budget-controls.md)  
→ [agent loop protection](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/agent-protection.md)

**for implementation**:  
→ [integration guide](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/integration.md)

## links

want to explore the full gatewaystack architecture?  
→ [view the gatewaystack github repo](https://github.com/davidcrowe/gatewaystack)

want to contact us for enterprise deployments?  
→ [reducibl applied ai studio](https://reducibl.com)

<div class="arch-diagram">
  <div class="arch-row">
    <div class="arch-node">
      <div class="arch-node-title">app / agent</div>
      <div class="arch-node-sub">chat ui · internal tool · agent runtime</div>
    </div>

    <div class="arch-arrow">→</div>

    <div class="arch-node arch-node-gateway">
      <div class="arch-node-title">gatewaystack</div>
      <div class="arch-node-sub">user-scoped trust &amp; governance gateway</div>

      <div class="arch-pill-row">
        <span class="arch-pill">identifiabl</span>
        <span class="arch-pill">transformabl</span>
        <span class="arch-pill">validatabl</span>
        <span class="arch-pill">limitabl</span>
        <span class="arch-pill">proxyabl</span>
        <span class="arch-pill">explicabl</span>
      </div>
    </div>

    <div class="arch-arrow">→</div>

    <div class="arch-node">
      <div class="arch-node-title">llm providers</div>
      <div class="arch-node-sub">openai · anthropic · internal models</div>
    </div>
  </div>

  <p class="arch-caption">
    every request flows from your app through gatewaystack's modules before it reaches an llm provider —
    <strong>identified, transformed, validated, constrained, routed, and audited.</strong>
  </p>
</div>