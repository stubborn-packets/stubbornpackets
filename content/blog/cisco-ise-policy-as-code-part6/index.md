---
title: "Cisco ISE Policy-as-Code Migration - Part 6"
date: 2026-10-16
draft: false
description: "YAML rank is the packed ISE index. scripts/read_policy.py emits the lab calls for every family. Bootstrap SGT moved into policy/. Sets stayed disabled."
categories: ["Terraform", "Cisco", "ISE"]
tags: ["Terraform", "Cisco", "ISE", "Policy-as-Code"]
cover:
  image: "ise-pac-yaml-hero.jpg"
  alt: "Homelab desk with a YAML file on the left, a generated Terraform tree on the right, and a pencil line connecting them"
---

# Cisco ISE Policy-as-Code Migration - Part 6

Part 5 put eight policy sets on the lab, all disabled. Infra, wireless 802.1X, wireless MAB, VPN, wired 802.1X, wired MAB, then the first business-unit pair. Network Access only. `CiscoDevNet/ise` 0.4.1. Local state. YAML rank was still the old design band. ISE rank is a packed insert index. Those were not the same number.

Part 6 is the conversion that closed the split. No new objects. No enable. The tree changed because the lab calls stopped being hand-copied. `scripts/read_policy.py` now emits them from `policy/`. Rank in the YAML is the index ERS stores. `band` is a label. It is not sent.

This series is still the same question as Part 1: *is Policy-as-Code on ISE 3.x a real operating model, or a Cisco Live slide*. Phase 5 answered "*can a disabled set exist beside the old GUI sets.*" This increment answers "*can the reviewed YAML be the thing the lab root reads, without a second hidden rank and without a hand-copied call that drifts.*" A plan is a plan. The copied lab file was the drift Part 5 left open. 

I used Grok AI the same way as Part 5. Direction and questions stayed mine. Each family was walked in pieces so I could argue with it — *why is this rank still a band, can the address stay derived from the name, what does not belong in `policy/`*. A generated tree of every ISE object in one paste is how I stop learning.

The post is a snapshot. The repo will move as later phases land.

**Note:** *This is a personal homelab project. Prefixes, object names, and example policy shapes are fictional. They came from public posts on this topic and from what I think a brownfield lab should look like so the export is interesting. They are not copied from an internal standard and they are not names used by my employer. Nothing here is employer policy, employer architecture, or employer data.*

*Opinions are mine.*

---

## Github Repos

