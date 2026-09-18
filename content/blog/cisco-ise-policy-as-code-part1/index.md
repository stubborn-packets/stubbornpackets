---
title: "Cisco ISE Policy-as-Code Migration - Part 1"
date: 2026-09-16
draft: false
description: "Brownfield ISE, no change window. Phase 0 is a repo that cannot talk to ISE. Phase 1 is GET-only so the non-standard names are visible."
categories: ["Terraform", "Cisco", "ISE"]
tags: ["Terraform", "Cisco", "ISE", "Policy-as-Code"]
cover:
  image: "ise-pac-part1-hero.jpg"
  alt: "Homelab laptop showing messy GUI names on the left and a Phase 0 repo plus Phase 1 export on the right"
---

# Cisco ISE Policy-as-Code Migration - Part 1

Cisco ISE is a beast when everything lives in the GUI. Fine if you are the only person who touches it. Ugly once a team is in the same Policy Sets page and nobody can answer who changed what, or whether the name on that authorization profile was ever a standard.

This series is me finding out whether Policy-as-Code on ISE 3.x is a real operating model or just a Cisco Live slide. I am guessing; it has been a while. I like tinkering, so I spun up an eval node in the homelab, seeded it with messy GUI objects on purpose, and used Grok AI to help break the work into phases that cannot smash the lab.

Part 1 is only Phase 0 and Phase 1. Nothing is applied. Terraform does not have a provider block yet.

The post is a snapshot. The repo will move as later phases land.

**Note:** *This is a personal homelab project. Prefixes, object names, and example policy shapes are fictional. They came from public posts on this topic and from what I think a brownfield lab should look like so the export is interesting. They are not copied from an internal standard and they are not names used by my employer. Nothing here is employer policy, employer architecture, or employer data.*

*Opinions are mine.*

---

## Github Repos

