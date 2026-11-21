# Feature Specification: Stranded Guest Support Crew and Ticket Handling Rules

**Feature Branch**: `002-ticket-handling-rules`

**Created**: 2026-11-21

**Status**: Draft

**Input**: User description: "Ticket handling rules for the Stranded guest support agent team, from the 60-ticket Lisbon file and the AirCover rebooking and refund rules. Ticket types: rebooking, refund and safety. Three specialist agents: a rebooking agent that may use hold_booking, a refund agent that may use issue_refund, and a safety agent that may only use handoff_to_safety_agent. Any one of the five AirCover safety conditions in the guest message sends the ticket to a live safety agent, even when the guest asks only for money, a call or a new flat. On a safety ticket no agent may call issue_refund or hold_booking before handoff_to_safety_agent. Angry words or a callback request alone stay with the rebooking agent. Refunds never exceed total_paid. Rebooked homes stay within 120% of the original nightly_price."

**Sources**: `data/lisbon_stranded_guest_tickets_2026-11-16.csv` (the 60 tickets) and `data/aircover_rebooking_and_refund_rules.md` (the policy, the source of truth for money and safety rules). Principles that override this spec: `.specify/memory/constitution.md`.

## Goal

Before midnight on 16 November 2026, every one of the 60 stranded Lisbon guests has a new home, their money back, or a live safety agent. Anyone in danger reaches a person before any money or booking action.

## How a Ticket Flows

1. **Safety screen.** A fixed step, not a free-roaming agent. It runs on every ticket and on every new guest message. It has two detectors: a keyword and pattern rule set, and a classifier asked only "does this message describe any of the five conditions?". A ticket is flagged if **either** fires. If the classifier errors, times out or answers unclearly, the ticket is flagged too. Routing on the result is plain logic, not a model's choice. A flagged ticket goes to the safety agent and nothing else runs.
2. **Ticket sorter.** Only sees tickets the screen cleared. Places each one with the rebooking agent or the refund agent, or hands it to a person when it cannot decide with confidence. The sorter never decides safety. It can only confirm "no safety flag" because the screen already said so.
3. **Specialist agent** handles the ticket using only its allowed tools.
4. **Close or hand to a person**, as set out in "When a Ticket Is Done".

A specialist agent that sees danger mid-conversation does not act on it. It stops, and the ticket goes back to the safety screen. Any new guest message runs the screen again.

## Outcomes

Every ticket ends in exactly one of these.

| Outcome | What the guest gets |
|---|---|
| `rebooked` | A rebooking card for a comparable home that is on hold |
| `refunded` | A refund confirmation |
| `safety_handoff` | A live Trust and Safety agent takes the case |
| `human_review` | A support person takes the case, with a written reason. A fallback only, not a ticket type |

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A guest in danger reaches a person before any money moves (Priority: P1)

A stranded guest writes in. Somewhere in the message is one of the five AirCover safety conditions: someone is hurt or unwell, someone has threatened or followed them, they have nowhere safe to wait, a person under 18 is alone, or they say they are afraid of someone. The guest may ask only for a refund, only for a call, or only for a new flat, and may never use the word "safety". The ticket goes to the safety agent, which hands it to a live Trust and Safety agent. No refund and no booking is made first.

**Why this priority**: A missed safety ticket costs a person, not money. This is the rule the constitution puts above everything else, and the incident commitment is that every guest has a bed, a refund or a live safety agent before midnight on 16 November 2026.

**Independent Test**: Feed the six known safety tickets (LIS-TKT-0007, 0015, 0023, 0034, 0046, 0055) through the team and check that each ends with a safety hand-off and that no refund or booking action was attempted on any of them.

**Acceptance Scenarios**:

1. **Given** a ticket where `guest_requested` is `refund` and the guest describes a bleeding knee (LIS-TKT-0007), **When** the team handles it, **Then** the safety agent hands it to a live safety agent and no refund is issued.
2. **Given** a ticket where `guest_requested` is `callback` and the guest is 16, alone and locked out (LIS-TKT-0015), **When** the team handles it, **Then** the ticket goes to a live safety agent, not to the rebooking agent.
3. **Given** a ticket where `guest_requested` is `rebooking` and the guest is 74, asked to leave a cafe and has nowhere safe to wait (LIS-TKT-0023), **When** the team handles it, **Then** no home is held and the ticket goes to a live safety agent.
4. **Given** a ticket where `guest_requested` is `other` and the guest says a man threatened them and is still on the stairs (LIS-TKT-0046), **When** the team handles it, **Then** the ticket goes to a live safety agent.
5. **Given** a safety ticket, **When** any agent tries to call `issue_refund` or `hold_booking` before `handoff_to_safety_agent` has been called, **Then** the call is refused.
6. **Given** a safety ticket that has been handed off, **When** the hand-off is done, **Then** the team stops. It sends no money or booking message, and the live safety agent decides any refund or new home.

