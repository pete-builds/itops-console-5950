# Personas: itops-console

**Stage**: INCEPTION - User Stories, Part 2
**Source**: `docs/vision.md` (Target users), `requirements.md` section 2
**Plan decision**: Question 4 = A (the three vision personas only)

---

## P1: Desktop support technician

| Item | Detail |
|---|---|
| Role | Front-line support. Gets asked "why is this laptop failing compliance?" and fixes it |
| Goal | Find out quickly why one device fails, and fix it safely, without opening four or five consoles |
| Context | Works one device at a time, often from a ticket that names a hostname, a serial number or a person. Comfortable in a terminal |
| Pain points | The dashboard says "failed" but not why; the same device has different names in each system; "no data" and "stale" look like failures but need different fixes; unsure whether a remote fix is safe to run |
| Surfaces | Terminal chat; approval web page |
| Needs from the console | Plain-language cause with evidence; a fix from the reviewed catalog or a clear "no reviewed fix"; an approval page that shows every target and exactly what will run; a demo that is quick to set up |
| Trust boundary | Approves or declines actions. Never writes scripts or changes the baseline |
| Also covers (Q4 = A) | Setting up and running the demo, and running the scenario suite |

## P2: Endpoint engineer

| Item | Detail |
|---|---|
| Role | Owns endpoint management tooling and fleet-wide fixes |
| Goal | Spend effort where it certifies the most devices |
| Context | Thinks in policies, profiles and groups, not single devices. Maintains the baseline policy file values with the baseline owners |
| Pain points | Per-check dashboards hide which device is one fix from certified; devices that stopped reporting or are retired inflate the failure counts |
| Surfaces | Fleet summary web page; baseline policy file |
| Needs from the console | Rankings by check and by remediation; separate lists for stopped-reporting, class-unknown and clean-up devices; deterministic numbers, with the agent's narrative clearly labeled |

## P3: Compliance or team lead

| Item | Detail |
|---|---|
| Role | Reviews what the team did and why |
| Goal | Confirm that every action was recommended on evidence and approved by a person |
| Context | Reads after the fact; does not run chats or approve actions |
| Pain points | Agent actions are hard to audit when tool calls and approvals are hidden |
| Surfaces | Transcript viewer web page |
| Needs from the console | Every session listed; every tool call, scoring step, approval answer and engine audit reference visible; records nobody can quietly edit |