**cisco-ise** → [stubborn-packets/cisco-ise](https://github.com/stubborn-packets/cisco-ise)  

*The post is a snapshot and the repos are live and will be updated as I progress through the phases. Things you see in the repos may have items that have been updated since this post.*

## The Lab

| Piece | Choice |
|-------|--------|
| Node | ISE eval, standalone |
| Version | 3.3 patch 1 |
| Role in this repo | `environments/lab/` — the only write target, later |

A second eval node may show up as `environments/stage/`. There is no `prod/` folder in this checkout. Network Access only. No TACACS. Not an ISE installer. Those can be added later, but not needed for this PoC.

I loaded the lab with non-standard Policy Sets, AuthN/AuthZ rules, NADs, NDGs, conditions, and endpoint groups so the export would look like a brownfield node, not a greenfield demo.

## What this PoC is

Take an existing ISE and introduce Git-reviewed policy without renaming live objects in place.

Cutover that will span later posts:

1. Export what ISE already has
2. Create new objects that match the naming standard, beside the old ones
3. Move test traffic
4. Disable the old objects
5. Delete after a bake period

YAML under `policy/` is the review surface. CSV under `exports/` is observed state. Terraform becomes the writer in a later part, and only against lab.

*Bonus: once policy is in Git, a policy-set review is a file and a diff, not a pile of GUI screenshots.*

## Phase 0 — Plan (the phase that actually pays)

This was the longest stretch, and ISE was not even in the loop. I was on the couch after work with a notes file, trying to answer: if another engineer opened this repo in six months, could they tell a Policy Set from a NAD list from a dump I pulled last Tuesday?

That is why the folders are split and stay split.

| Plane | Folder | Question | Applied? |
|-------|--------|----------|----------|
| Desired policy | `policy/` | What should ISE look like? | Not yet |
| Inventory | `inventory/` | Nodes, VLANs, curated NAD list | Never in this PoC |
| Observed | `exports/` | What did the API return? | Generated, gitignored |
| Environment | `environments/lab/` | Which ISE, which state file? | Wiring only |
| Docs | `docs/` | Naming, file map, phases, decisions | — |

`inventory/` is the stuff *around* ISE. `exports/` is what the node admitted to. Mixing those two is how a brownfield name sneaks into Terraform later and becomes "desired state" by accident.

I also refused to put a Terraform provider in this phase. The second `required_providers` hits the repo, somebody (me) is going to run `plan` against a live node "just to see." Phase 0 exit criteria on purpose: clone it, open the files, nothing talks to ISE.

Naming ate more time than the folders. I wanted something a reviewer can lint, not another house style in a Confluence page. User-created names are `{PREFIX}-` plus kebab-case. Prefix stays uppercase.

| Prefix | Object |
|--------|--------|
| `PS-` | Policy set |
| `AN-` | Authentication rule |
| `AZ-` | Authorization rule |
| `EX-` | Exception rule |
| `AP-` | Allowed protocols |
| `PR-` | Authorization profile |
| `ACL-` | Downloadable ACL |
| `SGT-` | Security group |
| `CND-` | Library condition |
| `EIG-` | Endpoint identity group |

Pass: `PS-global-wired-8021x`. Fail: `Corp Users VLAN`, `ps-global-wired-8021x`, `PS_global_wired`.

ISE already has names it owns. I am not renaming `Default`, `Default Network Access`, or `Unknown` to make the spreadsheet pretty.

The lab is full of the fail column on purpose. Those objects stay where they are. I am not importing them as desired state, and I am not renaming them in place as the first write. New standard objects go in beside them. That decision is the whole reason Phase 1 is "go look" and not "go change."

## Phase 1 — read-only discovery

First useful night on the node: get the data, do not touch a policy.

`scripts/export_ise.py` is GET-only Python 3.12, `requests` plus a `.env`. I know `requests`. I have not used `ciscoisesdk` yet, and I did not want the first script in the repo to be a science project. The HTTP helper will not send PUT, POST, or DELETE. If I fat-finger a method, it should refuse.

ISE 3.x will also refuse you if you guess the wrong API. Network Access is split ([ERS Open API](https://developer.cisco.com/docs/identity-services-engine/latest/ers-open-api-ers-open-api/)):

| Surface | Path | This increment |
|---------|------|----------------|
| ERS | `/ers/config/...` | NADs, NDGs, authz profiles, allowed protocols, DACLs, SGTs, EIGs |
| OpenAPI | `/api/v1/policy/network-access/...` | Policy sets, AuthN/AuthZ rules, exceptions, library conditions |

I flipped the lab on under **Administration > System > Settings > API Settings**: ERS, OpenAPI, API Gateway. Then an account that can GET both. Miss one of those and you get a 403 that looks like a script bug.

I did not start with `--resource all`. I wanted to see one CSV, then a boring NAD list, then the scary NAD detail call, then policy sets.

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r scripts/requirements.txt
cp .env.example .env          # ISE_URL, ISE_USERNAME, ISE_PASSWORD
python scripts/export_ise.py --resource ndg
python scripts/export_ise.py --resource nads --no-details
python scripts/export_ise.py --resource nads
python scripts/export_ise.py --resource policy-sets
python scripts/export_ise.py                 # default: all
```

`--no-details`  is the list GET (name, id). Drop that flag on NADs and you pay for a second GET per device. The script then strips RADIUS / SNMP / TrustSec secret keys before it writes `exports/nads.csv`. The dump is useful. The shared secret is not a souvenir.

CSVs are gitignored. Do not hand-edit them. Open policy-sets.csv, look at the name column, hold it up against `docs/naming.md`. Everything that looks like the fail examples stays in `exports/`. That row is evidence, not a candidate for `policy/`.

That was the payoff from Phase 0. The naming table stopped being theoretical the first time the export showed me what I had actually clicked together in the GUI.

## Next

Part 2 is the linter: `lint/prefixes.yaml` against `policy/` and `inventory/`. A bad name should fail the PR before ERS ever sees it. Still no writes. The first Terraform apply is a later post on purpose. I want the next "this is fun" moment to be a failed lint, not a surprise object in the lab GUI.

---

*Personal homelab only. Not affiliated with my employer. Hostnames, prefixes, object names, and example policy in this post and the repo are fictional lab material, not an internal standard.*