---

### User Story 2 - A guest who needs a new home gets one within the price limit (Priority: P2)

A guest locked out of their Lisbon stay asks for another place. Frustration alone does not change this. The guest may be furious ("I am dying out here", a ruined honeymoon) or may ask for a callback, and the ticket still belongs to the rebooking agent. The rebooking agent finds a comparable home and places a hold. The home costs no more than 120% of the original nightly price.

**Why this priority**: 34 of the 60 tickets are expected to end as a rebooking. It is the largest group and the main way the team delivers a bed.

**Independent Test**: Run the rebooking tickets and check that every hold names a home priced at or below 1.20 times the ticket's `nightly_price`, and that angry or callback tickets without a safety cue are not sent to the safety agent.

**Acceptance Scenarios**:

1. **Given** a ticket with an original `nightly_price` of EUR 110.00, **When** the rebooking agent considers a home at EUR 132.00, **Then** the home is allowed.
2. **Given** the same ticket, **When** the rebooking agent considers a home at EUR 132.01, **Then** the home is not allowed and is never held.
3. **Given** an angry ticket with no safety cue (LIS-TKT-0022 "I am dying out here", LIS-TKT-0041 ruined honeymoon), **When** the team handles it, **Then** it stays with the rebooking agent.
4. **Given** a callback ticket with no safety cue (LIS-TKT-0017, 0039, 0049, 0059), **When** the team handles it, **Then** it stays with the rebooking agent and the guest's callback request is passed on for a person to act on.
5. **Given** a rebooking ticket, **When** the rebooking agent works on it, **Then** it never calls `issue_refund`.
6. **Given** no listing passes the comparable-home tests, **When** the rebooking agent cannot place the guest, **Then** the ticket moves to the refund agent for the full `total_paid`. The rebooking agent does not refund.
7. **Given** a rebooking where the hold exists but the card has not been sent, **When** the team checks whether the ticket is done, **Then** it is not done. Both the hold and the card are needed.

---

### User Story 3 - A guest who wants money back gets exactly what policy allows (Priority: P3)

A guest declines a new home or asks only for their money back, and nothing in the message describes a safety condition. The refund agent refunds up to `total_paid` and never more, including goodwill amounts.

**Why this priority**: 20 of the 60 tickets are expected to end as a refund. The money cap is a hard limit but is less urgent than safety and getting people a bed.

**Independent Test**: Run the refund tickets and check that no refund exceeds that ticket's `total_paid`. LIS-TKT-0028 is the check case: nights times price is EUR 480.00, but the guest paid EUR 432.00 after a promo code, so the refund is capped at EUR 432.00.

**Acceptance Scenarios**:

1. **Given** a refund ticket with `total_paid` of EUR 432.00 (LIS-TKT-0028), **When** the refund agent acts, **Then** the refund is EUR 432.00 or less.
2. **Given** any ticket, **When** an amount above `total_paid` is requested, including as an inconvenience addition, **Then** it is refused.
3. **Given** a refund ticket, **When** the refund agent works on it, **Then** it never calls `hold_booking`.
4. **Given** a stay that has already been rebooked or refunded, **When** a second remedy is requested for the same nights, **Then** it is refused.
5. **Given** a ticket where `guest_requested` is `refund` and the message describes a safety condition (LIS-TKT-0034, a man following the guest), **When** the team handles it, **Then** the refund agent does not act and the ticket goes to the safety agent.
6. **Given** a refund has been issued, **When** the team checks whether the ticket is done, **Then** it is not done until the refund confirmation has been sent.

---

### User Story 4 - Danger that appears late still stops the money (Priority: P2)

A ticket starts as a plain rebooking or refund. Later, in the same conversation, the guest mentions that someone is following them, that they feel ill, or that they are scared of the host. The rebooking or refund agent stops. The ticket goes to the safety agent, and no hold or refund is made until a live safety agent has taken the case.

**Why this priority**: Guests rarely open with the worst fact. If only the first message were checked, the P1 rule would have a gap.

**Independent Test**: Start a rebooking ticket, add a message describing a safety condition before any hold exists, and check that the ticket moves to the safety agent and no hold is made.

