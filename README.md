# stranded-guest-support

A team of AI helpers that rescues the 60 guests stranded in Lisbon on 16 November 2026. Each guest chat ends in a new home, a refund, or a live person from Trust and Safety. Anyone in danger reaches a person before any money moves.

Python 3.12 or 3.13. The package lives in `src/stranded_guest_support` and is empty so far.

## Read first

- `.specify/memory/constitution.md`: the rules that override everything else
- `specs/002-ticket-handling-rules/spec.md`: the one spec. It covers the four agents, the six tools, ticket types, safety conditions, comparable-home tests, money caps and the 60 tickets
- `data/`: the refund and rebooking policy and the 60 tickets

## The team

Four agents and six tools. There is no separate safety agent.

| Agent | What it does | Tools it may use |
|---|---|---|
| Triage | Reads each ticket, runs the safety check first, hands unsafe tickets to a live person, sorts the rest as rebooking or refund | `handoff_to_safety_agent` |
| Rebooking | Finds a comparable home and holds it | `lookup_reservation`, `search_open_listings`, `hold_booking` |
| Refund | Refunds up to `total_paid` | `lookup_reservation`, `issue_refund` |
| Host messaging | Contacts the host on a cleared ticket | `message_host` |

Every ticket is one of three types: rebooking, refund or safety. The two money tools each belong to one agent: only the rebooking agent holds homes and only the refund agent refunds.

## The rules

### Safety is a hard gate

- Any one of the five AirCover safety conditions in the guest message sends the ticket to a live person:
  - `injury_or_medical_need`
  - `threat_or_violence`
  - `nowhere_safe_to_wait`
  - `unaccompanied_minor`
  - `stated_fear_of_person`
- This holds even when the guest asks only for money, a call or a new flat. The guest's words decide, not the `guest_requested` label.
- The safety check runs before any money or booking tool. `issue_refund` and `hold_booking` are refused on a ticket the check has not cleared, and on a flagged ticket before `handoff_to_safety_agent`. After the hand-off they stay forbidden until a live person has taken the case, and the team then stops. The booking system enforces this, not the agent prompts.
- No host message is sent on a flagged ticket, since the guest may fear the host.
- If a message might describe a condition, the ticket goes to a live person. If a safety cue shows up mid-chat, the rebooking or refund agent stops and the ticket goes back to triage.
- Anger or a callback request alone is not a safety condition. Those tickets stay with the rebooking agent.

### Money limits

- A refund never exceeds `total_paid`, goodwill included. LIS-TKT-0028 paid EUR 432.00 after a promo code, so its cap is EUR 432.00, not nights times price.
- A rebooked home costs at most 120% of the original `nightly_price`. On EUR 110.00, EUR 132.00 passes and EUR 132.01 fails.
- One remedy per stay: a rebooking or a refund, not both.
- If no home passes, or the guest declines one, the refund agent issues the full `total_paid`. The rebooking agent never refunds.

### What the 60 tickets should show

6 safety tickets (LIS-TKT-0007, 0015, 0023, 0034, 0046, 0055), 34 rebooking and 20 refund. All 6 safety tickets must reach a live person with no money action first.

## Set up

```bash
uv venv --python 3.13
uv pip install -e .
```

API keys go in environment variables or a local `.env`. Git ignores `.env`.

## Status

Spec drafted, code not started. The 450-listing file the policy names is still missing.
