---
name: liberate-advance-workflow
description: >-
  Advance a Liberate workflow instance that is paused on an Intermediate Event by delivering an event
  message and optional context. Use when an external system has finished a long-running operation, or
  when a human has made the approval or review decision the workflow is waiting on.
api: Liberate Orchestration Platform API
spec: ../openapi/liberate-innovations-orchestration-openapi.yml
operations:
  - sendIntermediateEvent
generated: '2026-07-19'
method: generated
source: >-
  openapi/liberate-innovations-orchestration-openapi.yml,
  https://docs.liberateinc.com/docs/intermediate-events
---

# Advance a paused Liberate workflow instance

## When this applies

Asynchronous Liberate workflows pause at **Intermediate Events** — synchronization points that wait
for something outside the workflow to finish. The documented reasons a workflow waits:

- **Asynchronous operations** — a long-running external process that takes minutes, hours, or days.
- **External system interactions** — waiting on a response from a partner API or service.
- **Human in the loop** — an approval, review, or adjuster decision.

While paused, the instance holds its position and its context. It resumes only when an event is
delivered to it.

## Before you begin

You need:

1. **Your tenant endpoint URL** and a **bearer token** for the correct environment (production or
   QA). Same credentials as `startWorkflow`.
2. **The `instanceId`** of the paused session — returned when the workflow was started. There is no
   API to look one up, so it must have been persisted at start time.
3. **The event message** the workflow is waiting for. Which string a given Intermediate Event expects
   is defined in the workflow on the Liberate canvas, not discoverable over the API.

## Steps

### 1. Confirm the instance is actually paused

Check the Liberate reporting interface, which organizes sessions at the Correlation ID level and
shows in-progress, completed, and failed sessions. Sending an event to a session that has already
completed, or to the wrong instance, has no defined recovery path.

### 2. Deliver the event

`POST /xmanager/event` with the `instanceId`, the `event` message, and optionally a `context` object:

```bash
curl -X POST \
     -H "Authorization: Bearer $LIBERATE_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "instanceId": "'"$INSTANCE_ID"'",
           "event": "adjuster-approved",
           "context": {}
         }' \
     "$LIBERATE_ENDPOINT/xmanager/event"
```

Include context when the event carries data the rest of the workflow needs:

```bash
curl -X POST \
     -H "Authorization: Bearer $LIBERATE_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{
           "instanceId": "'"$INSTANCE_ID"'",
           "event": "inspection-complete",
           "context": {
             "inspection": {
               "outcome": "approved",
               "estimatedAmount": 4820.00,
               "inspectorNotes": "Roof replacement required."
             }
           }
         }' \
     "$LIBERATE_ENDPOINT/xmanager/event"
```

### 3. Let the workflow reshape the data

Once the event lands, the workflow resumes at the next step. The Intermediate Event can declare an
**action name** and an **action transformation** that reshape the delivered payload and write it into
the workflow context under that namespace. Downstream steps then read it as
`$.actionName.{jsonobject}`.

This means you do **not** need to match the workflow's internal data shape — send the natural shape
of your source system and let the JSONata transformation on the Intermediate Event adapt it. That is
the documented separation of concerns.

## Rules and cautions

- **This is not idempotent.** No idempotency key, no deduplication window. Delivering the same event
  twice may advance the workflow twice or land on an unexpected step. Track delivery on your side and
  do not retry blindly on a timeout.
- **This can be an approval.** In human-in-the-loop flows, delivering the event *is* the decision.
  Never send speculatively, and never let an agent send one without explicit human confirmation.
- **Verify the `instanceId` first.** It is the only routing key. A wrong ID targets someone else's
  live orchestration.
- **A REST Task usually precedes the Intermediate Event.** The documented pattern is that the
  workflow makes the outbound HTTP POST to kick off the external process in a REST Task immediately
  before pausing on the Intermediate Event. If you are the external system, your callback is this
  endpoint.
- **No error catalog is published.** Treat non-2xx responses as opaque; `401` means a missing or
  invalid bearer token. Inspect the session in the reporting interface to see where it actually is.
- **Environment matters.** Instance IDs are not portable between QA and production. Use the token and
  endpoint for the environment the session was started in.

## Related

- Start a workflow: `liberate-innovations-start-workflow.md`
- Conventions, tracing, and error posture: `../conventions/liberate-innovations-conventions.yml`
- Entity graph (Session, Context, Event): `../data-model/liberate-innovations-data-model.yml`