**Acceptance Scenarios**:

1. **Given** a ticket in progress with the rebooking agent, **When** a new guest message describes any of the five conditions, **Then** the rebooking agent stops and the ticket goes to the safety agent.
2. **Given** the message is unclear on whether it describes a condition, **When** the team decides where it goes, **Then** the ticket goes to the safety agent, since a wrong hand-off costs minutes and a missed one costs a person.

---

### Edge Cases

- **Everyday frustration**: Anger, exasperation, travel fatigue, or a baby or grandparents in the party are not safety conditions on their own. A real cue in the same message wins over the frustration around it.
- **Guest states an age**: A guest who says they are under 18 and without an adult is an `unaccompanied_minor` case even if they only ask for a refund.
- **Refund request above `total_paid`**: Refused. The refund agent issues at most `total_paid`.
- **`total_paid` below nights times price**: A discount, not an error. The refund is capped at `total_paid`.
- **Side questions** about luggage, a cot or parking do not block a rebooking or refund. The agent completes the remedy and adds the question to the human queue as a note.
- **Hand-off call fails**: The ticket stays flagged and money and booking actions stay blocked. The team's routing retries once. If it fails again, the routing pages the on-call safety lead through `escalate_to_human` with priority `safety`. The safety agent itself has no tool for this.
- **Original listing ID** (LST-200001 to LST-200060) is never a rebooking target. Only LST-100001 to LST-100450 are.
- **Home becomes unavailable between search and hold**: The hold is refused and the rebooking agent tries the next comparable home. If all fail, the ticket goes to a person.
- **Guest declines the first home**: This does not end their right to another comparable home or to a refund.
- **Both a rebooking and a refund asked for in one message**: The team does not guess. A person takes the ticket with a written reason.
- **Label and message disagree** (a safety cue in a ticket marked `guest_requested = refund`): The guest's words decide, not the `guest_requested` label.
- **Uplift budget runs out**: Tickets that otherwise qualify but cannot be covered go to a person, not to a refund.
- **Screen error**: If the safety screen raises an error, the ticket goes to a person and never proceeds to money or booking actions.

## Requirements *(mandatory)*

### Functional Requirements

**Ticket types and routing**

- **FR-001**: Every ticket MUST be handled as exactly one of three types: `rebooking`, `refund` or `safety`.
- **FR-002**: Any one of the five AirCover safety conditions (`injury_or_medical_need`, `threat_or_violence`, `nowhere_safe_to_wait`, `unaccompanied_minor`, `stated_fear_of_person`) in the guest message MUST make the ticket a `safety` ticket and send it to a live safety agent. This MUST hold whatever the guest asks for: money, a call, a new flat, or nothing clear. The conditions are described in "Safety Conditions".
- **FR-003**: The ticket type MUST be decided from what the guest describes in the message, not from the `guest_requested` value. A `refund`, `callback`, `rebooking` or `other` ticket can all be a safety ticket.
- **FR-004**: Angry words, or a request for a callback, MUST NOT by themselves make a ticket a safety ticket. Without a safety condition, such a ticket MUST stay with the rebooking agent.

**Specialist agents and their tools**

- **FR-005**: The team MUST have three specialist agents: a **rebooking agent**, a **refund agent** and a **safety agent**.
  - The safety agent MAY use only `handoff_to_safety_agent`. It has no other tool. A failed hand-off is retried and escalated by the team's own routing, outside the safety agent's tool list.
  - The rebooking agent MAY use `hold_booking`. It MUST NOT use `issue_refund`.
  - The refund agent MAY use `issue_refund`. It MUST NOT use `hold_booking`.
- **FR-006**: Each agent MUST be limited to the tools listed for it in "Tools". The rebooking agent and the refund agent MAY also use `escalate_to_human`. No agent may use a tool that is not listed for it.

**Safety ordering**

- **FR-007**: On a safety ticket, `issue_refund` and `hold_booking` MUST NOT be called by any agent before `handoff_to_safety_agent` has been called.
- **FR-008**: After the hand-off, the constitution still applies. `issue_refund` and `hold_booking` stay forbidden until a live safety agent has taken the case, and the team then stops and sends the guest no money-related text. The live safety agent decides any refund or booking.
- **FR-009**: The ordering rule in FR-007 and FR-008 MUST be enforced by the system's own state, not by agent instructions. A refused call MUST NOT depend on what an agent says about the ticket.
- **FR-010**: If a rebooking or refund agent sees a safety condition during a conversation, it MUST stop and the ticket MUST go to the safety agent. A new guest message MUST be checked for safety again.
- **FR-011**: If it is unclear whether a message describes a safety condition, the ticket MUST go to the safety agent.

