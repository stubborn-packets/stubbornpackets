---
title: "Cisco ISE Policy-as-Code Migration - Part 2"
date: 2026-09-18
draft: false
description: "Phase 2 is a local naming linter. Bad names, missing state, and expired exceptions fail in Git. The API still has not seen a write."
categories: ["Terraform", "Cisco", "ISE"]
tags: ["Terraform", "Cisco", "ISE", "Policy-as-Code"]
cover:
  image: "ise-pac-part2-hero.jpg"
  alt: "Homelab laptop with messy ISE GUI names and a lint failure on screen, notebook mind map of lint_ise.py on the desk"
---

# Cisco ISE Policy-as-Code Migration - Part 2

Part 1 left a repo that could look at the lab ISE and a pile of CSVs that made the non-standard GUI names obvious. That is useful. It does not do much for me when I am about to type a name into `policy/`.

A naming table in `docs/naming.md` is still a dream until something refuses to merge `IT Users VLAN` into desired state. Phase 2 is that something: a local Python 3.12 linter that reads the YAML I would actually commit and fails the name before the first API call.

Nothing is applied. There is still no Terraform provider block. The lab node has not been written to.

This code is a little beyond where I wanted to start, so I used Grok AI to write it while I kept the direction and the questions. Instead of generating a whole phase and moving on, I made it walk the script in small pieces: what each check does, why it lives in config instead of a buried `if`, and what a fail is supposed to look like. A 600-line dump is noise. Chunks with an explanation I can argue with is how I actually learned the thing.

The post is a snapshot. The repo will move as later phases land.

**Note:** *This is a personal homelab project. Prefixes, object names, and example policy shapes are fictional. They came from public posts on this topic and from what I think a brownfield lab should look like so the export is interesting. They are not copied from an internal standard and they are not names used by my employer. Nothing here is employer policy, employer architecture, or employer data.*

*Opinions are mine.*

---

## Github Repos

