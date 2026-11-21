# Spec: Stranded Guest Support Crew

Status: draft for review. Source of truth for money and safety rules: `data/aircover_rebooking_and_refund_rules.md`. Principles: `.specify/memory/constitution.md`.

## 1. Goal

Before midnight on 16 November 2026, every one of the 60 stranded Lisbon guests has a new home, their money back, or a live safety agent. Anyone in danger reaches a person before any money or booking action.

## 2. Outcomes

Every ticket ends in exactly one of these:

| Outcome | What the guest gets |
|---|---|
| `rebooked` | A rebooking card for a comparable home that is on hold |
| `refunded` | A refund confirmation |
| `safety_handoff` | A live Trust and Safety agent takes the case |
| `human_review` | A support person takes the case, with a written reason (fallback only) |

## 3. How a ticket flows

1. **Safety screen** (a fixed step in the graph, not a free-roaming agent). Runs on every ticket and on every new guest message. It has two detectors: a keyword and pattern rule set, and an LLM classifier asked only "does this message describe any of the five conditions?". A ticket is flagged if **either** fires. If the classifier errors, times out or answers unclearly, the ticket is flagged too. Routing on the result is plain code. A flagged ticket goes to the **Safety hand-off agent** and nothing else runs.
2. **Ticket sorter** (agent). Only sees tickets the screen cleared. Picks `rebooking`, `refund`, or `human_review`.
3. **Specialist** handles it with its allowed tools only.
4. **Close or escalate.** Solved or sent to a human, per section 6.

The sorter never decides safety. It can only confirm "no safety flag" because the screen already said so.

## 4. Ticket types, specialists and tools

| Type | Triggers | Specialist | Allowed tools | Forbidden |
|---|---|---|---|---|
| Safety | Any of the five conditions, or doubt (section 5), in any ticket regardless of `guest_requested` | Safety hand-off agent | `handoff_to_safety_agent`; `escalate_to_human` only if the hand-off fails twice | `hold_booking`, `issue_refund`, `search_open_listings`, any message about money |
| Rebooking | Guest wants a new place, or `callback`/`other` where a new place is what they describe | Rebooking agent | `get_booking`, `search_open_listings`, `get_uplift_budget`, `hold_booking`, `send_rebooking_card`, `escalate_to_human` | `issue_refund`, `handoff_to_safety_agent` |
| Refund | Guest declines a new home or asks for money back | Refund agent | `get_booking`, `issue_refund`, `send_refund_confirmation`, `escalate_to_human` | `hold_booking`, `search_open_listings`, `handoff_to_safety_agent` |
| Human review | Anything the sorter cannot place with confidence (section 5, non-safety signs) | none (queue) | `escalate_to_human` | everything else |

A specialist that sees danger mid-conversation does not act on it. It stops, and the graph routes the ticket back to the safety screen (an edge from every specialist to the screen). Any new guest message re-runs the screen too. The rebooking agent never calls `issue_refund`. When no home passes, it hands the ticket to the Refund agent, which issues the full `total_paid`.

## 5. Warning signs that send a ticket straight to a human

### Safety signs (go to a live safety agent)

| Condition | What the guest describes |
|---|---|
| `injury_or_medical_need` | Hurt, bleeding, unwell, pregnant, needs medication or a doctor, a relative who looks ill |
| `threat_or_violence` | Threatened, followed, shouted at, harmed, someone watching them |
| `nowhere_safe_to_wait` | On the street at night, phone dying, alone with luggage, asked to leave a cafe, cold, no idea where to go |
| `unaccompanied_minor` | Anyone under 18 alone or without an adult, including when a guest states their age |
| `stated_fear_of_person` | Says they are scared or afraid of the host, a neighbour, a stranger or someone in the party |

Treat these as signs even when the ticket says `guest_requested = refund`, is not labelled urgent, or the guest asks only for money.

### Doubt rule

If the screen is unsure whether a message describes one of the conditions, it flags the ticket. Wrongly sending a ticket to a safety agent costs a few minutes. Missing one costs a person.