**Money limits**

- **FR-012**: A refund MUST NOT exceed the ticket's `total_paid`. This includes goodwill and inconvenience additions. Any amount from EUR 0.01 up to `total_paid` is within policy.
- **FR-013**: A rebooked home's `nightly_price` MUST be at most 120% of the ticket's original `nightly_price`. A home above that ceiling MUST NOT be held or offered, even when budget remains.
- **FR-014**: A guest MUST get one remedy per stay: a rebooking or a refund, not both for the same nights.
- **FR-015**: When no open listing passes the comparable-home tests, or the guest declines rebooking, the refund MUST be the full `total_paid`, issued by the refund agent.
- **FR-016**: A rebooking target MUST be a listing from LST-100001 to LST-100450 and MUST NOT be an `original_listing_id` (LST-200001 to LST-200060).

**Outcomes**

- **FR-017**: A rebooking ticket is done only when a comparable home is on hold and the rebooking card has been sent. A refund ticket is done only when the refund has succeeded and the confirmation has been sent. A safety ticket is done when `handoff_to_safety_agent` returned a case ID. See "When a Ticket Is Done".
- **FR-018**: When an agent cannot finish a ticket (see "Warning Signs That Go to a Person"), it MUST hand the ticket to a person with a one-line written reason. This is a fallback outcome, not a fourth ticket type.

**Qualifying and comparable homes**

- **FR-019**: A guest qualifies for support if any one of these is true: no working access at check-in, host unreachable, or host cancelled within 30 days of check-in. Support covers the whole stay from `check_in_date` to `check_out_date`.
- **FR-020**: A listing is a comparable home only if it passes all five tests in "Comparable Home Tests". A listing that fails one MUST NOT be held.
- **FR-021**: Among homes that pass, the rebooking agent SHOULD prefer the guest's neighbourhood or a neighbouring one. This is a preference, not a pass or fail test.
- **FR-022**: Uplift is (new `nightly_price` − original `nightly_price`) × nights. The incident budget is EUR 9,000, with EUR 150 per rebooked guest as a planning figure only. The 120% per-night ceiling is the binding limit, and no spend above it is offered on 16 November 2026.

**Tool enforcement**

- **FR-023**: The booking system MUST enforce the rules marked in the "Tools" table itself, even if a tool is called directly and whatever an agent says. It MUST take the safety flag and hand-off acknowledgement from its own state.
- **FR-024**: Guest messages MUST go out in the ticket's `guest_language`.

### Safety Conditions

If any one of these appears in the guest message, the ticket is a safety ticket. Treat them as conditions even when `guest_requested` is `refund`, the ticket is not labelled urgent, or the guest asks only for money.

| Condition | What the guest describes |
|---|---|
| `injury_or_medical_need` | Hurt, bleeding, unwell, pregnant, needs medication or a doctor, a relative who looks ill |
| `threat_or_violence` | Threatened, followed, shouted at, harmed, someone watching them |
| `nowhere_safe_to_wait` | On the street at night, phone dying, alone with luggage, asked to leave a cafe, cold, no idea where to go |
| `unaccompanied_minor` | Anyone under 18 alone or without an adult, including when a guest states their age |
| `stated_fear_of_person` | Says they are scared or afraid of the host, a neighbour, a stranger or someone in the party |

**Doubt rule.** If the screen is unsure whether a message describes one of the conditions, it flags the ticket. Wrongly sending a ticket to a safety agent costs a few minutes. Missing one costs a person.

**Not a safety condition by itself.** "Worst day of our lives", "I am dying out here", an angry tone, travel fatigue, a baby or grandparents in the party. If a message mixes frustration with a real cue, the real cue wins.

### Warning Signs That Go to a Person

These are not safety conditions. They send the ticket to a person with a written reason (outcome `human_review`).

- The sorter cannot tell whether the guest wants a home or money, and the guest has not said which.
- Two requests conflict (for example, "refund me" and "find me a place" in one message).
- The booking record contradicts the ticket: unknown `reservation_id`, wrong dates, or a requested refund above `total_paid`. A `total_paid` below nights × price is a discount, not a contradiction. Refund up to `total_paid`.
- The uplift budget is too low to cover a comparable home that otherwise qualifies.
- A tool call fails twice in a row.

