# Loan Origination API — Postman Onboarding

This repo contains the OpenAPI spec and Postman onboarding workflow for the **Loan Origination Service** (`loan-origination-api`). It is part of a Postman CSE implementation exercise demonstrating automated API catalog onboarding for a financial services customer.

This service was chosen as the second onboarding target specifically because it differs from the Payment Refund API in two meaningful ways: **different infrastructure** (ECS behind an Application Load Balancer, vs Lambda behind API Gateway) and **different auth patterns** (mTLS for service-to-service, vs OAuth 2.0). This demonstrates that the onboarding pattern generalizes across service types.

---

## What Changed vs the Payment Refund API Workflow

| Input | Payment Refund API | Loan Origination API | Why It Changed |
|---|---|---|---|
| `project-name` | `payment-refund-api` | `loan-origination-api` | Service identity |
| `domain` | `payments` | `lending` | Business domain |
| `domain-code` | `PAY` | `LEN` | Workspace naming prefix |
| `spec-url` | Points to payment spec | Points to loan spec | Different spec file in this repo |
| `env-runtime-urls-json` | API Gateway endpoints | ECS/ALB endpoints | Different infrastructure pattern |
| `governance-mapping-json` | `"payments":"Payments Engineering"` | `"lending":"Lending Engineering"` | Different governance group |

**What stayed the same:** the action reference, trigger, permissions block, all four secret references, `generate-ci-workflow: false`, and the overall workflow structure. This is intentional — the onboarding pattern is parameterized, not reimplemented per service.

---

## What This Workflow Does

