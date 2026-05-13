<h1 align="center">Eunomia</h1>

<p align="center">
  <em>Governed natural-language access to the data warehouse — for LLM-driven analytics that don't leak.</em>
</p>

<p align="center">
  <a href="https://github.com/anuj81/eunomia-middleware"><img alt="middleware" src="https://img.shields.io/badge/eunomia--middleware-blue?logo=github"></a>
  <a href="https://github.com/anuj81/eunomia-rag"><img alt="rag" src="https://img.shields.io/badge/eunomia--rag-blue?logo=github"></a>
  <a href="https://github.com/anuj81/eunomia-cli"><img alt="cli" src="https://img.shields.io/badge/eunomia--cli-blue?logo=github"></a>
  <a href="https://github.com/anuj81/eunomia-infrastructure"><img alt="infra" src="https://img.shields.io/badge/eunomia--infrastructure-blue?logo=github"></a>
  <a href="https://github.com/anuj81/eunomia-middleware/blob/main/LICENSE"><img alt="License" src="https://img.shields.io/badge/License-Apache_2.0-blue.svg"></a>
</p>

> *In Greek mythology, **Eunomia** (Εὐνομία) is the goddess of law, lawful conduct, and good order — one of the Horae, daughters of Zeus and Themis. From* eu *(good) +* nomos *(law).*
>
> *The project borrows that spirit: ordered, lawful governance of who can ask what of the data — deterministically enforced by code, not by hope.*

---

## Contents

