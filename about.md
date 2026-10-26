---
layout: page
title: About
permalink: /about/
intro: From running storage to building what runs it.
---

I started as a storage administrator. Level one. Tickets, break-fix, provisioning, the night shift — learning the trade from underneath, which is still the best way I know to learn it.

What became obvious fairly early is that the estate grows faster than the team does. The business wants to scale, so the infrastructure scales, and nobody scales the operations alongside it. At some point the ceiling is not the hardware or the vendor — it is how much a person can hold in their head and do by hand. Manual operations stop being inefficient and start being the actual limit.

So I began building on top of the vendors instead of just working inside them. It started small: scripts to take the repetitive work out of the day. Those turned into services, because a script someone has to remember to run is a script that doesn't get run. Services turned into a set of them, each owning one part of the estate. And those eventually turned into a single platform sitting over the lot.

That is what I work on now. One front door for every team that touches storage — search across the filesystem estate, backup history and failure triage, deletion forensics, capacity across several vendors at once, alerts wired straight through to tickets. The supporting and troubleshooting teams stop needing to know which of a dozen consoles holds the answer, or which engineer to ask. They ask the platform.

AI is the current rung of the same ladder, not a different ladder. It lets me build and integrate at a pace that was not available to me five years ago, and it has become part of the product too — natural-language queries across storage, backup and audit data, so someone who doesn't know the schema can still get an answer. I'm deliberate about that part: models sit behind an abstraction so one can be swapped for another by configuration, anything a user can influence is hardened against prompt injection, output is validated against a schema, and there's a deterministic fallback so the feature degrades instead of failing when a provider is unavailable.

Some things I learned early and still hold.

- An upstream outage returns an error, never an empty result. In a storage estate, "no files found" and "no owner" are answers people act on — a silent failure that renders as either one is worse than a page admitting it is broken.
- The thing that worked at one size stops working at the next, and you find out in production. Most capacity problems are that shape. So is most code.
- Guardrails go in at several independent layers, on the assumption that any one of them will eventually be wrong.

## Why this site exists

Almost none of my day-to-day work is publishable. So the rule here is simple: take the pattern, rebuild it clean in public with synthetic data, and write about the rebuilt thing. Every repository linked from here runs on data invented for the purpose — no employer, no estate, no hostnames. If a piece is worth writing, it is worth making reproducible by someone who has never seen my environment.

I'm in London.

[GitHub](https://github.com/svivekiyer) · [LinkedIn]({{ site.linkedin_url }})