**cisco-ise** → [stubborn-packets/cisco-ise](https://github.com/stubborn-packets/cisco-ise)

*The post is a snapshot and the repos are live and will be updated as I progress through the phases. Things you see in the repos may have items that have been updated since this post.*

Main changes in this increment, in the order they happened:

- Rank conversion 2026-10-13. `policy/policy-sets.yaml` `rank` is the packed index. `band` is a label, not sent.
- `scripts/read_policy.py` closed one family at a time. It now emits the lab calls for every managed family plus `outputs.tf`.
- Bootstrap SGT move 2026-10-16. `SGT_lab_bootstrap` / 1001 lives in `policy/sgt.yaml`. Address is `module.sgt_lab_bootstrap`.

The cloneable artifact is the reader plus the YAML it reads. Not a hand-copied `.tf` tree. There is still no `for_each` over YAML. A reviewer reads the YAML. The reader emits one module call per object. Ids stay out of `policy/`.

`it`, `PS-<bu>-vpn`, and `PS-<bu>-wireless` are still not on that list. A lab with nothing pointed at it does not gain a pattern from a copy. Sets stayed disabled. Enable is a cutover, not this post.

---

## Where Part 5 stopped

[Part 1](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1) was the repo and the read-only export. [Part 2](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part2) was the linter. [Part 3](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part3) was the first write. [Part 4](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part4) was the building blocks. [Part 5](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part5) was the sets, disabled.

- Three planes that stay split: `policy/` (desired), `inventory/` (reference), `exports/` (observed, gitignored)
- One lab ISE under `environments/lab/`. No `prod/` folder. No `stage/` folder unless a second eval node shows up.
- `{PREFIX}-` plus kebab-case, except SGTs. SGT is `SGT_` plus lowercase snake_case, max 32. YAML name equals the ISE name. No mapper.
- `scripts/lint_ise.py` reads `policy/` and `inventory/` only. No HTTP client. No Terraform.
- Provider pin is `CiscoDevNet/ise` 0.4.1. Credentials stay in repo-root `.env`. Both the exporter and the provider read `ISE_URL`.
- State is local: `environments/lab/terraform.tfstate`, gitignored. The id from `terraform output` stays in that file.

Phase 5 did not teach Terraform to read the YAML. The lab calls were still copied. YAML rank stayed the design band. That mismatch is what this post closes.

## Rank is the packed index

The paper model was a spacing convention. Infra in 1–9. VPN at 10. Wireless at 20 and 30. Wired at 40 and 50. Business-unit at 80. Default at 99. Lower number, evaluated first, with gaps so a later set could land between them.

ISE does not store those numbers. Policy-set rank is a packed insert index. The first apply in Part 5 posted rank 40. The node rejected it. Legal create rank on this lab started at 0. Default cannot be moved to 99. I am not going to manage the built-in so the YAML can keep a row called 99.

`rank` in `policy/policy-sets.yaml` is the index ISE stores. `band` is a label. Terraform does not send it. The linter checks that the ranks are packed 0..n-1 and that each band is a known label and contiguous. It does not enforce the family sequence. Live order is the insert order, not the sequence I first wrote in `docs/decisions.md`.

| rank | name | band |
| --- | --- | --- |
| 0 | `PS-hr-wired` | `bu-wired` |
| 1 | `PS-guest-wired` | `bu-wired` |
| 2 | `PS-infra-health-checks` | `infra` |
| 3 | `PS-global-wireless-8021x` | `global-wireless` |
| 4 | `PS-global-wireless-mab` | `global-wireless-mab` |
| 5 | `PS-global-vpn` | `global-vpn` |
| 6 | `PS-global-wired-8021x` | `global-wired` |
| 7 | `PS-global-wired-mab` | `global-wired-mab` |

HR and guest are 0 and 1 because they were inserted at the front. Moving a set means editing `rank` on the sets that shift. That edit is the order change. There is no second hidden number. `POLICY_SET_RANK_EXACT` is retired. `ise_network_access_policy_set_update_rank` stays unused. It cannot open a gap, and the YAML no longer asks for one.

Unmanaged lab sets still occupy slots. Terraform rank output does not show them. GUI order is the evaluation order. They were not imported. Rank 99 Default is not in this table.

## The reader emits the lab calls

I did not teach Terraform to read the YAML. `scripts/read_policy.py` does. One family at a time. Policy sets first, then authentication rules, then authorization rules, then conditions, allowed protocols, DACLs and profiles, SGTs, EIGs, NDGs, and finally `outputs.tf`.

The generated files are:

- `environments/lab/policy-sets.tf`
- `environments/lab/authn-rules.tf`
- `environments/lab/authz-rules.tf`
- `environments/lab/conditions.tf`
- `environments/lab/allowed-protocols.tf`
- `environments/lab/dacls.tf`
- `environments/lab/authz-profiles.tf`
- `environments/lab/sgts.tf`
- `environments/lab/eigs.tf`
- `environments/lab/ndg.tf`
- `environments/lab/outputs.tf`

Do not hand-edit them. Regenerate with `python3 scripts/read_policy.py`.

Addresses stay `module.ps_*`, `module.an_*`, `module.az_*`, and the matching family prefixes. Map keys are derived from the YAML name only. No `for_each`. A condition reference in the YAML is a name. The reader turns `condition: CND-vpn` into `module.cnd_vpn.id` inside the generated file. It does not look up a UUID. Ids stay in state. They do not go into `policy/` or `inventory/`.

NDG objects are derived from the allow-lists in `policy/ndg.yaml`. Built-in Location and Device Type roots are not managed. The custom roots (`BusinessUnit#BusinessUnit`, `Stage#Stage`, `Function#Function`) and the leaves are. VLAN names on profiles are resolved against `inventory/vlans.yaml`. Default Network Access is not managed. No SGT on the profiles.

That is still slower than a loop over the whole file. It is also the point. A reviewer can read the YAML, then read the generated call, and see that the name, the rank, and the condition match. A single `for_each` would hide the first-write subset inside the loop. I wanted the PR to show the emitted calls, not "and whatever else the reader produced."

## Outputs and the bootstrap move

Outputs were the last hand-written family. The same reader emits `outputs.tf`. Map keys are still derived from the YAML name. Bootstrap SGT outputs stayed hand-written until the next increment, so a bad regenerate could not orphan the Phase 3 object.

The fence came out only after the state move. `SGT_lab_bootstrap` / 1001 moved into `policy/sgt.yaml`. The reader emits `module.sgt_lab_bootstrap`. The map key is `lab_bootstrap`. Derived label is `sgt_lab_bootstrap`, not the old abbreviated `sgt_bootstrap`. The state move was required:

```text
terraform state mv module.sgt_bootstrap module.sgt_lab_bootstrap
```

The hand-written module call in `main.tf`, the variables, and the tfvars values are gone. No scalar outputs. The three scalars (`sgt_id`, `sgt_name`, `sgt_value`) were kept for one regenerate and then dropped. They duplicated the map keys.

```yaml
sgts:
  - name: SGT_lab_bootstrap
    value: 1001
    description: Phase 3 bootstrap object. Safe to destroy.
    owner: lab
    bu: lab
    propagate_to_apic: false
    state: enabled
```

The object is still the same Phase 3 bootstrap. It is just no longer a special case outside the reader.

## What this increment doesn't do (a good thing)

- Enable any managed set
- Create the F5 user, place a NAD, or put an endpoint in an EIG
- `it`, `PS-<bu>-vpn`, or `PS-<bu>-wireless`
- Import the old GUI sets, or move Default to 99
- An SGT on any profile
- Manage Default Network Access or PermitAccess
- `environments/stage/` or `prod/`
- A NAD module, or an import map

Those are later phases, or they are not this lab. Mixing an enable into this post would hide the only question this increment was allowed to answer: can the reviewed YAML be the source the lab root reads, with the rank bruise closed and the bootstrap no longer a special case.

## Next

Sets stay disabled. Enable is the cutover. Phase 6 is exceptions. `EX-*` requires ticket, owner, and expiry. The linter already fails expired exceptions. Still lab only. Still no `prod/`. Still no NAD module. The sets this post walked are names. They are not traffic.

## Series

- [Part 1 — repo + read-only export](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1)
- [Part 2 — naming linter](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part2)
- [Part 3 — first lab apply](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part3)
- [Part 4 — policy building blocks](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part4)
- [Part 5 — policy sets, disabled](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part5)
- Part 6 — YAML conversion and the reader ← this post

---

*Personal homelab only. Not affiliated with my employer. Hostnames, prefixes, object names, and example policy in this post and the repo are fictional lab material, not an internal standard.*
