# Lab 7: Application & API Security Architecture Design

Check box if done: [ ]

## Overview
Lab 6 secured how users and workloads *reach* applications. This lab secures the applications and APIs themselves — the layer that Private Access and Conditional Access can't protect once a request is actually let through. A finance API sitting behind a perfectly designed Zero Trust network is still exploitable if it has no rate limiting, accepts unvalidated tokens, or ships a SQL injection vulnerability that was never caught before release. This lab designs the enforcement point for API security and the secure SDLC strategy that catches vulnerabilities before they reach that enforcement point at all.

**Estimated time**: 60–75 minutes
**Cost**: ~$1–$3 (API Management Consumption tier bills per call at a low per-million rate and has no idle hourly charge; this lab stays well under any meaningful cost if cleaned up same-session)

---

## Scenario
The org's application team ships a finance API (the same finance data from Lab 1's incident) that's about to be exposed for the first time to the contractor population from Lab 2 and, eventually, to a partner integration. The team's current process is "deploy it, then run a pen test before the partner integration goes live" — no automated scanning in the CI/CD pipeline, no API gateway, just the app directly behind Lab 6's Private Access policy. You're designing where API-layer security actually gets enforced (gateway vs. WAF vs. both) and how the secure SDLC catches issues before Private Access and a gateway ever see the traffic.

---

## Objectives
- Decide between WAF-only, API Management (APIM) built-in policies, and a layered combination for API-layer enforcement
- Decide between manual pre-release pen testing, automated SAST/DAST in CI/CD, and runtime-only protection for the secure SDLC
- Deploy an APIM instance and configure JWT validation and rate-limiting policies
- Validate that an unauthenticated request is rejected and that rate limiting actually triggers
- Connect this lab's API gateway decision to Lab 6's network access decision and AZ-400's pipeline security
- Apply gateway-level response shaping so least-privilege access extends to individual response fields, not just endpoint access

---

## Part 1: Design Decision — API Enforcement Point and Secure SDLC Model

### Decision 1: WAF-Only vs. APIM Built-In Policies vs. Layered (Both)

| Factor | WAF-Only (Application Gateway/Front Door WAF in front of the API, no gateway policies) | APIM Built-In Policies Only (JWT validation, rate limiting, IP filtering — no WAF) | Layered — WAF + APIM Together |
|---|---|---|---|
| **OWASP Top 10 web attack coverage** (SQLi, XSS, etc.) | Strong — WAF managed rule sets are purpose-built for these signature-based attacks | Weak — APIM policies aren't a signature-based web attack engine | Strong — WAF still does what it's built for |
| **API-specific abuse coverage** (broken object-level auth, excessive data exposure, credential stuffing via valid-looking tokens) | Weak — a WAF doesn't understand API semantics like scopes, claims, or per-subscription rate budgets | Strong — APIM validates JWT claims/scopes per operation, enforces per-key/per-subscription rate limits, and can strip fields from responses | Strong — each layer covers what the other structurally can't |
| **Auth/token validation capability** | None — WAFs don't parse or validate OAuth tokens | Strong — `validate-jwt` policy checks issuer, audience, expiry, and required claims before the request reaches the backend | Strong |
| **Cost** | Lower — one component | Lower — one component | Higher — both components billed and operated |
| **Operational ownership** | Usually the network team | Usually the API/platform team | Split ownership, needs a clear RACI or this becomes the two teams' blind-spot boundary |
| **Fit for this scenario** | Insufficient alone — doesn't validate the tokens the contractor population and partner integration will present | Insufficient alone — doesn't stop generic web-layer attacks like injection against any request body fields | Correct fit — the finance API is about to face both external partner traffic (web-attack surface) and token-based access (API-semantic surface) |

### Decision 2: Manual Pre-Release Pen Test vs. Automated SAST/DAST in CI/CD vs. Runtime-Only Protection

| Factor | Manual Pen Test Before Release | Automated SAST/DAST in CI/CD (cross-link [AZ-400](../AZ-400/README.md)) | Runtime-Only Protection (WAF/APIM catches issues in production, no pre-release testing) |
|---|---|---|---|
| **When vulnerabilities are caught** | Once, right before release — misses anything introduced afterward until the next scheduled test | Every commit/build — a SQL injection introduced today is flagged before it merges, not months later at the next pen test cycle | Only after the vulnerable code is already live — the WAF/gateway may block *some* exploit attempts, but the vulnerability itself ships |
| **Cost per finding** | High — manual tester time, usually contracted, and findings arrive late enough that fixes are expensive (already in production or close to it) | Low per finding — automated scans are cheap to run repeatedly; a finding caught in CI costs a code review comment, not a hotfix | Lowest visible cost, highest hidden cost — a runtime-only strategy means the org's actual test environment is production traffic |
| **Coverage cadence given a partner integration deadline** | One-time snapshot — doesn't scale to the ongoing pace of API changes once the partner integration is live and iterating | Continuous — matches the pace of ongoing development after go-live, not just the one deadline | No pre-release coverage at all |
| **Fit for this scenario** | Insufficient alone — the org needs recurring assurance as the API keeps changing after the partner goes live, not a single point-in-time check | Best fit — catches issues before Lab 6's network controls or this lab's gateway ever see the traffic, and scales with ongoing development | Poor fit as the sole strategy — leaves the org's most sensitive API to be pen-tested by real attackers |

