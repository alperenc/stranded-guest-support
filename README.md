# stranded-guest-support

A team of AI helpers that rescues the 60 guests stranded in Lisbon on 16 November 2026. Each guest chat ends in a new home, a refund, or a live safety agent. Anyone in danger reaches a person before any money moves.

Python 3.12 or 3.13. The package lives in `src/stranded_guest_support` and is empty so far.

## Read first

- `.specify/memory/constitution.md`: the rules that override everything else
- `specs/001-stranded-guest-support/spec.md`: ticket types, specialists, tools, escalation
- `data/`: the refund and rebooking policy and the 60 tickets

## Set up

```bash
uv venv --python 3.13
uv pip install -e .
```

API keys go in environment variables or a local `.env`. Git ignores `.env`.

## Status

Spec drafted, code not started. The 450-listing file the policy names is still missing.