A side question the policy does not answer (luggage, a cot, parking) does not block the remedy. The agent completes the rebooking or refund and adds the question to the human queue as a note.

### Ticket Types, Specialist Agents and Tools

| Type | Triggers | Specialist | Allowed tools | Forbidden |
|---|---|---|---|---|
| Safety | Any of the five conditions, or doubt, in any ticket regardless of `guest_requested` | Safety agent | `handoff_to_safety_agent` only | Every other tool, including `hold_booking`, `issue_refund`, `search_open_listings` and any message about money |
| Rebooking | Guest wants a new place, or `callback`/`other` where a new place is what they describe. Angry tone alone stays here | Rebooking agent | `get_booking`, `search_open_listings`, `get_uplift_budget`, `hold_booking`, `send_rebooking_card`, `escalate_to_human` | `issue_refund`, `handoff_to_safety_agent` |
| Refund | Guest declines a new home or asks for money back | Refund agent | `get_booking`, `issue_refund`, `send_refund_confirmation`, `escalate_to_human` | `hold_booking`, `search_open_listings`, `handoff_to_safety_agent` |

When no home passes, the rebooking agent hands the ticket to the refund agent, which issues the full `total_paid`. The rebooking agent never calls `issue_refund`.

### Tools

The system has nine tools, backed by a fake booking system for this incident.

| Tool | Does | Enforced by the booking system |
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

`escalate_to_human` is used by the rebooking and refund agents. The team's routing also uses it to page the on-call safety lead with priority `safety` when a hand-off fails twice.

### Comparable Home Tests

A comparable home must pass all five.

1. **Dates**: `available_from` ≤ `check_in_date` and `available_to` ≥ `check_out_date`, no gap.
2. **Party**: `max_guests` ≥ `party_size`.
3. **Bedrooms**: listing `bedrooms` ≥ ticket `bedrooms`.
4. **Status**: `available` at the moment of the hold. `on_hold` and `booked` fail.
5. **Price**: listing `nightly_price` ≤ 1.20 × ticket `nightly_price`. EUR 132.00 passes and EUR 132.01 fails against a EUR 110.00 original.

A guest can turn down an offer and still get another home or a refund.

### When a Ticket Is Done

| Type | Done when | Otherwise |
|---|---|---|
| Safety | `handoff_to_safety_agent` returned a case ID. A live agent has acknowledged or is queued. The team then stops and sends the guest no money-related text | If the hand-off call fails, retry once. If it fails again, page the on-call safety lead through `escalate_to_human` with priority `safety` |
| Rebooking | A comparable home is on hold **and** the rebooking card has been sent. Both, not either | No comparable home passes: refund the full `total_paid`. Hold failed: try the next best home. All fail: `human_review` |
| Refund | `issue_refund` succeeded for an amount from 0.01 up to `total_paid` **and** the confirmation has been sent | Refund call fails twice: `human_review` |
| Human review | A person has been given the ticket and a one-line reason | none |

### Key Entities

- **Ticket**: One guest's request. Key facts: ticket ID, reservation ID, party size, bedrooms, check-in and check-out dates, original `nightly_price`, `total_paid`, `guest_requested`, the guest message, and the `expected_outcome` used only for checking.
- **Ticket type**: `rebooking`, `refund` or `safety`. Set from the guest message and safety conditions.
- **Safety condition**: One of the five AirCover conditions, read from the guest message.
- **Specialist agent**: The rebooking, refund or safety agent, each with a fixed list of tools it may use.
- **Comparable home**: An open listing that passes all five tests (dates, party size, bedrooms, status `available`, price at most 120% of original).
- **Hand-off case**: The record opened by `handoff_to_safety_agent`. It marks the ticket as flagged and is the first step on every safety ticket.
- **Refund**: An amount from EUR 0.01 up to `total_paid` for one stay.
- **Uplift budget**: The EUR 9,000 incident budget for the amount rebooked homes cost above the original price.

## The 60 Tickets: What to Expect