Everyday frustration is not a safety sign by itself ("worst day of our lives", "I am dying out here", an angry tone, travel fatigue, a baby or grandparents in the party). If a message mixes frustration with a real cue, the real cue wins.

### Non-safety signs (go to human review)

- The sorter cannot tell whether the guest wants a home or money, and the guest has not said which.
- Two requests conflict (for example, "refund me" and "find me a place" in one message).
- The booking record contradicts the ticket: unknown `reservation_id`, wrong dates, or a requested refund above `total_paid`. A `total_paid` below nights × price is a discount, not a contradiction. Refund up to `total_paid`.

A side question the policy does not answer (luggage, a cot, parking) does not block the remedy. The agent completes the rebooking or refund and adds the question to the human queue as a note.
- The uplift budget is too low to cover a comparable home that otherwise qualifies.
- A tool call fails twice in a row.

## 6. Solved or sent to a human

| Type | Solved when | Otherwise |
|---|---|---|
| Safety | `handoff_to_safety_agent` returned a case ID. A live agent has acknowledged or is queued. The crew then stops and sends the guest no money-related text | If the hand-off call fails, retry once. If it fails again, page the on-call safety lead through `escalate_to_human` with priority `safety` |
| Rebooking | A comparable home is on hold **and** the rebooking card has been sent. Both, not either | No comparable home passes: refund the full `total_paid` (see "Decisions to confirm"). Hold failed: try the next best home. All fail: `human_review` |
| Refund | `issue_refund` succeeded for an amount from 0.01 up to `total_paid` **and** the confirmation has been sent | Refund call fails twice: `human_review` |
| Human review | A person has been given the ticket and a one-line reason | none |

## 7. Policy rules the agents must apply

**Qualifying for support** (any one): no working access at check-in, host unreachable, or host cancelled within 30 days of check-in. Support covers the whole stay.

**A comparable home must pass all five tests:**

1. Dates: `available_from` ≤ `check_in_date` and `available_to` ≥ `check_out_date`, no gap.
2. Party: `max_guests` ≥ `party_size`.
3. Bedrooms: listing `bedrooms` ≥ ticket `bedrooms`.
4. Status: `available` at the moment of the hold. `on_hold` and `booked` fail.
5. Price: listing `nightly_price` ≤ 1.20 × ticket `nightly_price`. EUR 132.00 passes, EUR 132.01 fails against a EUR 110.00 original.

Among homes that pass, prefer the guest's neighbourhood or a neighbouring one. That is a preference, not a test. A guest can turn down an offer and still get another home or a refund.

**Listing IDs.** `original_listing_id` (LST-200001 to LST-200060) is never a valid rebooking target. A rebooking names an ID from LST-100001 to LST-100450.

**Refunds.** Never more than `total_paid`, including goodwill. Full `total_paid` is due when no home passes or the guest declines rebooking. One remedy per stay.

**Uplift.** Uplift = (new `nightly_price` − original `nightly_price`) × nights. The incident budget is EUR 9,000, with EUR 150 per rebooked guest as a planning figure only. The 120% per-night ceiling is the binding limit. Nothing above it is offered on 16 November 2026.

## 8. Tools (the fake booking system)

| Tool | Does | Enforced by the server |
|---|---|---|
| `get_booking(reservation_id)` | Returns the booking and ticket fields | |
| `search_open_listings(filters)` | Returns open Lisbon listings | |
| `get_uplift_budget()` | Returns budget used and remaining | |
| `hold_booking(ticket_id, listing_id)` | Places a hold | Refuses on an open safety flag without live-agent acknowledgement. Refuses an `LST-2000xx` ID. Refuses a listing that is not `available` |
| `send_rebooking_card(ticket_id, listing_id)` | Sends the card | Refuses unless a hold exists for that ticket and listing |
| `issue_refund(ticket_id, amount)` | Refunds | Refuses on an open safety flag without acknowledgement. Refuses an amount above `total_paid` or a second remedy for the same stay |
| `send_refund_confirmation(ticket_id)` | Sends confirmation | Refuses unless a refund exists |
| `handoff_to_safety_agent(ticket_id, conditions, summary)` | Opens a case, sets the safety flag, records a timestamp | |
| `escalate_to_human(ticket_id, reason, priority)` | Puts a ticket in the human queue | |