### Recommendation for This Scenario
**Layered enforcement**: Application Gateway WAF (or Front Door WAF, per AZ-305 Lab 5's regional-vs-global guidance) in front of Azure API Management, with APIM enforcing JWT validation, per-subscription rate limiting, and response shaping. **Automated SAST/DAST integrated into the CI/CD pipeline** (the AZ-400 discipline) as the primary secure-SDLC control, with a manual pen test retained only as a periodic deeper assurance check before major releases — not the sole gate. Runtime protection (WAF + APIM) is the safety net, never the only line of defense. Part 2–3 implement the APIM half of this design.

---

## Part 2: Deploy API Management with JWT Validation and Rate Limiting

### Step 1: Deploy the APIM Instance (Consumption Tier — No Idle Cost)
```bash
az group create --name sc100-lab7-rg --location eastus

az apim create \
  --resource-group sc100-lab7-rg \
  --name sc100-lab7-apim \
  --publisher-name "SC-100 Lab" \
  --publisher-email "security-architecture@<your-domain>.com" \
  --sku-name Consumption
```
Consumption tier bills per call with no hourly minimum — appropriate for a lab, and for a real low-to-moderate-traffic API where the Developer/Standard tier's fixed hourly cost isn't yet justified.

### Step 2: Import the Finance API
```bash
az apim api create \
  --resource-group sc100-lab7-rg --service-name sc100-lab7-apim \
  --api-id finance-api --path finance \
  --display-name "Finance API" \
  --protocols https \
  --service-url "https://<backend-finance-api-hostname>"
```

### Step 3: Apply a JWT Validation Policy
```bash
cat > jwt-validation-policy.xml << 'EOF'
<policies>
  <inbound>
    <base />
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401" require-scheme="Bearer">
      <openid-config url="https://login.microsoftonline.com/<tenant-id>/v2.0/.well-known/openid-configuration" />
      <audiences>
        <audience>api://finance-api</audience>
      </audiences>
      <required-claims>
        <claim name="roles" match="any">
          <value>FinanceApi.Read</value>
        </claim>
      </required-claims>
    </validate-jwt>
  </inbound>
</policies>
EOF

az apim api policy create \
  --resource-group sc100-lab7-rg --service-name sc100-lab7-apim \
  --api-id finance-api --policy-format xml \
  --value @jwt-validation-policy.xml
```
This is the API-semantic control a WAF structurally cannot provide — it rejects any request whose token isn't issued by the org's own Entra ID tenant, isn't scoped to this specific API, or lacks the required `FinanceApi.Read` role claim, before the request ever reaches the backend.

### Step 4: Apply a Rate-Limiting Policy
```bash
cat > rate-limit-policy.xml << 'EOF'
<policies>
  <inbound>
    <base />
    <rate-limit-by-key calls="60" renewal-period="60"
      counter-key="@(context.Subscription.Id)" />
  </inbound>
</policies>
EOF

az apim api policy create \
  --resource-group sc100-lab7-rg --service-name sc100-lab7-apim \
  --api-id finance-api --policy-format xml \
  --value @rate-limit-policy.xml
```
Rate limiting per subscription key means the contractor population from Lab 2 and the future partner integration each get their own budget — one misbehaving or compromised caller can't exhaust capacity for everyone else.

**Validation checkpoint**:
```bash
az apim api policy show \
  --resource-group sc100-lab7-rg --service-name sc100-lab7-apim --api-id finance-api \
  --query "value" -o tsv
```
Confirm both `validate-jwt` and `rate-limit-by-key` appear in the returned policy XML.

---

## Part 3: Validate the Enforcement Behavior

### Step 1: Confirm an Unauthenticated Request Is Rejected
```bash
GATEWAY_URL=$(az apim show --resource-group sc100-lab7-rg --name sc100-lab7-apim --query gatewayUrl -o tsv)

curl -s -o /dev/null -w "%{http_code}\n" "$GATEWAY_URL/finance/accounts"
```
**Expected result**: `401`. No `Authorization` header was sent, so the `validate-jwt` policy from Part 2 Step 3 rejects the request before it reaches the finance API backend — proving the API-semantic control works independently of whatever the WAF layer would have allowed through.

### Step 2: Confirm Rate Limiting Triggers
```bash
for i in $(seq 1 65); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Ocp-Apim-Subscription-Key: <a-valid-apim-subscription-key>" \
    "$GATEWAY_URL/finance/accounts"
done | sort | uniq -c
```
**Expected result**: the first 60 requests within the 60-second window return whatever status a valid authenticated request returns (`200` or a downstream code); requests 61–65 return `429 Too Many Requests` — the rate-limit policy's `calls=60, renewal-period=60` from Part 2 Step 4 enforcing exactly as configured.

---

## Part 4: Add Response Shaping to Prevent Excessive Data Exposure

Decision 1 flagged "excessive data exposure" as an API-specific risk a WAF structurally can't catch — this closes that specific gap. The finance API's backend returns full account records, but not every caller should see every field (a contractor with `FinanceApi.Read` shouldn't necessarily see the same fields a full-time finance employee sees).

### Step 1: Apply a Response-Shaping Policy Based on the Caller's Claims
```bash
cat > response-shaping-policy.xml << 'EOF'
<policies>
  <inbound><base /></inbound>
  <outbound>
    <base />
    <choose>
      <when condition="@(!context.User.Groups.Any(g => g.Name == "FinanceFullAccess"))">
        <set-body>@{
          var body = context.Response.Body.As<JObject>();
          body.Property("ssn")?.Remove();
          body.Property("fullAccountNumber")?.Remove();
          return body.ToString();
        }</set-body>
      </when>
    </choose>
  </outbound>
</policies>
EOF

az apim api policy create \
  --resource-group sc100-lab7-rg --service-name sc100-lab7-apim \
  --api-id finance-api --policy-format xml \
  --value @response-shaping-policy.xml
```
This is enforced at the gateway, in front of every caller, rather than trusting the backend application (or every future backend change) to remember the field-masking rule correctly — the same "enforce once, centrally" principle Lab 2's Conditional Access layering and Lab 6's Private Access design both apply to their respective layers.

### Step 2: Validate the Masking Behavior
```bash
curl -s -H "Authorization: Bearer <contractor-scoped-token>" \
  "$GATEWAY_URL/finance/accounts" | grep -o '"ssn"' || echo "ssn field correctly masked"
```
**Expected result**: `ssn field correctly masked` for a contractor-scoped token; a token carrying the `FinanceFullAccess` group claim should still see the full record. This is the technical proof that "least privilege" from Zero Trust (Lab 1) applies at the data-field level, not just at the API-access level.

---

## Cleanup
```bash
az group delete --name sc100-lab7-rg --yes --no-wait
```
Confirm with `az apim list --resource-group sc100-lab7-rg -o table` — expect a "ResourceGroupNotFound" error once deletion completes. Consumption-tier APIM has no idle cost, but deleting it removes the resource and its public gateway URL, which is good hygiene regardless.

---

## What You Practiced

| Concept | Why It Matters |
|---|---|
| WAF vs. API gateway policies as complementary layers | Same pattern as Lab 6's SSE-vs-hub-spoke decision — recognizing "both, for different threats" instead of picking one |
| JWT validation at the gateway | The concrete mechanism that makes Lab 2's Conditional Access-issued tokens actually mean something at the API layer |
| Per-subscription rate limiting | Directly maps to the multi-tenant access pattern (contractors, partners) most SC-100 API scenarios describe |
| Shift-left secure SDLC (SAST/DAST in CI/CD) | The highest-leverage secure-SDLC control — catches issues before any runtime control (WAF, APIM, network) ever has to |
| Recognizing runtime protection as a safety net, not a strategy | A recurring SC-100 theme: production traffic is not an acceptable test environment |

---

## Common Mistakes to Avoid
- **Deploying a WAF and calling API security "done"**: a WAF doesn't validate tokens, scopes, or claims — it has no concept of API semantics, only web-attack signatures
- **Deploying APIM policies without a WAF in front of a public-facing API**: APIM's policies don't replace signature-based protection against generic web attacks like SQL injection in a request body
- **Treating a pre-release pen test as sufficient ongoing assurance**: it's a point-in-time snapshot; anything shipped after that snapshot is untested until the next cycle
- **Rate limiting by a global counter instead of per-subscription/per-key**: a global limit means one caller's abuse degrades service for every legitimate caller, including the ones the API was built for
- **Validating JWT issuer/audience but skipping required claims/scopes**: a token can be genuinely issued by the tenant and still be the wrong token for this API — claim/scope validation is what stops a valid token for one API being replayed against another

---

## Next Steps
- Continue to [Lab 8: Data Security Architecture Design (Capstone)](lab-8-data-security-architecture-design.md) to design protection for the finance data this API actually serves, and tie every prior lab's design into one Zero Trust architecture
- For the CI/CD pipeline security (SAST/DAST integration, secure release gates) this lab's secure SDLC decision assumes, see AZ-400 (Azure DevOps Engineer Expert) — not yet published in this repo, but the natural home for that operational depth
- For the network access layer this API sits behind, see [Lab 6: Network Security Architecture Design](lab-6-network-security-architecture-design.md) in this track