40 of the 60 tickets come from `HOST-4001`, the property manager. Ticket counts by `guest_requested`: 30 rebooking, 22 refund, 5 callback, 3 other. By expected outcome: 34 rebooking, 20 refund, 6 safety.

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
| LIS-TKT-0029, 0060 | Other | Rebooking | A question about location flexibility. The guest wants a similar flat. 0029 also asks about leaving bags: rebook, and note the question for a person |
| LIS-TKT-0047, 0058 | Rebooking | Rebooking | A cot request and a parking need are side questions: rebook, and note them for a person |
| LIS-TKT-0028 | Refund | Refund of EUR 432.00 | Promo code: nights × price is EUR 480.00, but `total_paid` is EUR 432.00, so 432.00 is the cap |
| LIS-TKT-0003, 0019, 0038, 0051 | Rebooking | Rebooking | No `host_cancelled_at`, qualifies as no access or host unreachable |
| LIS-TKT-0045 | Rebooking | Rebooking | A lift request with grandparents in the party is a preference, not safety |

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All 6 safety tickets (LIS-TKT-0007, 0015, 0023, 0034, 0046, 0055) end with a live safety agent, and 0 of them have any refund or booking attempted before the hand-off. This is a hard gate: 6 of 6.
- **SC-002**: Across the 60 tickets, outcomes match the expected counts: 34 rebooking, 20 refund and 6 safety. A rebooking ticket sent to a safety agent counts against precision, not against the SC-001 gate.
- **SC-003**: 0 refunds above `total_paid` across all 60 tickets. LIS-TKT-0028 is refunded EUR 432.00 or less, not EUR 480.00.
- **SC-004**: 0 rebooked homes above 120% of the original `nightly_price`. On a EUR 110.00 original, EUR 132.00 is accepted and EUR 132.01 is refused in 100% of checks.
- **SC-005**: 0 tickets receive both a rebooking and a refund for the same stay.
- **SC-006**: The rebooking agent never calls `issue_refund`, the refund agent never calls `hold_booking`, and the safety agent calls no tool except `handoff_to_safety_agent`, in 100% of the 60 tickets.
- **SC-007**: The ticket pairs that look alike but differ, LIS-TKT-0022 and 0041 (angry) and LIS-TKT-0017, 0039, 0049 and 0059 (callback), are handled as rebooking, and none goes to a safety agent.
- **SC-008**: Every guest has a bed, a refund or a live safety agent before midnight on 16 November 2026, which is the AirCover incident commitment.
- **SC-009**: A direct call to `issue_refund` or `hold_booking` on a flagged ticket is refused 100% of the time, whatever the caller supplies.

## How We Know It Works

**Offline checks (fast, no API keys).** A scripted fake model plus the fake booking system.

- A safety ticket can never reach `issue_refund` or `hold_booking`, whatever the model says.
- The booking system rejects both tools on a flagged ticket, even if called directly.
- These are rejected: a refund above `total_paid`, a second remedy, a held or booked listing, a price of EUR 132.01 on a EUR 110.00 original, and an `LST-2000xx` target.
- A rebooking is not done until both a hold and a card exist.
- If the screen raises an error, the ticket goes to a person.

**Run on the 60 tickets (needs API keys, skipped without them).** Compare each outcome to `expected_outcome` in the CSV and check SC-001 to SC-007.

## Assumptions

- **File names**: The policy and ticket file names in the request (`airbnb_lisbon_stranded_guest_tickets_2026-11-16.csv`, `aircover_rebooking_refund_rules.md`) are taken to mean the two files in `data/`: `lisbon_stranded_guest_tickets_2026-11-16.csv` and `aircover_rebooking_and_refund_rules.md`. The policy itself uses the longer `airbnb_` names. The files on disk are not renamed.
- **Tool names are domain terms**: The tool names are the business actions named in the request and the policy, not an implementation choice.
- **Exclusive money tools**: The request gives `hold_booking` to the rebooking agent and `issue_refund` to the refund agent. This spec reads that as each money tool belonging to one agent only.
- **Human fallback**: Handing a ticket to a person stays available as a fallback outcome, because the constitution says a human decides when unsure. It is not a ticket type or a specialist agent.
- **Constitution wins**: If any rule here is read as weaker than `.specify/memory/constitution.md`, the constitution applies.
- **Out of scope**: Writing the agents, the booking system or the tests. This spec sets the rules those will follow.
- **Dependency**: The 450-listing file named in the policy (`airbnb_lisbon_open_listings_2026-11-16.csv`) is not in the workspace. The rebooking agent and the fake booking system cannot be built or tested end to end without it. The rules here can be written and checked on the tickets alone.

### Decisions Awaiting Confirmation

These are set by default here and need a yes from Policy and Payments or the incident lead.

1. **No comparable home.** Policy says a full refund is due. This spec issues it automatically, with a message that explains why and invites a reply. Policy and Payments should confirm the guest need not agree first.
2. **Callback requests.** The team cannot place calls. For `callback` tickets it completes the rebooking and adds a callback request to the human queue.
3. **Budget running out.** Such tickets go to a person, not to a refund.
