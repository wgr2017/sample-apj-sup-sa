# agentic-payments-mpp-workshop

Reference implementation for the **AI Engineer Conference Singapore 2026**
workshop (90 minutes, hands-on) on building **machine-to-machine paid
agents** with Amazon Bedrock and Stripe MPP.

## Production-ready workshop branches

Two branches hold the production-ready workshop content. The `main` branch
is an older baseline and should **not** be used to build or deploy the
workshop. Pick one of the two variants below depending on which Stripe MPP
payment method you want to teach:

| Branch | Payment method | What participants pay with | Best for |
|---|---|---|---|
| [`mpp-spt-option`](../../tree/mpp-spt-option) | **Fiat via Shared Payment Tokens (SPT)** | A shared sandbox buyer key (announced by instructors) mints a one-time SPT per call, scoped to each participant's own Stripe Profile. Each `/generate` call charges $1.00 USD against the participant's profile via card + link. | Default path for the 2026 workshop — no crypto knowledge required, participants just need a Stripe sandbox with Agentic Commerce enabled. |
| [`mpp-crypto-option`](../../tree/mpp-crypto-option) | **Crypto stablecoin on Tempo testnet** | Each participant gets an auto-generated, faucet-funded Tempo wallet. `/generate` charges ~$0.01 USDC on-chain before LLM runs. | Alternative path when you want to teach the stablecoin / on-chain side of MPP. |

Both branches deploy through the same Workshop Studio workflow
(`bash infrastructure/scripts/package_for_workshop.sh` + S3 sync + git
push to `mainline` on the WS repo). See
[`docs/workshop-owner-runbook.md`](docs/workshop-owner-runbook.md) on the
SPT branch for owner-side setup specific to that variant.

**The rest of this README describes the crypto flow** and will be
refreshed as the branches diverge further. For the authoritative overview
of either variant, check its branch's `workshop/content/introduction/index.en.md`.

---

Two agents talk over HTTP and **pay each other per call**:

- **Samurai** — a user-facing assistant that gathers product details from a
  human and orchestrates payments.
- **ListingBot** — a paid marketplace-listing generator. Each call to its
  `/generate` endpoint costs ~$0.01 USDC on the Tempo testnet, gated by
  Stripe's [Machine Payments Protocol (MPP)](https://docs.stripe.com/payments/machine/mpp).

The step-by-step participant tutorial lives in [`workshop/content/`](workshop/content/).
This README covers how the moving parts fit together so you can navigate
the code and the CloudFormation templates.

## What gets deployed

### Workshop CloudFormation (already deployed for participants)

```
infrastructure/stacks/main-stack.yaml            (root)
├── secrets-stack.yaml                           Stripe PLACEHOLDER + auto-generated Tempo wallet + MPP secret
├── mpp-logs-stack.yaml                          DynamoDB table for MPP protocol event log (per session)
├── samurai-spa-stack.yaml                         S3 + CloudFront for the SPA (empty bucket on create)
├── cognito-stack.yaml                           User Pool + Identity Pool + pre-created "participant" test user
├── listing-bot-lambda-stack.yaml                API Gateway REST + Lambda + listings DynamoDB
└── code-editor-stack.yaml                       EC2 VS Code with config.env wired to every stack output
```

### Participant CloudFormation (deployed by the participant during chapter 6)

```
workshop/overlay/workshop/code/participant/samurai-agentcore.yaml
└── Samurai AgentCore Runtime + AgentCore Memory (from a container the participant builds + pushes to ECR)
```

### Application code

```
app/listing-bot-lambda/          API Gateway Lambda (MPP gate + Bedrock Converse) — participant-editable
app/samurai-agentcore/             Strands TS container for the Samurai AgentCore Runtime — participant-editable
app/samurai-spa/                   React + Vite + Amplify SPA
```

## How Samurai and ListingBot fit together

```
Browser  ──► CloudFront / S3 (SPA)
   │
   │  Cognito USER_SRP_AUTH → ID token
   │  Cognito Identity Pool → temporary AWS credentials
   │  bedrock-agentcore:InvokeAgentRuntime (SigV4, signed in-browser)
   │
   ▼
Samurai AgentCore Runtime   ── Strands TS agent with 3 tools:
 (participant-deployed)       1. discover_service   → GET /openapi.json   (free)
                              2. check_completeness → POST /validate      (free)
                              3. generate_listing   → POST /generate      (paid via mppx/client)
                            + AgentCore Memory (short-term session)
                            + writes protocol events to DynamoDB
   │
   │  (mppx/client handles: 402 challenge → sign USDC transfer on Tempo
   │   → retry with Authorization: Payment <credential>)
   │
   ▼
ListingBot API Gateway (REST)
   │
   ▼
ListingBot Lambda
   ├─ GET  /openapi.json  → publishes per-platform input schemas + x-payment-info
   ├─ POST /validate      → deterministic rule-based check, free
   └─ POST /generate      → tiered response:
         400  malformed JSON
         422  invalid input (RFC 7807 problem, NO payment taken)
         402  MPP challenge (Stripe mints a Tempo deposit address)
         200  Bedrock Converse (Sonnet 4.6) → listing JSON
   │
   ├──► Stripe API: paymentIntents.create (crypto / Tempo deposit)
   ├──► Tempo RPC: mppx/server verifies the on-chain Transfer log
   └──► Bedrock Converse (global.anthropic.claude-sonnet-4-6)
```