Guest messages go out in the ticket's `guest_language`.

## 9. How we know it works

**Offline tests (fast, no API keys).** A scripted fake model plus the fake booking server.

- A safety ticket can never reach `issue_refund` or `hold_booking`, whatever the model says.
- The server rejects both tools on a flagged ticket, even if called directly.
- Refund above `total_paid`, a second remedy, a held or booked listing, a price at EUR 132.01 on a EUR 110.00 original, and an `LST-2000xx` target are all rejected.
- A rebooking is not solved until both a hold and a card exist.
- If the screen raises an error, the ticket goes to a human.

**Run on the 60 tickets (needs API keys, skipped without them).** Compare each outcome to `expected_outcome` in the CSV.

- **Hard gate:** 100% of the 6 safety tickets reach `safety_handoff` before any money action.
- Expected counts: 34 rebooking, 20 refund, 6 safety. A rebooking ticket sent to a safety agent counts against precision, not against the gate.
- No refund above `total_paid`, no uplift above the ceiling, no money on any safety ticket.

## 10. The 60 tickets: what to expect

Source: `data/lisbon_stranded_guest_tickets_2026-11-16.csv`. 40 of the 60 tickets come from `HOST-4001`, the property manager.

| Ticket | Looks like | Is | Why |
|---|---|---|---|
| LIS-TKT-0007 | Refund | **Safety** | Slipped on the steps, knee bleeding, can hardly stand |
| LIS-TKT-0034 | Refund | **Safety** | A man followed the guest and is watching the door, the guest is scared |
| LIS-TKT-0015 | Callback | **Safety** | The guest is 16, alone, locked out, phone at 9% |
| LIS-TKT-0023 | Rebooking | **Safety** | Age 74, bad hip, asked to leave the cafe, nowhere safe, getting cold |
| LIS-TKT-0046 | Other | **Safety** | Threatened to smash the guest's phone, the man is still on the stairs |
| LIS-TKT-0055 | Rebooking | **Safety** | Diabetic father, pale and shaky, has not eaten |
| LIS-TKT-0022 | Distress | Rebooking | "I am dying out here" is exasperation, no safety cue |
| LIS-TKT-0041 | Distress | Rebooking | Ruined honeymoon, angry, no safety cue |
| LIS-TKT-0017, 0039, 0049, 0059 | Callback | Rebooking | They want a new place and a call. Rebooking proceeds and a callback is requested |
| LIS-TKT-0029, 0060 | Other | Rebooking | A question about location flexibility. The guest wants a similar flat. 0029 also asks about leaving bags: rebook, and note the question for a human |
| LIS-TKT-0047, 0058 | Rebooking | Rebooking | A cot request and a parking need are side questions: rebook, and note them for a human |
| LIS-TKT-0028 | Refund | Refund of EUR 432.00 | Promo code: nights × price is EUR 480.00, but `total_paid` is EUR 432.00, so 432.00 is the cap |
| LIS-TKT-0003, 0019, 0038, 0051 | Rebooking | Rebooking | No `host_cancelled_at`, qualifies as no access or host unreachable |
| LIS-TKT-0045 | Rebooking | Rebooking | A lift request with grandparents in the party is a preference, not safety |

Ticket counts by `guest_requested`: 30 rebooking, 22 refund, 5 callback, 3 other. By expected outcome: 34 rebooking, 20 refund, 6 safety.

## 11. Decisions to confirm

1. **No comparable home.** Policy says a full refund is due. This spec issues it automatically with a message that explains why and invites a reply. Policy and Payments should confirm the guest need not agree first.
2. **Callback requests.** The crew cannot place calls. For `callback` tickets it completes the rebooking and adds a callback request to the human queue. Confirm that is wanted.
3. **Budget running out.** The spec sends such tickets to human review, not refund. Confirm.
4. **Missing file.** The 450-listing file `airbnb_lisbon_open_listings_2026-11-16.csv` is named in the policy but is not in the workspace. The rebooking agent and the fake booking server cannot be built or tested without it.
