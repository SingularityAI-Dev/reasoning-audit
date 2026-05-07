# Sample Reasoning Audit — anonymised
**Buyer:** Acme AI (anonymised, name redacted at buyer's request)
**Stack:** TypeScript, OpenAI gpt-4.1-mini via Vercel AI SDK, custom orchestration, Postgres-backed memory
**Audit date:** 2026-04-22
**Auditor:** Rainier Potgieter, maintainer of [LOGIC.md](https://github.com/SingularityAI-Dev/logic-md)
**Google Meet:** redacted (24 minutes, sent to buyer)

This is a real anonymised deliverable from a recent audit. Names, prompts, and identifying snippets are scrubbed. Every other detail is verbatim.

---

## Executive summary

Acme AI ships a customer-support agent that retrieves order history, drafts replies, and posts to Zendesk. The agent passes their evals and silently drops messages in production roughly 3% of the time, which surfaces as "ghost ticket" complaints from users. The most critical failure mode is silent tool-call failure on the Zendesk post step. Recommended fix: replace the bare `try/await` around the post call with a structured retry policy and a postcondition assertion that verifies the ticket exists in Zendesk after the call. This fix, applied alone, will close roughly 80% of their reported "ghost ticket" incidents.

---

## Top 3 failure modes (ranked)

### #1 — Silent tool-call failure on Zendesk post

**What's happening:**
The agent calls `zendesk.tickets.create()` inside a `try/await` block. When Zendesk returns a 5xx (which they do 1-2% of the time during peak hours), the call throws, the catch block logs to stderr, the agent then enters a "user notified" branch as if the post succeeded. The user sees the agent reply in the chat thread, but no ticket is filed in Zendesk. Support team has no record. Users complain.

**Root cause:**
The success branch is selected based on `!error`, not on the actual return value of the post. The agent has no postcondition asserting that the ticket exists in Zendesk after the call.

**Fix:**

```typescript
// BEFORE (the bug)
try {
  await zendesk.tickets.create({ ... });
  notifyUserSuccess();
} catch (e) {
  log.error(e);
  notifyUserSuccess(); // bug: still in the success branch
}

// AFTER (the fix)
async function postTicketWithVerification(payload) {
  let ticket;
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      ticket = await zendesk.tickets.create(payload);
      // postcondition: ticket actually exists
      const verify = await zendesk.tickets.show(ticket.id);
      if (verify?.id === ticket.id) return ticket;
    } catch (e) {
      log.warn({ attempt, error: e.message }, 'zendesk.create failed');
      await sleep(2 ** attempt * 200);
    }
  }
  throw new ToolCallFailure('zendesk.tickets.create did not verify after 3 attempts');
}
```

Then in the agent:

```typescript
try {
  const ticket = await postTicketWithVerification(payload);
  notifyUserSuccess(ticket.id);
} catch (e) {
  notifyUserFailure(); // distinct branch
  alertOpsTeam(e);
}
```

**Confidence the fix lands:** **High.** This is a textbook silent-tool-call-failure pattern. The retry loop alone closes most of the 5xx-induced cases. The postcondition assertion catches the residual long tail.

---

### #2 — Contract drift on reply tone over long conversations

**What's happening:**
The agent's system prompt specifies "professional, empathetic tone" and lists 5 example reply patterns. Over conversations longer than 8 turns, the agent starts using more casual phrasing, sometimes drops the empathy markers entirely, and occasionally adopts the user's frustrated tone. Logged QA reviews show this happens in ~12% of long conversations.

**Root cause:**
The system prompt is included at turn 0 only. Subsequent turns rely on the model maintaining the contract through the chat history alone. Long histories dilute the system-level instructions, especially when the user injects emotionally charged content.

**Fix:**
Re-anchor the contract every N turns (5 in their case) by injecting a compressed restatement of the tone contract as an assistant-side message before the next user turn. Or use a prefix-cache friendly system reminder.

Recommended LOGIC.md contract approach:

```yaml
# tone-contract-stub.md
contract:
  name: customer_support_tone
  version: 0.1
  invariants:
    - tone: professional, empathetic
    - never_match_user_frustration: true
    - empathy_markers_per_reply: ">= 1"

quality_gates:
  - id: tone_check_every_5_turns
    when: "turn_index % 5 == 0"
    assert: "reply matches tone invariants"
    action_on_fail: re-inject system reminder
```

**Confidence the fix lands:** **Medium-high.** Tone is fuzzy and the assertion is judgment-based. A small classifier or a second LLM call (gpt-4.1-mini) per check can score the reply against the invariants. Costs about $0.003 per check.

---

### #3 — Retry-loop blow-up on rate-limited memory writes

**What's happening:**
When Postgres-backed memory writes hit the `pgbouncer` connection limit during traffic spikes, the agent retries indefinitely without backoff or cap. Last week this produced a 14-minute outage where 70% of agent traffic was wedged trying to write the same memory record.

**Root cause:**
The retry logic uses `while (true)` with a flat `sleep(500)`. No exponential backoff, no max-attempts cap, no circuit breaker.

**Fix:**

```typescript
async function writeMemoryBounded(record, opts = { maxAttempts: 3, baseMs: 200 }) {
  for (let attempt = 0; attempt < opts.maxAttempts; attempt++) {
    try {
      return await memory.write(record);
    } catch (e) {
      if (e.code === 'PG_CONN_LIMIT' && attempt < opts.maxAttempts - 1) {
        await sleep(opts.baseMs * 2 ** attempt);
        continue;
      }
      throw e;
    }
  }
}
```

Plus: open a circuit breaker on the memory writer if 5 of the last 10 writes hit `PG_CONN_LIMIT`. When the breaker is open, queue writes locally for replay.

**Confidence the fix lands:** **High.** Standard pattern, well-understood. Closes the production outage class entirely.

---

## LOGIC.md contract stub

If you adopt LOGIC.md as the reasoning contract layer, a starter spec for the support agent looks like this:

```yaml
contract:
  name: customer_support_agent
  version: 0.1
  inputs:
    - name: user_message
      type: string
      required: true
    - name: order_id
      type: string
      required: false

  outputs:
    - name: reply
      type: string
      shape: { tone: professional }
    - name: ticket_id
      type: string
      shape: { exists_in_zendesk: true }

steps:
  - id: retrieve_order_history
    pre: [order_id is present OR user_message contains order reference]
    post: [order_history is loaded OR user_message_about_order is false]
    retry: { max: 2, on: [PG_CONN_LIMIT] }

  - id: draft_reply
    pre: [tone_contract loaded]
    post: [reply matches tone invariants]

  - id: post_ticket_zendesk
    pre: [reply finalized]
    post: [ticket_id exists in zendesk]
    retry: { max: 3, on: [HTTP_5XX] }

quality_gates:
  - id: silent_failure_check
    assert: ticket_id is verified in zendesk after step post_ticket_zendesk
    fail_action: distinct_user_facing_failure_path

  - id: tone_drift_check
    when: turn_index % 5 == 0
    assert: reply matches tone invariants
    fail_action: re_inject_tone_reminder
```

Drop this into `contracts/support_agent.md` and run it through the [LOGIC.md CLI](https://github.com/SingularityAI-Dev/logic-md) to validate.

---

## Recommended next steps

1. **This week:** Apply fix #1 (silent tool-call). Highest impact, lowest effort. Ship behind a feature flag.
2. **This week:** Apply fix #3 (retry-loop). Addresses the 14-minute-outage class.
3. **Next week:** Roll out the LOGIC.md contract stub. Start with the silent-failure quality gate.
4. **Next week:** Wire the tone-drift check (fix #2) as a sampled eval, not blocking.
5. **Two weeks out:** Trajectory eval (not just final-output eval) on the agent in CI. The current eval suite passes the silent-failure case because it only checks the user-facing reply not the side-effect on Zendesk.

---

## Refund clause

Full refund if no fix recommendation in this report lands inside 14 days of payment. Reply to info@singlesource.co.za stating which fix you tried and what the outcome was. No questions, no negotiation.

---

## Optional: case study opt-in

If you would like this audit to feature as a non-anonymised case study on the reasoning-audit repo, reply with "case study OK" and the level of detail you are comfortable sharing. We never publish without written consent.

