# Stranded Guest Support Constitution

Rules that override every other document in this repo. If the spec, a prompt, or a model's output disagrees with this file, this file wins.

## 1. A person in danger comes before money

- Any ticket showing one of the five safety conditions (`injury_or_medical_need`, `threat_or_violence`, `nowhere_safe_to_wait`, `unaccompanied_minor`, `stated_fear_of_person`) goes to a live Trust and Safety agent.
- On such a ticket the crew's only allowed booking action is `handoff_to_safety_agent`. `issue_refund` and `hold_booking` are forbidden until a live agent has taken the case.
- This holds even when the guest asks only for a refund and never uses the word "safety".
- After the hand-off the crew stops. The live safety agent decides any refund or booking.

## 2. The rule is enforced by the system, not by a prompt

- A deterministic safety screen runs on every ticket before any other agent sees it.
- The booking server's `issue_refund` and `hold_booking` read the ticket's safety flag and hand-off acknowledgement from the server's own state and refuse when the flag is set and no live agent has acknowledged. They never trust an argument supplied by an agent.
- A model being told to be careful does not count as enforcement.

## 3. When unsure, a human decides

- If the screen errors, times out, or cannot decide, the ticket goes to a human. It never proceeds to money or booking actions by default.
- If a specialist cannot finish a ticket, it hands it to a human with a written reason. It does not guess.

## 4. Money limits are hard limits

- A refund never exceeds the ticket's `total_paid`.
- A guest gets one remedy per stay: a rebooking or a refund, never both for the same nights.
- A rebooked home never costs more than 120% of the original `nightly_price`. Nobody can approve more on 16 November 2026.

## 5. Safe data only

- This repo is public. It holds only the provided incident tickets, which are synthetic, and the policy. No real guest data is ever added, because anything committed is visible to everyone.
- API keys live in environment variables, never in code or in git.