Key property: the payment primitive is **orthogonal to validation**. The
Lambda validates input BEFORE minting a PaymentIntent, so a buggy caller
never gets charged for a 422.

### Payment verification (what `mppx/server` actually does)

When Samurai sends the retry with `Authorization: Payment <credential>`:

1. `mppx/server` parses the credential (pull-mode: signed transaction;
   push-mode: transaction hash).
2. It calls **Tempo RPC directly** (not Stripe) — broadcasts the signed tx
   or fetches the receipt, and verifies the `Transfer` event log matches
   the expected amount + recipient + currency.
3. If valid, the handler continues; otherwise a fresh 402 is returned.

Stripe's webhook fires asynchronously for reconciliation. It is NOT the
real-time gate. On testnet, the Lambda also calls Stripe's
`simulate_crypto_deposit` test helper so the PaymentIntent appears as
captured in the Stripe Dashboard — again, this is cosmetic, not the gate.

## Auth and data-flow details

| Actor                    | How it gets permission                                                                 |
|--------------------------|----------------------------------------------------------------------------------------|
| Human → SPA              | Amplify Auth (USER_SRP_AUTH) against the Cognito User Pool                             |
| SPA → AgentCore Runtime  | Identity Pool exchanges the Cognito ID token for temporary AWS creds. The authenticated role holds `bedrock-agentcore:InvokeAgentRuntime` scoped to this region's runtimes. |
| SPA → DynamoDB (MPP logs)| Same Identity Pool creds. The authenticated role holds `dynamodb:Query` on the MPP logs table only. |
| Samurai Runtime → Stripe secret / Tempo wallet | The runtime's execution role holds `secretsmanager:GetSecretValue` on exactly two secrets. |
| Samurai Runtime → ListingBot| Plain HTTPS to the API Gateway URL. No IAM; MPP is the only gate.                     |
| ListingBot Lambda → Stripe| Lambda execution role reads the Stripe secret from Secrets Manager on each invoke (30 s cache). |
| ListingBot Lambda → Bedrock| Lambda execution role holds `bedrock:Converse` on the Sonnet 4.6 inference profile.   |

**No private keys ever live in the browser.** The Tempo wallet private key
is a Secrets Manager entry; it is loaded once at AgentCore container
startup and held in module-level closure. The LLM never sees it, and no
tool takes it as a parameter.

## The three endpoints — why the listing service needs all of them

- **`GET /openapi.json`** — free. Samurai calls this at startup (cached) to
  learn what fields each marketplace needs. New platforms can be added by
  editing `rules.json` only — Samurai discovers them automatically.
- **`POST /validate`** — free. Samurai calls this before paying to confirm
  input shape is acceptable. Same rules the paid gate uses.
- **`POST /generate`** — paid. Tiered: 400 → 422 → 402 → 200. Bedrock is
  invoked ONLY after payment verifies.

This three-tier shape is MPP-compliant: payment only happens for requests
that would actually do work. Malformed or invalid requests never charge
the caller.

## Workshop CLI artifacts

- `infrastructure/scripts/package_for_workshop.sh` — packages CFN
  templates + Lambda zips + a repo zip with a TODO overlay.
- `infrastructure/scripts/deploy_to_workshop.sh` — pushes packaged content
  to a Workshop Studio repo and syncs assets to S3.
- `infrastructure/scripts/build-and-upload-spa.sh` — post-stack: builds
  the Vite SPA with Cognito IDs injected, uploads `dist/` to the SPA
  bucket, seeds `/config.json`, invalidates CloudFront.
- `workshop/overlay/workshop/code/participant/participant-deploy.sh`
  — participant-side: builds Samurai container for linux/arm64, pushes to
  ECR, deploys the participant CFN, writes the runtime ARN into the SPA's
  `/config.json`.

## Assumptions and limits

- Stripe is **test mode only** (`sk_test_...`). Crypto/deposit mode
  requires API version `2026-03-04.preview`.
- Tempo is testnet. The workshop's `LISTING_PRICE` is hard-coded to
  `0.01`. Samurai's wallet is auto-generated at stack creation; fund it via
  the Stripe/Tempo testnet faucet if you want real on-chain transfers.
- The MPP logs table is session-scoped with a 24 h TTL. The SPA queries
  only the signed-in user's session rows.
- The CFN assumes `us-east-1` by default (parameterized; Bedrock Sonnet
  4.6 is reachable globally via the inference profile).

## Where to learn more

- Participant tutorial (chapters 1–6): [`workshop/content/`](workshop/content/)
- Stripe MPP docs: https://docs.stripe.com/payments/machine/mpp
- Tempo MPP server guide: https://docs.tempo.xyz/guide/machine-payments/server
- Bedrock AgentCore Runtime: https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime.html
- Strands Agents SDK (TS): https://github.com/strands-agents/sdk-typescript
- Earlier FIAT variant (Cognito + Checkout SAM stack):
  `archive/fiat-reference/` — kept for historical context; not deployed.
