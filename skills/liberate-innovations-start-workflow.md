---
name: liberate-start-workflow
description: >-
  Start a Liberate Orchestration Platform workflow by slug, optionally seeding it with context data,
  and correctly interpret the synchronous versus asynchronous response. Use when triggering an
  insurance orchestration such as a first-notice-of-loss intake, a policy change, or a claim filing
  from an external system.
api: Liberate Orchestration Platform API
spec: ../openapi/liberate-innovations-orchestration-openapi.yml
operations:
  - startWorkflow
generated: '2026-07-19'
method: generated
source: >-
  openapi/liberate-innovations-orchestration-openapi.yml,
  https://docs.liberateinc.com/docs/calling-a-workflow
---

# Start a Liberate workflow

## Before you begin

You need three things, and none of them are self-service — Liberate provisions all of them:

1. **Your tenant endpoint URL.** Every Liberate customer gets their own. The documented production
   example is `https://integration.liberateinc.io`, but that is an example, not a shared host. Find
   your real URL in the curl example inside the **Start Event** properties of the workflow on the
   Liberate canvas.
2. **The environment.** Every customer has both a production and a QA environment, each with its own
   URL and its own token. The Start Event shows the curl request for each. Pick deliberately.
3. **A bearer token** issued by Liberate for that environment.
4. **The flow slug**, also from the Start Event properties. Slugs look like
   `simple-flow-d9379aa1-2508-49a2-b7be-7537013535f0`. There is no list operation — the API cannot
   tell you which slugs exist, so they must be supplied out of band.

## Steps

### 1. Choose the workflow type you are calling

Check whether the target workflow is synchronous or asynchronous, because it changes how you handle
the response:

- **Synchronous** — responds with a status code and instance ID, runs to completion, and returns the
  flow's end output as the response body. Use for real-time lookups where the caller waits.
- **Asynchronous** — responds immediately with a status code and instance ID, then continues in the
  background for minutes, hours, or days, pausing at Intermediate Events for external systems or
  human decisions.

### 2. Build the context

`context` is free-form JSON that becomes the workflow's starting state. Everything you put here is
readable downstream with JSONata. Shape it to whatever the workflow's Sample Context expects — pass
an empty object `{}` if the workflow needs no seed data.

### 3. Call `startWorkflow`

`PUT /` with a bearer token and a JSON body containing `slug` and `context`:

```bash
curl -X PUT \
     -H "Authorization: Bearer $LIBERATE_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"slug": "'"$FLOW_SLUG"'", "context": {}}' \
     "$LIBERATE_ENDPOINT"
```

With context, for a property first-notice-of-loss intake:

```bash
curl -X PUT \
     -H "Authorization: Bearer $LIBERATE_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "slug": "'"$FLOW_SLUG"'",
           "context": {
             "session": {
               "hubRadio": "file_a_claim",
               "propertyFnolLob": "home",
               "propertyFnolLossType": "weather",
               "propertyFnolLossCause": "hurricane",
               "propertyFnolLossDescriptionInput": "Roof damage after the storm."
             }
           }
         }' \
     "$LIBERATE_ENDPOINT"
```

### 4. Capture the identifiers

Keep both:

- **`instanceId`** — this specific run. You need it to advance the workflow later with
  `sendIntermediateEvent`, and to find the session in the Liberate reporting interface.
- **`correlationId`** — the parent orchestration, when this workflow was invoked by another through
  a Workflow Instance Task. Reporting groups sessions at this level.

Store them against your own record before doing anything else. There is no way to look up a session
by business key over the API — if you lose the `instanceId` you lose your handle on the run.

## Rules and cautions

- **This is not idempotent.** It uses `PUT`, but every call creates a *new* workflow session with a
  new Instance ID. Do not retry blindly on a timeout — you will start a second orchestration.
  Liberate publishes no idempotency key header and no deduplication window.
- **Side effects are real.** A Liberate workflow can write to a carrier's claims system, a rating
  engine, a CRM, or send customer notifications. Confirm with a human before triggering in
  production.
- **Handle retries inside the workflow, not outside it.** Liberate's documented retry mechanism is
  the Error Task: attach one to any task prone to failure and define recovery, retry, or alternative
  behavior there. Errors routed to an Error Task are reported into Liberate's reporting system.
- **There is no published error catalog.** Liberate documents no error envelope, no error codes, and
  no `application/problem+json` responses. Treat a non-2xx as opaque, log the full body, and check
  the session in the reporting interface. A `401` means a missing or invalid bearer token.
- **No rate limits are published.** Absence of a documented limit is not a guarantee of none — back
  off on failure rather than hammering.
- **Never hardcode the token or endpoint.** Both are per-customer and per-environment. Read them
  from configuration so promoting from QA to production is a config change, not a code change.

## Related

- Advance a paused instance: `liberate-innovations-advance-workflow.md`
- Conventions and error posture: `../conventions/liberate-innovations-conventions.yml`
- Authentication detail: `../authentication/liberate-innovations-authentication.yml`
- Entity graph: `../data-model/liberate-innovations-data-model.yml`
