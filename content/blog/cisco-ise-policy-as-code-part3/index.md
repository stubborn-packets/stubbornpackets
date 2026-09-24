---
title: "Cisco ISE Policy-as-Code Migration - Part 3"
date: 2026-09-22
draft: false
description: "Phase 3 is the first Terraform write: one lab SGT, local state, create then destroy. The plan survived. The naming standard did not."
categories: ["Terraform", "Cisco", "ISE"]
tags: ["Terraform", "Cisco", "ISE", "Policy-as-Code"]
cover:
  image: "ise-pac-part3-hero.jpg"
  alt: "Homelab laptop with a Terraform plan for one lab SGT next to the ISE TrustSec security groups page"
---

# Cisco ISE Policy-as-Code Migration - Part 3

Part 2 left a linter that could fail a bad name in VS Code and an ISE that still had not seen a write. That was the point. I did not want the first "this is fun" moment to be a surprise object in ISE.

Part 3 is where the Terraform fun starts — the first real write of the project — and seeing if it can create and destroy one object: an SGT. Lab ISE only. Local state. Create, then destroy. Nothing about policy sets, nothing about NADs, and `policy/sgt.yaml` is still an empty list.

This series is still the same question as Part 1: *is Policy-as-Code on ISE 3.x a real operating model, or a Cisco Live slide*. The only way I get an honest answer is to run the plan against the lab ISE and see. A plan is a plan. The repo, the prefix table, even the name I typed into `terraform.tfvars` are ideas, hopes, and dreams, until ERS accepts them. Turning this to facts means the standard will change when the API disagrees. *That happened in this phase*. I am leaving the bruised ego in, because hiding it would make the next person, or later me, think the first table in `docs/naming.md` was revealed truth.

I used Grok AI the same way as Part 2. Direction and questions stayed mine. The provider pin, the thin Terraform module, and the name fight were walked in pieces so I could argue with each one - *why is it done this way, can it be done another way, what is the point of this code object*. A whole `.tf` tree in one paste is how I stop learning.

The post is a snapshot. The repo will move as later phases land.

**Note:** *This is a personal homelab project. Prefixes, object names, and example policy shapes are fictional. They came from public posts on this topic and from what I think a brownfield lab should look like so the export is interesting. They are not copied from an internal standard and they are not names used by my employer. Nothing here is employer policy, employer architecture, or employer data.*

*Opinions are mine.*

---

## Github Repos