The workflow at `.github/workflows/postman-onboard.yml` uses [`postman-cs/postman-api-onboarding-action@v0`](https://github.com/postman-cs/postman-api-onboarding-action) — a composite GitHub Action that chains two sub-actions:

1. **Bootstrap** (`postman-cs/postman-bootstrap-action@v0`):
   - Creates a Postman workspace named with the `domain-code` prefix (`LEN`) and `project-name`
   - Uploads the OpenAPI spec to Postman Spec Hub
   - Lints the spec using the Postman CLI
   - Generates three collections: **Baseline** (full API coverage), **Smoke** (critical path health checks), **Contract** (schema validation against the spec)
   - Injects test scripts into the generated collections
   - Assigns the workspace to the `Lending Engineering` governance group (requires valid `POSTMAN_ACCESS_TOKEN`)
   - Persists `POSTMAN_WORKSPACE_ID`, `POSTMAN_SPEC_UID`, and three collection UIDs as GitHub repo variables for idempotent reruns

2. **Repo Sync** (`postman-cs/postman-repo-sync-action@v0`):
   - Creates two Postman environments (`prod` and `staging`) with the base URLs configured
   - Exports all three collections as multi-file YAML under `postman/collections/`
   - Creates a mock server (from the baseline collection)
   - Creates a smoke monitor (from the smoke collection)
   - Links the Postman workspace to this GitHub repo via Bifrost (requires valid `POSTMAN_ACCESS_TOKEN`)
   - Commits all generated artifacts back to this repo

---

## Service Characteristics

| Property | Value |
|---|---|
| Service | Lending Platform API — Loan Origination Service |
| Version | 1.4.0 |
| Infrastructure | ECS behind an Application Load Balancer (ALB) |
| Service-to-Service Auth | mTLS (mutual TLS) |
| Internal Gateway Auth | JWT (Bearer token) |
| Domain | lending |
| Prod URL | `https://lending-api.example.com/v1` |
| Staging URL | `https://lending-api-staging.example.com/v1` |

---

## mTLS Auth: What the Onboarding Action Handles vs What It Doesn't

The Loan Origination service uses **mTLS** for service-to-service communication. The onboarding action creates Postman environments with `baseUrl` variables set to the runtime URLs above. It does **not** provision TLS certificates.

In a real engagement, after running this workflow, the customer's ops team would need to manually add the following as Postman environment variables (marked as **Secret** in Postman):

| Variable | Description |
|---|---|
| `mtls_client_cert` | Base64-encoded PEM client certificate issued by the internal CA |
| `mtls_client_key` | Base64-encoded PEM private key for the client certificate |
| `mtls_ca_cert` | CA certificate for validating the server's identity |

These values are stored in the customer's internal PKI/secrets manager (Vault, AWS ACM, etc.) and must be provided by the ops team. The CSE cannot provision them without access to the customer's certificate infrastructure.

Once added, collection pre-request scripts can reference these variables to configure mTLS on requests. This is documented in the handoff runbook.

---

## Infrastructure Note: ECS vs Lambda

The payment service runs on **AWS Lambda behind API Gateway** — stateless, event-driven, short-lived invocations. The loan origination service runs on **ECS behind an Application Load Balancer** — long-lived containers with persistent connections.

This infrastructure difference does not change anything in the Postman onboarding workflow itself. The action works at the API contract level (OpenAPI spec + Postman API), not at the infrastructure level. What changes in a real engagement is:
- The runtime base URLs (ALB DNS names vs API Gateway invoke URLs)
- Health check behavior (ECS target group health checks vs Lambda cold starts)
- Auth credential provisioning path (certificates from internal PKI vs OAuth tokens from the auth server)

All of this is captured in environment variables — the `env-runtime-urls-json` input and the manually-added auth variables above.

---

## How to Run the Workflow

1. Go to **Actions** tab in this repo
2. Select **"Postman API Onboarding — Loan Origination Service"**
3. Click **"Run workflow"** → **"Run workflow"**

### Prerequisites

All three secrets must be set on this repo (Settings → Secrets → Actions):

| Secret | Description |
|---|---|
| `POSTMAN_API_KEY` | Postman API key (PMAK) — long-lived, generated from Postman account settings |
| `POSTMAN_ACCESS_TOKEN` | Session token — extract via `postman login` then `cat ~/.postman/postmanrc \| jq -r '.login._profiles[].accessToken'`. **Expires — refresh before triggering.** |
| `GH_FALLBACK_TOKEN` | GitHub PAT with `repo` and `workflow` scopes |

---

## What to Validate After a Successful Run

- [ ] GitHub Actions run is green
- [ ] New workspace visible at [app.getpostman.com](https://app.getpostman.com) (named `LEN-loan-origination-api` or similar)
- [ ] Spec visible in **Spec Hub** inside the workspace
- [ ] Three collections present: Baseline, Smoke, Contract
- [ ] Two environments: `prod` and `staging` with correct base URLs
- [ ] `postman/collections/` directory committed to this repo
- [ ] GitHub repo variables set: Settings → Actions → Variables for `POSTMAN_WORKSPACE_ID` etc.
- [ ] Mock server URL in workflow run output
- [ ] Smoke monitor created

---

## Idempotent Reruns

The bootstrap action writes the created Postman asset IDs as GitHub repo variables. On rerun, the action reads these variables and skips creation steps, targeting the existing assets instead. Simply re-trigger the workflow if a run fails partway through.

---

## What's Universal vs What Changes Per Service

**Universal:**
- The action reference and workflow structure
- The `workflow_dispatch` trigger + permissions block (`actions: write`, `contents: write`, `variables: write`)
- The four secret references
- `generate-ci-workflow: false`
- Idempotency via GitHub repo variables

**Changes per service:**
- `project-name`, `domain`, `domain-code`
- `spec-url`
- `environments-json` and `env-runtime-urls-json` (service-specific tiers and gateway URLs)
- `governance-mapping-json`
- Post-run manual steps for auth-specific variables (mTLS certs, OAuth client IDs, API keys)

---

## What the Customer's Ops Team Must Provide

- Real ALB DNS names (from AWS) for prod and staging
- mTLS certificates (`mtls_client_cert`, `mtls_client_key`, `mtls_ca_cert`) from internal PKI
- Postman user IDs for workspace admin assignment
- System environment IDs for Spec Hub linking
- Permission to add secrets and the workflow file to the service repo in the customer's GitHub org

---

## AI Assistance Disclosure

This workflow file and README were generated with Claude Code (claude-sonnet-4-6) as part of the CSE exercise. The structural differences from the payment workflow were identified by comparing the two OpenAPI specs directly. The mTLS auth guidance was derived from the spec's security scheme definition and general knowledge of mTLS patterns in Postman environments, then validated for accuracy before inclusion.