**cisco-ise** → [stubborn-packets/cisco-ise](https://github.com/stubborn-packets/cisco-ise)

*The post is a snapshot and the repos are live and will be updated as I progress through the phases. Things you see in the repos may have items that have been updated since this post.*

Main changes in this increment:

- `scripts/lint_ise.py`
- `lint/prefixes.yaml`
- `lint/builtins.yaml`
- `lint/README.md`
- `tests/lint/test_lint_ise.py`

---

## Where Part 1 stopped

[Part 1](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1) was Phase 0 plus Phase 1.

- Three planes that stay split: `policy/` (desired), `inventory/` (reference), `exports/` (observed, gitignored)
- One lab ISE under `environments/lab/`. No `prod/` folder. No `stage/` folder unless a second eval node shows up.
- `{PREFIX}-` plus kebab-case for user-created names. ISE built-ins keep ISE names.
- `scripts/export_ise.py` GET-only against ISE 3.3 patch 1 so the fail-column names were visible

The export answered *what did I actually click together in the GUI?* It does not answer *should this YAML become desired state?* If I copy a brownfield name from `exports/policy-sets.csv` into `policy/policy-sets.yaml`, the next phase treats that string as something Terraform should own. That is how a leftover GUI name stops being evidence and starts being grammar.

The linter exists so that copy-paste dies in VS Code, not in ISE.

## What the linter is allowed to see

I was tempted to point the script at `exports/` and "clean up the lab." That is the wrong plane. Observed state is allowed to be ugly. Desired state is not.

`scripts/lint_ise.py` walks two folders and two config files. That is the whole input set.

| Reads | Does not read |
|-------|----------------|
| `lint/prefixes.yaml` | `exports/` |
| `lint/builtins.yaml` | ISE (no HTTP client) |
| `policy/*.yaml` | `.env` |
| `inventory/*.yaml` | Terraform |

Empty scaffolds pass on purpose:

```text
lint_ise: ok (0 object(s))
```

An empty `policy_sets: []` is not a warning. I wanted the repo to exist before it contained objects. The first write is still a later phase.

```bash
python3.12 scripts/lint_ise.py
python3.12 -m unittest tests.lint.test_lint_ise
```

Twenty-four tests. No node in the path. If those fail, I broke the grammar. If they pass and I still have a bad name in `policy/`, the linter is the bug, not my memory of the standard.

## The grammar is a regex, not a vibe

Part 1 spent more time on naming than on folders. I wanted something a reviewer can fail, not another house style I will ignore at 11pm.

Human table: `docs/naming.md`.
Machine table: `lint/prefixes.yaml`.
Enforcement: `NAME_RE` in `scripts/lint_ise.py`.

```text
NAME_RE  = PREFIX- + kebab-case [a-z0-9] tokens
KEBAB_RE = [a-z0-9]+(?:-[a-z0-9]+)*
```

Prefix stays uppercase. Tokens after the first hyphen are lowercase. No spaces, underscores, or Title Case pretending to be kebab. Consistency is the whole point. If I have to squint to decide whether a name is "close enough," the regex already lost.

| Pass | Fail |
|------|------|
| `PS-global-wired-8021x` | `IT Users VLAN` |
| `AN-dot1x-ad` | `DOT1X_CERTS` |
| `PR-wired-corp-access` | `guest-VLAN` |
| `Default` | `LAN-POLICY-1` |

`Default`, `Default Network Access`, `Unknown`, and the rest of the system names live in `lint/builtins.yaml`. They skip the prefix rule. I am not renaming ISE's own objects to make a spreadsheet pretty.

The file map matters too. `PS-wired-dot1x` looks legal until it sits in `allowed-protocols.yaml`. That file wants `AP-`. A good string in the wrong collection is still a fail. `FOO-wired-dot1x` is just an unknown prefix.

Policy-set ranks are part of the name on purpose. ISE evaluates by rank. If rank 40 can be any string I liked that week, the evaluation order and the name drift apart and I am back to screenshots.

| Rank | Required name |
|------|----------------|
| 1–9 | `PS-infra-health-checks` |
| 10 | `PS-global-vpn` |
| 20 | `PS-global-wireless-8021x` |
| 30 | `PS-global-wireless-mab` |
| 40 | `PS-global-wired-8021x` |
| 50 | `PS-global-wired-mab` |
| 60–69 | `PS-<bu>-vpn` |
| 70–79 | `PS-<bu>-wireless` |
| 80–89 | `PS-<bu>-wired` |
| 99 | `Default` |

`<bu>` must be kebab-case. `PS-Lab-wired` at rank 80 fails. `PS-lab-wired` matches the pattern. The token is **not** joined to `policy/ndg.yaml` BusinessUnit `allowed_values` yet. Those lists start empty on purpose. Turn the join on while they are `[]` and every BU-scoped set fails. That wiring is a follow-up, written down in `lint/README.md` so I do not "just add it" on a tired night.

## Names were the headline. They are not the only check.

A legal name on an incomplete object is still how I would ship a landmine.

**`state` is required on policy objects.** `enabled` or `disabled`. Not `active`. Not omitted because "of course it is on." Cutover later is create-beside, move traffic, disable, delete. If `state` is not a field now, disable-then-delete is a GUI click again.

**Exceptions are not immortal.** I have watched temporary access become the real policy because nobody put a date on it. `EX-*` needs `ticket`, `owner`, and `expires_on` as `YYYY-MM-DD`. A date in the past fails. The sample in the repo is lab fiction:

```yaml
- name: EX-vendor-wired-temp
  state: enabled
  policy_set: PS-global-wired-mab
  ticket: CHG0001234
  owner: lab
  expires_on: "2099-12-31"
```

**NDG roots are locked.** `All Locations`, `All Device Types`, `BusinessUnit`, `Stage`, `Function`. A sixth root that only exists because I felt creative in `policy/ndg.yaml` fails as "unknown NDG root." Location leaves are ISO 3166-1 alpha-3 (`USA` passes, `usa` fails). Other leaves are kebab-case. NAD `ndg:` keys in `inventory/nads.yaml` have to map to those roots.

NAD `ndg.business_unit` *is* checked against the BusinessUnit allow-list once that list is non-empty. Fill the list before Phase 4 writes NDGs. An empty list is not a standard. It is a reminder that I have not decided the values yet.

**UUIDs do not belong in `policy/` or `inventory/`.** Desired state is a name. The id belongs in an import map later, when I am ready to own the object. Paste an ERS `id` into YAML now and the review surface is no longer something a human can read.

**Network Access only.** Keys or scopes that look like `tacacs` or `device-admin` fail. This PoC is not Device Admin. The rejected-scope list lives in `lint/prefixes.yaml` so that is a config change, not a comment I will miss.

Inventory is looser on purpose. A NAD can be `lab-sw-01`. A VIP has to be `VIP-*`. A VLAN name has to be `VLAN-*`. Those prefixes are documentation-only. NADs are not Terraform-managed in this repo. Switch hostnames are someone else's problem, and unless every team that racks a switch already follows one pattern, a lot of them will not. That is why they stay in `inventory/` instead of becoming desired policy.

## A bad name should look like a bad name

Drop a GUI leftover into an otherwise empty fixture and the output is boring, which is the point:

```text
lint_ise: 1 error(s), 1 object(s)
  policy/authz-profiles.yaml: IT Users VLAN: name must be PREFIX- + kebab-case [a-z0-9] (e.g. PS-global-wired-8021x)
```

Wrong prefix for the file:

```text
  policy/allowed-protocols.yaml: PS-wired-dot1x: expected prefix AP- for this object, found PS-
```

Missing state:

```text
  policy/sgt.yaml: SGT-lab-users: missing state (enabled|disabled)
```

That is the "this is fun" moment from the end of Part 1. The editor goes red. The lab GUI does not gain a surprise object.

## Two documents, one job

If the linter and the docs disagree, CI believes the script and the PR argument believes the markdown. I have had that fight with myself already. `lint/README.md` is the change procedure so I cannot edit one layer and call it done.

```text
docs/naming.md          human standard
docs/decisions.md       locked rank bands and NDG roots
lint/prefixes.yaml      PREFIX table
lint/builtins.yaml      built-ins + NDG lock
scripts/lint_ise.py     regex + rank map + file map
policy/ + inventory/    desired / reference objects
```

Want rank 40 to be `PS-campus-wired-dot1x` instead of `PS-global-wired-8021x`? Edit the human table, change `POLICY_SET_RANK_EXACT` in the script, fix the test that uses the old name, run the 24 tests, *then* put the new name in `policy/policy-sets.yaml`. Skip a layer and I will merge a name I cannot explain in six months.

Same idea for a new prefix or a new ISE built-in. Built-ins go in `lint/builtins.yaml` as ISE spelled them. I am not inventing a `PREFIX-` for `PermitAccess`. I am also not stuffing leftover lab GUI names onto that list just to silence the linter. Those names stay in `exports/`.

Pointing this model at another ISE later is a config job: different prefixes, extra system names, that site's allow-lists, maybe a different rank table. What does not change in this checkout: `environments/lab/` is the only write target, there is no `prod/` folder, and NADs stay in `inventory/`.

## What Phase 2 doesn't do (a good thing)

- Connect to ISE
- Lint `exports/`
- Rewrite `scripts/export_ise.py` onto `ciscoisesdk`
- Add a Terraform provider
- Import the non-standard lab names as desired state
- Rename live objects in place

Mixing `exports/` into `policy/` is how Phase 3 inherits a mess. I would rather stare at an empty scaffold than teach Terraform a name I already decided was wrong.

The current repo still passes with zero objects. That is the correct Phase 2 exit: a bad name fails, a good name passes, ISE is untouched.

## Next

Part 3 is the first object Terraform is allowed to create. Pin `CiscoDevNet/ise` under `environments/lab/`, local state only, one low-risk lab object. Still no `prod/`. Still no policy-set cutover.

The linter stays in front of that apply. If the name is wrong, the API should never see the call.

## Series
- [Part 1 — repo + read-only export](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1)
- Part 2 — naming linter  ← this post
- [Part 3 — first lab apply]((https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part3))
- Part 4 - policy building blocks (coming soon)

---

*Personal homelab only. Not affiliated with my employer. Hostnames, prefixes, object names, and example policy in this post and the repo are fictional lab material, not an internal standard.*