- [The problem](#the-problem)
- [What Eunomia does](#what-eunomia-does)
- [How it differs](#how-it-differs-from-typical-llm-to-database-tooling)
- [Architecture](#architecture)
- [The query flow, end to end](#the-query-flow-end-to-end)
- [How access is governed](#how-access-is-governed)
- [Audit](#audit)
- [Running the stack](#running-the-stack)
- [The four repositories](#the-four-repositories)
- [Worked example: the killer demo](#worked-example-the-killer-demo)
- [Status, limitations, roadmap](#status-limitations-roadmap)
- [About the author](#about-the-author)

---

## The problem

Pointing an LLM at a data warehouse is dangerously easy. A natural-language query goes in, SQL comes out, the SQL runs, results come back. The model is helpful, fast, and increasingly accurate.

It is also, in any honest threat model, a security catastrophe waiting to happen:

- **The model doesn't *know* it's leaking PII.** It produces well-formed SQL that touches whichever tables it was told about. If a system prompt says "don't show emails," the model usually complies — until it doesn't, because a clever question, a long context, or a poorly-worded instruction tipped it over.
- **"Best effort" hiding is not a control.** Auditors, regulators, and your own security team don't accept "the prompt said please be careful." They want deterministic enforcement of who can see what.
- **The catalog and the runtime are decoupled.** Most LLM-on-warehouse tools have rules in the LLM's prompt; the data catalog (OpenMetadata, DataHub, Atlan, etc.) sits off to the side with the *real* ownership of "what does this column mean" and "what is sensitive." That gap is where things go wrong.
- **Identity is usually faked.** Demo apps hard-code a user role. Real systems need OIDC, real JWT validation, and the trust line all the way down to the database query.
- **There's no audit trail worth showing a regulator.** Logs say "user X ran a query." They don't say *which policy decision granted access*, *which columns were considered sensitive*, *whether the answer was masked or not*, and *which downstream policy was overridden*.

Eunomia is built on the position that an LLM is a very good *suggestion engine* for SQL — and that the right way to use it safely is to wrap it in deterministic, identity-aware governance.

---

## What Eunomia does

A user authenticates against an identity provider (Keycloak today). They send a natural-language question. The middleware:

1. **Verifies the identity** — real OIDC JWT, signature checked against the realm's JWKS.
2. **Resolves authorization in the catalog**, not in code — OpenMetadata's tag policies decide what views this user's roles may see.
3. **Optionally ranks the allowed views by relevance** to the question (Qdrant + sentence-transformers) so the LLM only sees the most relevant top-K in its prompt — but the security boundary is the *full* allowed set.
4. **Asks an LLM to write SQL** (Google Gemini today; swappable).
5. **Validates the generated SQL** at the AST level using `sqlglot`, against the OM-allowed view list. If the LLM hallucinates a base table or any other unauthorized reference, the validator rejects it; the rejection is fed back as context, and the LLM is asked to retry — typically self-correcting within one attempt.
6. **Executes the validated SQL** on MySQL.
7. **Masks PII columns** based on OpenMetadata's `PII.Sensitive` tags, unless the user's Keycloak roles include `eunomia-pii-unmask`.
8. **Streams progress + results back over Server-Sent Events**.
9. **Records a per-request audit trail** to two side-by-side files — one human-readable, one newline-delimited JSON for SIEM ingestion.

Every decision in that chain is made by a named component with a single responsibility. The LLM, deliberately, doesn't get to make a single security-relevant call.

---

## How it differs from typical LLM-to-database tooling

| Concern | Typical LLM-to-DB tool | Eunomia |
|---|---|---|
| **Identity** | "Pass an API key" or a hard-coded user | Real OIDC JWT (Keycloak), Device Code flow for CLI, signature + issuer + expiry verified per request |
| **Authorization** | Hardcoded in app config or prompt instructions | OpenMetadata **tag policies** — the catalog is the policy decision point |
| **Schema visibility** | Send everything in the prompt; hope the LLM filters | OM returns the user's exact allowed view set; the prompt sees only those (further narrowed by relevance via RAG) |
| **SQL safety** | Maybe a regex check; usually just trust the LLM | `sqlglot` AST validation against the OM-allow list; rejection → retry-with-error-feedback loop |
| **PII handling** | LLM is told "don't show emails"; sometimes works | Post-execution row-level masking using OM PII tags + the user's Keycloak `eunomia-pii-unmask` role |
| **Catalog freshness** | The model sees a fixed schema dump or a hand-curated YAML | An indexer pulls from OM via a Keycloak service account; per-tag retrieval at request time |
| **Audit** | App logs `user → query → result count` | Per-request trail: `request_id`, sub, email, roles, auth provider, allowed_views, executed_sql, validation_attempts, pii_columns_masked, status, duration — both human-readable and JSON-lines |
| **Verification** | "Trust me, I tested it" | A 30-case end-to-end harness that asserts the whole identity → authz → SQL → masking chain against the live stack |
| **The trust line** | Implicit, prompt-shaped | Explicit, code-shaped, drawn deliberately between named components |

Eunomia is not trying to be the fastest LLM-to-SQL tool, the cleverest semantic layer, or a better OpenMetadata. It is trying to be the thing you can point at an auditor and say *"here is the governance contract, and here is the test that proves it holds."*

---

## Architecture

Eunomia is split into four cooperating repositories, each owning a single responsibility:

```
┌──────────────────────────┐   OIDC Device Code     ┌────────────────────────────┐
│       eunomia-cli        │ ─────────────────────▶ │           Keycloak         │
│  (user-facing terminal)  │  ◀───── JWT ─────────  │   (Identity Provider)      │
└──────────┬───────────────┘                        └────────────────────────────┘
           │                                                      ▲
           │  Bearer JWT                                          │ JWKS
           ▼                                                      │
┌──────────────────────────┐                        ┌─────────────┴──────────────┐
│   eunomia-middleware     │ ──── role's tag ─────▶ │       OpenMetadata         │
│  (governance enforcer)   │ ◀── allowed views ──── │  (catalog + policy DPP)    │
└──────────┬───────┬───────┘                        └────────────────────────────┘
           │       │                                              ▲
   query + │       │ Bearer JWT (for fetching                     │ service-account
   allowed │       │   view metadata + PII tags)                  │ JWT (indexer)
     views │       │                                              │
           ▼       │                                              │
┌──────────────────────────┐                        ┌─────────────┴──────────────┐
│       eunomia-rag        │ ◀──── catalog pull ─── │      eunomia-rag           │
│  (relevance + Qdrant)    │                        │     (indexer, cron)        │
└──────────┬───────────────┘                        └────────────────────────────┘
           │
   top-K   │
           ▼
┌──────────────────────────┐                        ┌────────────────────────────┐
│    LLM (Gemini)          │ ◀── prompt ─────────── │   eunomia-middleware       │
│                          │ ──── SQL ────────────▶ │    (continues below)       │
└──────────────────────────┘                        └─────────────┬──────────────┘
                                                                  │
                                  sqlglot AST validation  ◀───────┤
                                  against FULL allowed set        │
                                                                  ▼
                                                    ┌────────────────────────────┐
                                                    │           MySQL            │
                                                    └─────────────┬──────────────┘
                                                                  │ raw rows
                                                                  ▼
                                                    ┌────────────────────────────┐
                                                    │  PII masker (OM tags +     │
                                                    │   eunomia-pii-unmask role) │
                                                    └─────────────┬──────────────┘
                                                                  │ safe rows
                                                                  ▼
                                                    ┌────────────────────────────┐
                                                    │   SSE stream → eunomia-cli │
                                                    │   audit record → logs/     │
                                                    └────────────────────────────┘
```

**Trust line in one sentence**: identity comes from Keycloak, view authorization comes from OpenMetadata, query relevance comes from Qdrant, SQL safety comes from `sqlglot`, PII masking comes from OM tags + a Keycloak role — the middleware is the orchestrator that wires them together and the LLM is the suggestion engine.

---

## The query flow, end to end

```mermaid
sequenceDiagram
    autonumber
    participant C as eunomia-cli
    participant KC as Keycloak
    participant M as eunomia-middleware
    participant OM as OpenMetadata
    participant RAG as eunomia-rag
    participant LLM as Gemini
    participant DB as MySQL
    participant AUD as Audit Sinks

    rect rgba(255, 250, 230, 0.7)
    Note over C, KC: Once per session — Device Code flow
    C->>KC: POST /auth/device (client_id=eunomia-cli)
    KC-->>C: device_code, user_code, verification_uri
    Note over C: User approves in browser
    C->>KC: poll POST /token
    KC-->>C: access_token (JWT) + refresh_token
    Note over C: Cached at ~/.eunomia/cli.json (mode 0600)
    end

    C->>M: POST /v1/execute_nlq + Bearer JWT + query

    rect rgba(245, 235, 255, 0.7)
    Note over M, KC: Validate inbound JWT
    M->>KC: GET /.well-known/openid-configuration + JWKS (cached)
    KC-->>M: JWKS
    M->>M: Verify signature, iss, exp; extract sub, email, roles
    end

    rect rgba(230, 240, 255, 0.7)
    Note over M, OM: OM is the policy decision point
    M->>M: role → access tag (settings.authz.role_to_access_tag)
    M->>OM: GET /search/query?q=tags.tagFQN:"..."  (Bearer = user JWT)
    OM-->>M: Allowed view names per tag policy
    M->>OM: GET /tables/name/{fqn}?fields=columns,tags  (per allowed name)
    OM-->>M: Rich payload + PII tags
    M-->>AUD: record.allowed_views, pii_columns
    end

    rect rgba(240, 255, 240, 0.7)
    Note over M, RAG: Relevance ranking — never widens authorization
    M->>RAG: POST /v1/retrieve { query, allowed_views, k }
    RAG-->>M: Top-K ranked subset
    end

    rect rgba(245, 240, 255, 0.7)
    Note over M, LLM: Constrained generation + AST validation
    loop Up to llm.max_retries
        M->>LLM: Prompt (top-K only)
        LLM-->>M: Generated SQL
        M->>M: sqlglot AST validate against FULL allowed_views
        alt invalid
            M->>LLM: error context (next iteration)
        else valid
            Note over M: break
        end
    end
    M-->>AUD: record.executed_sql, validation_attempts
    end

    rect rgba(255, 240, 240, 0.7)
    Note over M, DB: Execute + post-process
    M->>DB: Execute validated SQL
    DB-->>M: Raw rows
    M->>M: mask_pii (unmask iff JWT carries eunomia-pii-unmask)
    M-->>AUD: record.rows_returned, pii_columns_masked, status
    end

    M-->>C: SSE stream + final { results, executed_sql, request_id }
```

Read this once and you've got the trust contract. Every box and every arrow corresponds to code you can read.

---

## How access is governed

This is the part that matters. The flow above lists the steps; this section explains the *decision authority* at each one.

### 1. Identity authority: Keycloak

A user has an identity in Keycloak. The `eunomia` realm contains roles, clients, and users. A login (Device Code in the CLI; password grant in tests) returns a JWT whose `realm_access.roles[]` claim is the source of truth for who the user is.

Test users seeded by the realm export:

| Username | Realm roles | What they get |
|---|---|---|
| `finance.alice` | `eunomia-finance-user`, `eunomia-pii-unmask` | Finance views, PII visible |
| `auditor.bob` | `eunomia-external-auditor` | Payment-history view only, PII redacted |
| `marketing.carol` | `eunomia-marketing-lead`, `eunomia-pii-unmask` | Marketing views, PII visible |
| `agency.dave` | `eunomia-agency-partner` | Agency-safe regional view only |
| `om.admin` | `eunomia-om-admin`, `eunomia-pii-unmask` | Full catalog incl. gold base tables |

The `eunomia-pii-unmask` role is **compositional** — independent of the view-access role. Hold it and PII is visible; lack it and the middleware masks PII columns post-execution.

### 2. Policy authority: OpenMetadata

This is the design decision that anchors everything.

Each view in OpenMetadata is tagged with `eunomia-access.<role>` for each role that may see it. The catalog ships with a tag classification, four access tags, four deny-by-tag policies, and four Group teams whose `defaultRoles` bind the policies to their members:

```
Policy: policy-finance-user
  Rule: deny ViewAll if !matchAnyTag('eunomia-access.finance-user')
```

When the middleware queries `OM /api/v1/search/query?q=tags.tagFQN:"eunomia-access.finance-user"`, OM evaluates the tag policies and returns only the views that policy authorizes. The middleware **does not** decide what the user can see — it asks the catalog. To change the access matrix you tag (or untag) a view in OM; no middleware deploy required.

The middleware-side YAML (`settings.authz.role_to_access_tag`) holds **only the naming convention** mapping JWT role names to OM tag FQNs — not the policy itself.

### 3. SQL safety: `sqlglot` AST validation

The LLM is asked to write a MySQL SELECT. Before the SQL ever touches the database, the validator:

- Parses the SQL with `sqlglot` (MySQL dialect)
- Rejects: anything other than a single SELECT, multiple statements, references to base tables, references to any view *not in the OM-allowed list*

The defense-in-depth twist: the validator uses the **full** OM-allowed list, not the RAG-narrowed top-K. If the LLM picks a view that was authorized by OM but didn't make the relevance top-K (cheap to imagine when `top_k` is small), the validator still accepts. The relevance layer is for context budget, not for security.

### 4. Result safety: PII masker

After execution, the result rows are post-processed. The masker:

- Asks OM (via the RAG indexer's earlier pull) which columns are tagged `PII.Sensitive`
- Checks the user's JWT for the `eunomia-pii-unmask` role
- If the role is absent, replaces values in PII columns with `"***"`; otherwise passes through unchanged

The PII tag info travels along with the catalog data — not via a separate Python dict somewhere in middleware code.

### 5. The hard limit: admin bypass

A single role, `eunomia-om-admin`, short-circuits tag filtering and returns the entire catalog (including gold base tables). It is intended for catalog-admin operators only, and its use is audited like any other request. The seeded user `om.admin` carries this role.

---

## Audit

Every NLQ produces one record across two side-by-side files. The trail is captured *regardless of outcome* — so when something fails (LLM upstream error, validator rejection, MySQL outage), you still see *which user asked what, against which views, with which roles*.

```
logs/audit.log    (one human-readable line per request)
```
```
2026-05-13T04:24:27.465+00:00 | id=4804ccc4 | user=auditor.bob roles=[eunomia-external-auditor] provider=keycloak | query='show me the payment history' | allowed=1 prompt_top_k=1 | sql_attempts=1 rows=2 | masked_pii=card_last_four,email,first_name,last_name unmask=False | dur_ms=508 status=OK
```

```
logs/audit.jsonl   (newline-delimited JSON, SIEM-friendly)
```
```json
{
  "request_id": "4804ccc4-1470-40f5-b0a0-24e68e77d093",
  "ts_request": "2026-05-13T04:24:27.465+00:00",
  "duration_ms": 508,
  "sub": "1f60e5f7-...",
  "email": "auditor.bob@open-metadata.org",
  "preferred_username": "auditor.bob",
  "roles": ["eunomia-external-auditor"],
  "auth_provider": "keycloak",
  "unmask_pii": false,
  "admin_bypass": false,
  "query": "show me the payment history",
  "allowed_views": ["finance_customer_payment_history_view"],
  "relevant_views": ["finance_customer_payment_history_view"],
  "pii_columns": {
    "finance_customer_payment_history_view": ["first_name", "last_name", "email", "card_last_four"]
  },
  "executed_sql": "SELECT * FROM finance_customer_payment_history_view LIMIT 10;",
  "validation_attempts": 1,
  "validation_errors": [],
  "rows_returned": 2,
  "pii_columns_masked": ["card_last_four", "email", "first_name", "last_name"],
  "status": "ok"
}
```

The `request_id` is returned to the client in the final SSE event so a user-facing ticket can link straight to the trust trail.

---

## Running the stack

The four repos compose like this:

```
eunomia-infrastructure   (docker-compose)
  ├── Keycloak               ← identity
  ├── OpenMetadata           ← catalog + policy decision point
  ├── OM-MySQL + Elasticsearch  ← OM's own backing stores
  │
eunomia-rag                  (FastAPI :9000)
  ├── Qdrant (docker-compose) ← vector store
  ├── indexer                 ← pulls OM catalog via Keycloak service account
  └── /v1/retrieve            ← top-K ranking
  │
eunomia-middleware           (FastAPI :8000)
  ├── auth + JWKS cache       ← real JWT verification
  ├── om_access resolver      ← role → OM tag → /search/query
  ├── ComposedCatalog         ← OM + RAG composition
  ├── sqlglot validator       ← AST safety
  ├── MySQL execution         ← warehouse on :3306
  ├── PII masker
  └── audit log               ← dual sink
  │
eunomia-cli                  (terminal)
  ├── login   (Device Code → Keycloak)
  ├── ask     (NLQ → middleware SSE)
  └── admin   (reindex-rag, whoami-om, runbooks)
```

### Quickstart

Assuming Docker is running and the local analytics MySQL (`:3306`) has the seeded `zenith_corp_eunomia` views:

```bash
# 1. Bring up Keycloak + OpenMetadata + Elasticsearch + OM-MySQL
git clone git@github.com:anuj81/eunomia-infrastructure.git
cd eunomia-infrastructure && docker-compose up -d

# 2. Bring up Qdrant
git clone git@github.com:anuj81/eunomia-rag.git
cd eunomia-rag && docker-compose up -d qdrant
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env  # fill in KEYCLOAK_RAG_INDEXER_SECRET, RAG_API_KEY

# 3. Seed OM with tag policies + role-teams
cd ../
git clone git@github.com:anuj81/eunomia-middleware.git
cd eunomia-middleware
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python seed_openmetadata.py        # database + schema + views (one-time)
python seed_om_policies.py         # tag classification, deny policies, teams

# 4. Index the catalog
cd ../eunomia-rag && ./venv/bin/python -m src.indexer --reset

# 5. Start the RAG service
./venv/bin/python -m src.main &

# 6. Start the middleware
cd ../eunomia-middleware
cp .env.example .env  # set GEMINI_API_KEY etc.
EUNOMIA_AUTH__PROVIDER=keycloak \
EUNOMIA_OPENMETADATA__MOCK=false \
EUNOMIA_RAG__MOCK=false \
./start-fastapi.sh

# 7. Verify the whole chain — 30 cases
cd ../eunomia-infrastructure
python verify_phase_d.py
# → TOTAL: 30/30  PASS

# 8. Use the CLI
cd ../
git clone git@github.com:anuj81/eunomia-cli.git
cd eunomia-cli
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
./venv/bin/python -m src.main login
./venv/bin/python -m src.main ask "what is our daily revenue last week"
```

### Offline / mock mode

Hacking without Keycloak/OM/MySQL up? The middleware ships with mock providers that preserve every code path:

```bash
EUNOMIA_AUTH__PROVIDER=mock           # accept string-match dev tokens
EUNOMIA_OPENMETADATA__MOCK=true       # in-memory fixture catalog
EUNOMIA_RAG__ENABLED=false            # skip relevance narrowing entirely
./start-fastapi.sh
```

Use `finance-token`, `external-auditor-token`, `marketing-token`, `agency-token`, or `om-admin-token` as the Bearer.

---

## The four repositories

| Repository | Responsibility | Live ports |
|---|---|---|
| **[eunomia-middleware](https://github.com/anuj81/eunomia-middleware)** | Governance engine. JWT verify, OM tag-policy resolution, RAG composition, LLM call, SQL validation, MySQL execution, PII masking, audit. 117 tests. | `:8000` |
| **[eunomia-rag](https://github.com/anuj81/eunomia-rag)** | Catalog-aware relevance ranker. FastAPI + Qdrant + sentence-transformers. Indexer pulls from OM via a Keycloak service account. | `:9000` + Qdrant `:6333` |
| **[eunomia-cli](https://github.com/anuj81/eunomia-cli)** | Terminal client. OIDC Device Code login, token cache (mode 0600), refresh-on-401, admin commands. 40 tests. | — |
| **[eunomia-infrastructure](https://github.com/anuj81/eunomia-infrastructure)** | Docker-compose for Keycloak + OpenMetadata + Elasticsearch + OM-MySQL. Realm export, OM OIDC env, end-to-end verification harness (30 cases). | Keycloak `:8080`, OM `:8585` |

Each repo has its own README walking through layout, configuration, and tests.

---

## Worked example: the killer demo

The agency partner role (`eunomia-agency-partner`) is authorized to see exactly one view: `marketing_regional_performance_view`. They have no PII unmask role.

What happens when they explicitly ask for the gold base table?

```
$ eunomia-cli ask "give me data from core.dim_customers"

> Authenticating & Fetching Roles...
> Found 1 Allowed Views...
> Generating SQL (Attempt 1)...
```

Behind the scenes the middleware log shows:

```
NLQ received | id=… user=agency.dave roles=['eunomia-agency-partner'] query='give me data from core.dim_customers'
Generated SQL (attempt 1):
  SELECT * FROM core.dim_customers
WARNING | SQL validation rejected attempt 1: Unauthorized table/view reference: dim_customers.
         Allowed views: marketing_regional_performance_view
Generated SQL (attempt 2):
  SELECT * FROM marketing_regional_performance_view
NLQ complete | rows=4
```

The LLM **literally wrote what the user asked for** — `SELECT * FROM core.dim_customers`. The validator rejected it. The retry loop passed the error back to the LLM. The LLM, now knowing the constraint, picked the one view it actually had access to. The end-user got the regional rollup — no data from `core.dim_customers` ever touched the wire.

This is the entire governance contract working in production. The LLM is naive, the validator is strict, and the user gets a useful answer scoped exactly to what their role grants.

---

## Status, limitations, roadmap

**Live and verified end-to-end** (Phase D):
- Keycloak OIDC identity (Device Code + service-account flows)
- OM `custom-oidc` mode with tag-policy enforcement
- RAG indexer via Keycloak service-account auth
- Middleware: real JWT validation, OM-tag-search resolver, RAG composition, sqlglot validation, MySQL execution, PII masking
- Audit trail (human + JSONL)
- CLI Device Code login + token cache + auto-refresh
- 30-case end-to-end verification harness passes against the live stack

**Known limitations** (called out so they don't surprise anyone):
- OM 1.12's `/tables` list endpoint does *not* filter by per-entity policy — that's a documented OM behavior. Eunomia uses `/search/query` with the tag filter, which respects tag policies by construction.
- OpenMetadata persists runtime auth config in its database, not the env file. Switching providers in-place requires updating the `openmetadata_settings` table directly (a one-line SQL UPDATE; documented in `eunomia-infrastructure/README.md`).
- The dev `realm-export.json` contains test passwords (`test`) and client secrets — production deployments need real secret management and disabled Direct Access Grant.
- Today the LLM is Gemini; the LLM client is a thin async wrapper, swappable in a few lines for any provider that accepts a prompt and returns a string.

**Roadmap candidates** (not committed to a phase):
- MCP adapter so Eunomia can be invoked as a tool from an agentic-AI host
- Real-time OM-webhook → RAG push sync (today: cron / admin-trigger)
- Bulk per-entity permission probe to scale catalog enumeration past a few hundred views per role
- A web UI on top of the same SSE endpoint (companion `eunomia-ui` repo, not yet built)

---

## About the author

I'm Anuj. I enjoy solving complex problems with technology — and Eunomia is the most recent of those, scratching a specific itch about the gap between "LLMs are useful at SQL" and "we cannot safely point them at the warehouse without governance code." Beyond engineering I read about human psychology and philosophy — how people actually think, what makes systems work or fail when humans are in them.

Outside of code I cycle 🚴, do woodworking 🪵, read philosophy and behavioural science 📚, and try to be a good dad 👨‍👧.

> We're all here for a short time, so don't think too much.
> Enjoy what you work on. Work on what you enjoy. :)

**Current focus**: cracking the agentic-AI world — building useful agents that don't accidentally do unsafe things.

---

<p align="center"><sub>The project name is Eunomia — Greek goddess of law, lawful conduct, and good order. The aspiration is the same: orderly, lawful access to data by humans helped by machines.</sub></p>