**cisco-ise** → [stubborn-packets/cisco-ise](https://github.com/stubborn-packets/cisco-ise)

*The post is a snapshot and the repos are live and will be updated as I progress through the phases. Things you see in the repos may have items that have been updated since this post.*

Main changes in this increment:

- `terraform/modules/sgt/`
- `environments/lab/*.tf` and `terraform.tfvars`
- SGT grammar exception in `docs/naming.md`, `scripts/lint_ise.py`, and `tests/lint/`

`policy/sgt.yaml` is not on that list. It is still empty. The bootstrap name lives in lab tfvars, not in desired-policy YAML.

---

## Where Part 2 stopped

[Part 1](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1) was Phase 0 plus Phase 1. [Part 2](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part2) was the linter.

- Three planes that stay split: `policy/` (desired), `inventory/` (reference), `exports/` (observed, gitignored)
- One lab ISE under `environments/lab/`. No `prod/` folder. No `stage/` folder unless a second eval node shows up.
- `{PREFIX}-` plus kebab-case for user-created names. ISE built-ins keep ISE names.
- `scripts/lint_ise.py` reads `policy/` and `inventory/` only. No HTTP client. No Terraform. Empty lists pass.

The linter answered *should this YAML become desired state?* It still does not write. That was the correct Phase 2 exit: a bad name fails in the editor, ISE does not gain an object.

I am repeating the split because Phase 3 is the first time something in this repo can POST. The second those planes collapse, a leftover GUI name in `exports/` starts looking like something Terraform should own. I would rather keep an empty `policy/sgt.yaml` than teach the provider a string I already decided was wrong.

## Why an SGT, and why it is not in `policy/`

I wanted the first write to be something I could easily create and then take back. An unused SGT is deletable through ERS — especially when nothing is using it. A policy set or a NAD would be a little trickier. NADs stay in `inventory/` in this repo. They are not Terraform-managed.

So the bootstrap is one security group: name plus tag, no traffic, nothing hanging off it. If apply works and destroy works, the writer is real. If either side fails, I find out on an object I can throw away.

That object does not live in `policy/sgt.yaml`. That file is still an empty list. Phase 4 is when desired-policy YAML starts meaning "*Terraform should own this family.*" Phase 3 is only "*can this root talk to the lab ISE at all.*" I put the name and tag in `environments/lab/terraform.tfvars` so I could not pretend the review surface was ready.

Part 1 and Part 2 already had a prefix for this family: `SGT-` plus kebab-case. Same rule as `PS-` and `PR-`. One table, one regex. That is what I typed.

```hcl
sgt_name              = "SGT-lab-bootstrap"
sgt_value             = 1001
sgt_description       = "Phase 3 bootstrap object. Safe to destroy."
sgt_propagate_to_apic = false
```

Tag `1001` is a lab pick. Do not treat it as a site standard. The description is a note to future me: *this row is allowed to disappear*. The name is the house style I had committed to on paper. ERS had not seen it yet.

Create-beside, move traffic, disable, delete is still the cutover for later families. This SGT is not cutover. If it is still on the node when the phase is done, I did the phase wrong.

## Writer and wiring

The writer is `CiscoDevNet/ise` 0.4.1, pinned under `environments/lab/`. It is not `netascode/nac-ise`. I looked at both. nac-ise wants to own a lot of ISE from one stack. *That is something I'll possibly come back to if I run into limitations*. This phase needed one resource type, one lab node, and a provider I could pin and read.

The module does not know which ISE it talks to. `terraform/modules/sgt/` is one `ise_trustsec_security_group` resource. The root that may call it is `environments/lab/` only. There is no `environments/stage/`. There is no `prod/`. I am not going
to invent a second node on paper so the folder tree looks grown-up —
*even though I really want to.* `stage/` shows up when a second eval
node exists. Not before.

Credentials stay in the repo-root `.env`. Terraform does not load that file by itself. If I run `terraform plan` from a clean shell, the provider has no URL and no user. That is annoying and correct. I source the file, then I run the commands:

```bash
cd environments/lab
set -a && source ../../.env && set +a
terraform init
terraform plan
```

`scripts/export_ise.py` used to look for `ISE_BASE_URL`. The TF provider looks for `ISE_URL`. Two names for the same node is how you spend a night debugging a 404 that is really a typo. Phase 3 collapsed that. Both the exporter and the provider read `ISE_URL`. `.env.example` has one URL key. Same value, two tools, one file. *This was a one-line cleanup in `scripts/export_ise.py`.*

State is local: `environments/lab/terraform.tfstate`, gitignored. I am not standing up remote state for one object on a homelab eval. The id that comes back from `terraform output` stays in that state file. It does not go into `policy/` or `inventory/`. Desired state is a name. The UUID is bookkeeping.

`terraform validate` did not catch the errors I thought it would — I was expecting it to check variable values, *but it does not*.

```bash
terraform validate
Success! The configuration is valid.
```

It checks syntax and types. It does not load `terraform.tfvars`. A bad SGT name will look fine until `plan`, which is the first command that actually evaluates the value I typed.

## Create, then destroy

Now to the bruised ego, and how I got humbly reminded that a plan is just a plan until you test it.

`SGT-lab-bootstrap` was a legal name in *my* table. Prefix uppercase, kebab tokens, same shape as `PS-global-wired-8021x`. `terraform validate` was green. I sourced `.env` and ran `plan`. The plan wanted to create one `ise_trustsec_security_group`. I applied it.

ERS returned **HTTP 400**:

```text
Invalid Security Group name, name may not be null andlonger than 32
characters and only contain the alphanumeric or underscore characters.
```

That is ISE's wording, missing space and all. SGT names are `[A-Za-z0-9_]`, max 32. A hyphen is not in that set. The house style I had been so pleased with in Part 1 is a character the API will not store. That is not a Terraform bug and it is not a provider bug. **It is ISE telling me the spreadsheet was a guess.**

I wanted one standard. One regex. One review comment: *make it look like the others.* A mapper that turned `SGT-lab-bootstrap` into `SGT_lab_bootstrap` on the way to ERS would have kept the YAML pretty and made the node the liar. I would spend the rest of the project wondering which name was real. YAML name equals ISE name. No mapper.

So the plan changed. The object I can actually create is:

```hcl
sgt_name              = "SGT_lab_bootstrap"
sgt_value             = 1001
sgt_description       = "Phase 3 bootstrap object. Safe to destroy."
sgt_propagate_to_apic = false
```

Same tag. Same "safe to destroy" note. Different character in the name because the node owns the charset.

That apply worked. I created `SGT_lab_bootstrap` / tag `1001` on the lab ISE through `CiscoDevNet/ise` 0.4.1, then destroyed it. The unused SGT came off the node through ERS. Local state only. That is the Phase 3 exit.

```bash
terraform apply
terraform output
terraform destroy
```

Check the GUI if you want the comfort: **Work Centers > TrustSec > Components > Security Groups**. After destroy it should not be there. If it is, something is broken.

`terraform output` will print an id. Leave it in state. Do not paste it into `policy/` or into this post as if it were desired state.

The linter and the module check still thought `SGT-lab-users` was the good name. That is the next fix. ISE already voted. The docs had not caught up.

## The linter follows ISE

I sat with this longer than the module. The whole point of Phase 2 was one table a reviewer could fail the merge without much thought if they saw the name was wrong. `SGT-` plus kebab lived in that table next to `PS-` and `PR-`. Making SGT a one-off feels like the start of "*except for this family*" forever.

The other option is worse. Widen the regex so hyphens and underscores both pass, add a mapper so YAML can stay kebab, or invent `SGT-Lab-Users` because Title Case "*looks official*." Then Git, Terraform, and ISE are three different strings. The 400 already told me which string counts.

So the house style does not move. Everything else stays `{PREFIX}-` plus kebab-case. SGT is the exception, and it is documented as an exception.

`SGT_` + lowercase snake_case, max 32. YAML name equals the ISE name.

| Pass | Fail |
|------|------|
| `SGT_lab_users` | `SGT-lab-users` |
| `SGT_lab_bootstrap` | `SGT_Lab_Users` |
| `SGT_hr_users` |  `SGT_HR-Users` |

Same change procedure as Part 2. Skip a layer and CI believes the script while the PR argument believes last month's markdown.

```text
docs/naming.md          human standard (SGT exception lives here)
lint/prefixes.yaml      PREFIX tokens — not the separator
scripts/lint_ise.py     SGT_RE, still max 32
tests/lint/             fail SGT-lab-users, pass SGT_lab_users
terraform/modules/sgt/  same regex on var.name
```

`prefixes.yaml` still says the token is `SGT`. It does not own `-` vs `_`. That fight belongs in the regex and the human doc.

The module check is the same rule as the linter, on purpose. After I changed tfvars to `SGT_lab_bootstrap`, an old kebab validation on `var.name` would fail `plan` before ERS ever saw the corrected string. ISE's charset is now in both places. I did not widen the module to "*whatever might work.*" Hyphens still fail. Title Case still fails.

`policy/sgt.yaml` is still an empty list. The pass/fail names above live in the tests, not in that file. The first real SGT in desired-policy YAML is Phase 4. The linter is ready for that name. The file is not.

This is the loop I wanted in the opener: plan, work, test, the node disagrees, re-plan, work, test. The lab exists so the standard can lose once, in private. If every family that 400s on hyphen gets a mapper instead of an exception, I am back to a Confluence page.

## What Phase 3 doesn't do (a good thing)

- Fill `policy/sgt.yaml`
- Manage NADs with Terraform
- Import a leftover GUI SGT as `module.sgt_bootstrap`
- Apply anywhere but lab
- Stand up `environments/stage/` or `prod/`
- Start NDG, allowed protocols, conditions, or authz profiles

Those families are Phase 4. Mixing them into this post would hide the only question this phase was allowed to answer: can this root create one object and take it back.

The linter still does not talk to ISE. `exports/` is still not desired state. The first write was a bootstrap in tfvars, not a cutover.

## Next

Part 4 is Phase 4: policy building blocks, in order — NDG, then allowed protocols, then conditions, then authz profiles / DACLs, then SGTs in `policy/`, then EIGs.

Still lab only. Still no `prod/`. Still no NAD module. Fill the NDG allow-lists before any NDG write. `policy/sgt.yaml` stays empty until that phase writes it on purpose.

The linter stays in front of those applies. If the name is wrong, ERS should not see the call. I already paid for that lesson once.

## Series
- [Part 1 — repo + read-only export](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1)
- [Part 2 — naming linter](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part2)
- Part 3 — first lab apply ← this post
- Part 4 - policy building blocks (coming soon)

---

*Personal homelab only. Not affiliated with my employer. Hostnames, prefixes, object names, and example policy in this post and the repo are fictional lab material, not an internal standard.*