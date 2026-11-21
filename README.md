# stranded-guest-support

A team of AI helpers that rescues the 60 guests stranded in Lisbon on 16 November 2026. Each guest chat ends in a new home, a refund, or a live safety agent. Anyone in danger reaches a person before any money moves.

Python 3.12 or 3.13. The package lives in `src/stranded_guest_support` and is empty so far.

## Read first

- `.specify/memory/constitution.md`: the rules that override everything else
- `specs/002-ticket-handling-rules/spec.md`: the ticket handling rules explained below
- `specs/001-stranded-guest-support/spec.md`: the wider crew design (flow, tools, tests). Where it differs from 002, 002 wins
- `data/`: the refund and rebooking policy and the 60 tickets

## How tickets are handled

Every ticket is one of three types, and each type has one specialist agent.

| Type | Agent | Tool it may use |
|---|---|---|
| Rebooking | Rebooking agent | `hold_booking` (never `issue_refund`) |
| Refund | Refund agent | `issue_refund` (never `hold_booking`) |
| Safety | Safety agent | `handoff_to_safety_agent` only |

The agents also have read and messaging tools, listed in spec 001. The two money tools each belong to one agent.

### Safety comes first

- Any one of the five AirCover safety conditions in the guest message sends the ticket to a live safety agent:
  - `injury_or_medical_need`
  - `threat_or_violence`
  - `nowhere_safe_to_wait`
  - `unaccompanied_minor`
  - `stated_fear_of_person`
- This holds even when the guest asks only for money, a call or a new flat. The guest's words decide, not the `guest_requested` label.
- On a safety ticket no agent may call `issue_refund` or `hold_booking` before `handoff_to_safety_agent`. After the hand-off they stay forbidden until a live safety agent has taken the case, and the team then stops. The system enforces this, not the agent prompts.
- If a message might describe a condition, the ticket goes to the safety agent. If a safety cue shows up mid-chat, the rebooking or refund agent stops and the ticket moves to safety.
- Anger or a callback request alone is not a safety condition. Those tickets stay with the rebooking agent.

### Money limits

- A refund never exceeds `total_paid`, goodwill included. LIS-TKT-0028 paid EUR 432.00 after a promo code, so its cap is EUR 432.00, not nights times price.
- A rebooked home costs at most 120% of the original `nightly_price`. On EUR 110.00, EUR 132.00 passes and EUR 132.01 fails.
- One remedy per stay: a rebooking or a refund, not both.
- If no home passes, or the guest declines one, the refund agent issues the full `total_paid`. The rebooking agent never refunds.

### What the 60 tickets should show

6 safety tickets (LIS-TKT-0007, 0015, 0023, 0034, 0046, 0055), 34 rebooking and 20 refund. All 6 safety tickets must reach a live safety agent with no money action first.

## Set up

```bash
uv venv --python 3.13
uv pip install -e .
```

API keys go in environment variables or a local `.env`. Git ignores `.env`.

## Status

Specs 001 and 002 drafted, code not started. The 450-listing file the policy names is still missing.
