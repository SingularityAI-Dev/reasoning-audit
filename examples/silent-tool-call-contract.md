# Proof of concept: catching silent tool-call failures with a LOGIC.md contract

This file shows the single most-shipped fix from the audit, expressed as a portable LOGIC.md reasoning contract. Drop it into a project, run it through the LOGIC.md CLI, and the runtime enforces the postcondition for you.

## The bug we are catching

```typescript
// Naive agent code
async function postTicket(payload) {
  try {
    await zendesk.tickets.create(payload);
    return { status: 'success' };  // bug: succeeds even on 5xx, the catch falls through
  } catch (e) {
    log.error(e);
    return { status: 'success' };  // bug: still claims success
  }
}
```

This fails 1-2% of the time during traffic spikes. The user sees a confirmation reply. Support has no record. Audit ticket: "ghost tickets."

## The contract that catches it

```yaml
# contracts/post-ticket.logic.md
contract:
  name: post_ticket_zendesk
  version: 0.1
  description: |
    Posts a support ticket to Zendesk. Verifies the ticket exists
    in Zendesk after the call. Distinguishes success from silent failure.

  inputs:
    - name: payload
      type: object
      required: true
      shape:
        - subject: string
        - description: string
        - requester_email: string

  outputs:
    - name: ticket_id
      type: string
      required: true
      assert:
        # The postcondition that catches the silent failure
        - exists_in_zendesk: true

steps:
  - id: create_ticket
    description: POST to Zendesk tickets API
    pre:
      - payload.subject is non-empty
      - payload.requester_email matches email format
    post:
      - response.id is present
      - response.status_code in [200, 201]
    retry:
      max: 3
      on: [HTTP_5XX, HTTP_429, NETWORK_TIMEOUT]
      backoff: exponential
      base_ms: 200

  - id: verify_ticket_exists
    description: GET the ticket back to confirm it landed
    depends_on: [create_ticket]
    pre:
      - create_ticket.response.id is present
    post:
      - verify_response.id == create_ticket.response.id
    retry:
      max: 2
      on: [HTTP_5XX, HTTP_404]
      backoff: linear
      base_ms: 500

quality_gates:
  - id: no_silent_failure
    description: |
      The agent's user-facing success path must only fire if
      verify_ticket_exists passed. This is the gate that closes
      the silent-failure class.
    assert: |
      verify_ticket_exists.passed == true
    on_fail:
      action: distinct_failure_branch
      message: "Ticket creation could not be verified after retries. Alerting ops."

  - id: bounded_retries
    description: Prevents the retry-loop blow-up cousin failure mode
    assert: |
      total_retry_attempts <= 5
    on_fail:
      action: open_circuit_breaker
      cooldown_ms: 30000
```

## How the contract changes runtime behaviour

Without LOGIC.md (the buggy version above):
- Zendesk returns 503 → catch block fires → return `{status: success}` → user sees confirmation → no ticket exists → ghost ticket complaint.

With the LOGIC.md runtime executing this contract:
- Zendesk returns 503 → `create_ticket.post` assertion fails → retry policy kicks in (3 attempts, exponential backoff).
- All 3 retries fail → step itself fails → `no_silent_failure` quality gate fails → `on_fail.action: distinct_failure_branch` runs → user sees a "we hit an issue, your message has been queued, we will follow up" reply, NOT a fake confirmation.
- Ops gets paged. Support has the queued message in dead-letter storage for replay.

## Test fixture (TypeScript)

```typescript
// tests/post-ticket.test.ts
import { runContract, mockTool } from '@logic-md/core';
import { describe, it, expect } from 'vitest';

describe('post_ticket_zendesk contract', () => {
  it('routes 503 responses through the failure branch, not the success branch', async () => {
    const fakeZendesk = mockTool('zendesk', {
      'tickets.create': () => { throw { code: 'HTTP_5XX', status: 503 }; },
      'tickets.show': () => { throw { code: 'HTTP_404' }; }
    });

    const result = await runContract('contracts/post-ticket.logic.md', {
      payload: {
        subject: 'order issue',
        description: 'my order is late',
        requester_email: 'user@example.com'
      },
      tools: { zendesk: fakeZendesk }
    });

    expect(result.passed).toBe(false);
    expect(result.failed_gate).toBe('no_silent_failure');
    expect(result.user_facing_branch).toBe('distinct_failure_branch');
    expect(fakeZendesk.callCount('tickets.create')).toBe(3); // retries fired
  });

  it('passes when Zendesk recovers on retry', async () => {
    let attempts = 0;
    const fakeZendesk = mockTool('zendesk', {
      'tickets.create': () => {
        attempts++;
        if (attempts < 2) throw { code: 'HTTP_5XX', status: 503 };
        return { id: 'tk_1234', status_code: 201 };
      },
      'tickets.show': () => ({ id: 'tk_1234' })
    });

    const result = await runContract('contracts/post-ticket.logic.md', {
      payload: {
        subject: 'order issue',
        description: 'my order is late',
        requester_email: 'user@example.com'
      },
      tools: { zendesk: fakeZendesk }
    });

    expect(result.passed).toBe(true);
    expect(result.outputs.ticket_id).toBe('tk_1234');
  });

  it('catches the silent-success bug: 503 must never produce ticket_id', async () => {
    const fakeZendesk = mockTool('zendesk', {
      'tickets.create': () => { throw { code: 'HTTP_5XX', status: 503 }; }
    });

    const result = await runContract('contracts/post-ticket.logic.md', {
      payload: { subject: 'x', description: 'y', requester_email: 'a@b.c' },
      tools: { zendesk: fakeZendesk }
    });

    expect(result.outputs.ticket_id).toBeUndefined();
    expect(result.passed).toBe(false);
  });
});
```

## How to run this in your repo

```bash
npm install -g @logic-md/cli
logic-md validate contracts/post-ticket.logic.md
logic-md test contracts/post-ticket.logic.md --fixtures tests/
```

The contract is portable across runtimes. Same spec runs on top of LangGraph, Mastra, Vercel AI SDK, or vanilla OpenAI calls. The LOGIC.md adapter for your stack handles the dispatch.

## What the audit produces for you

If you book the audit ([repo](https://github.com/SingularityAI-Dev/reasoning-audit) / mailto: info@singlesource.co.za):

1. I read your agent code and find the equivalent of the bug above (every team has at least one).
2. I write a `your-failure-mode.logic.md` contract specific to your stack, like the one above.
3. I deliver it as a fork-able `contracts/` folder you drop into your repo.

48 hours, $49, refund if no fix lands inside 14 days.

