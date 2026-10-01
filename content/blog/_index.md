---
title: "Cisco ISE Policy-as-Code Migration - Part 4"
date: 2026-10-01
draft: false
description: "Phase 4 is the building blocks a policy set would call: NDG, allowed protocols, conditions, a DACL and profile, two SGTs, two EIGs. YAML is the review surface. The modules stayed thin. ERS still got a vote."
categories: ["Terraform", "Cisco", "ISE"]
tags: ["Terraform", "Cisco", "ISE", "Policy-as-Code"]
cover:
  image: "ise-pac-part4-hero.jpg"
  alt: "Homelab desk with a YAML review of ISE building blocks beside a network device group hierarchy, sticky note reading building blocks not policy sets"
---

# Cisco ISE Policy-as-Code Migration - Part 4

Part 3 left one object I could create and take back: `SGT_lab_bootstrap` / tag `1001`, local state, lab ISE only. The hyphenated name I had been pleased with in Part 1 did not survive ERS. That was the point of the phase. The writer was real. Desired-policy YAML was still empty.

Part 4 is the building blocks a later policy set would call. Not the policy set. Network device groups, two allowed-protocols services, two library conditions, one downloadable ACL and the profile that hangs off it, two SGTs in `policy/sgt.yaml`, two static endpoint identity groups. One family at a time. Lab ISE only. `CiscoDevNet/ise` 0.4.1. Local state. Exit met 2026-10-01.

This series is still the same question as Part 1: *is Policy-as-Code on ISE 3.x a real operating model, or a Cisco Live slide*. Phase 3 answered "can this root POST." Phase 4 answers "can the objects a policy set needs exist as reviewed YAML, and will ERS store the shape I wrote down." A plan is a plan. The first table was wrong again, in six different ways. I am leaving the bruises in. Hiding them would make the next person, or later me, think `docs/naming.md` already knew about `#`.

I used Grok AI the same way as Part 3. Direction and questions stayed mine. Each family was walked in pieces so I could argue with it — *why is the name shaped like that, can the module stay one resource, what does not belong in `policy/`*. A generated tree of every ISE object in one paste is how I stop learning.

The post is a snapshot. The repo will move as later phases land.

**Note:** *This is a personal homelab project. Prefixes, object names, and example policy shapes are fictional. They came from public posts on this topic and from what I think a brownfield lab should look like so the export is interesting. They are not copied from an internal standard and they are not names used by my employer. Nothing here is employer policy, employer architecture, or employer data.*

*Opinions are mine.*

---

## Github Repos

**cisco-ise** → [stubborn-packets/cisco-ise](https://github.com/stubborn-packets/cisco-ise)

*The post is a snapshot and the repos are live and will be updated as I progress through the phases. Things you see in the repos may have items that have been updated since this post.*

Main changes in this increment:

- `policy/ndg.yaml`, `allowed-protocols.yaml`, `conditions.yaml`, `authz-profiles.yaml`, `sgt.yaml`, `eig.yaml`
- Thin modules under `terraform/modules/{ndg,allowed-protocols,condition,dacl,authorization-profile,eig}/`
- Lab calls in `environments/lab/{ndg,allowed-protocols,conditions,dacls,authz-profiles,sgts,eigs}.tf`
- `SGT_lab_bootstrap` still lives in lab tfvars. It is not a row in `policy/sgt.yaml`.

The cloneable artifact is those modules plus the lab calls. Not a generated tree. There is no `for_each` over YAML. A reviewer reads the YAML. Terraform writes one object per module call.

`policy/policy-sets.yaml` is not on that list. It is still the Phase 5 scaffold.

---

## Where Part 3 stopped

[Part 1](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1) was the repo and the read-only export. [Part 2](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part2) was the linter. [Part 3](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part3) was the first write.

- Three planes that stay split: `policy/` (desired), `inventory/` (reference), `exports/` (observed, gitignored)
- One lab ISE under `environments/lab/`. No `prod/` folder. No `stage/` folder unless a second eval node shows up.
- `{PREFIX}-` plus kebab-case, except SGTs. SGT is `SGT_` plus lowercase snake_case, max 32. YAML name equals the ISE name. No mapper.
- `scripts/lint_ise.py` reads `policy/` and `inventory/` only. No HTTP client. No Terraform.
- Provider pin is `CiscoDevNet/ise` 0.4.1. Credentials stay in repo-root `.env`. Both the exporter and the provider read `ISE_URL`.
- State is local: `environments/lab/terraform.tfstate`, gitignored. The id from `terraform output` stays in that file.

Phase 3 did not fill `policy/`. It proved the root could create one unused SGT and destroy it. Phase 4 is the first time a YAML row means *Terraform should own this family*.

I am repeating the split because this is the phase where it is tempting to collapse it. A NAD in `exports/nads.csv` starts looking like a resource. A VLAN in `inventory/vlans.yaml` starts looking like an ISE object. A UUID from `terraform output` starts looking like something the review comment should mention. None of those belong in `policy/`.

## YAML is the review surface

I did not teach Terraform to read `policy/*.yaml`. The module does not know which ISE it talks to, and it does not know the file the name came from. `terraform/modules/ndg/` is one `ise_network_device_group`. The lab root calls it once per object I am willing to own.

That is slower than `for_each`. It is also the point. A generated address for every allow-list value would hide the first-write subset inside a loop. I wanted the PR to show eight NDG module calls, not "and whatever else is in the file." The allow-list is the vocabulary. The lab apply is a subset.

Ownership (`owner`, `bu`) stays in the YAML even when ISE has no field for it. The module does not grow a fake attribute so the spreadsheet can feel complete.

After the fills, the linter still only reads `policy/` and `inventory/`:

```text
lint_ise: ok (11 object(s))
```

Eleven is the reviewed set. It is not "everything on the node."

## Why NADs stayed in inventory

A network device is a shared secret, a hostname, and an IP, plus the groups I just spent a family night getting right. Terraform-managing that in this phase would put credentials next to desired policy, and it would make the first cutover a device instead of a name. `docs/decisions.md` already locked this: curated list in `inventory/nads.yaml`, live list in `exports/nads.csv`, not a resource.

The groups those NADs will eventually sit in *are* desired state. The device is not. I would rather have a leaf called `BusinessUnit#BusinessUnit#lab` with nothing assigned to it than import a lab switch so the plan looks busy. Import is Phase 7, and only for objects I am ready to own. A NAD is not one of them.

VLANs took the same exit, for a different reason. `VLAN-lab-data` / id `20` lives in `inventory/vlans.yaml`. The SVI is on the switch. ISE only needs the string the profile will send. Treating the VLAN as an ISE object would be a second source of truth for a number the NAD already owns.

## NDG: the name is three segments, not one

The paper model was five roots. Location and Device Type already exist on ISE. BusinessUnit, Stage, and Function do not. Allow-lists got filled before any write: Location `USA`, `CAN`, `GBR`, `JPN`, `DEU`; Device Type `switch`, `wlc`, `firewall`, `vpn-concentrator`; BusinessUnit `lab`, `hr`, `guest`, `it`; Stage `monitor`, `low-impact`, `closed`; Function `lab`, `test`, `building-mgmt`, `pilot`. No sixth root. No `PS-<bu>-*` join. The BusinessUnit list is filled. The policy-set check that would require a token from it stays off until Phase 5.

I typed the custom root the way the GUI row looks: `BusinessUnit`. *ERS does not store that*. A network device group name has to carry the hierarchy, delimited by `#`. A bare type token returns HTTP 400. The shape that makes the GUI row read `BusinessUnit` is the type token twice:

```hcl
name       = "BusinessUnit#BusinessUnit"
root_group = "BusinessUnit"
is_root    = true
```

A custom leaf is `type#type#value`. Built-in Location and Device Type keep the containers ISE already named. I do not get to rename `All Locations` because my table would prefer `Location#Location`.

| Role | ISE name |
| --- | --- |
| Type container | `BusinessUnit#BusinessUnit`, `Stage#Stage`, `Function#Function` |
| Built-in leaf | `Location#All Locations#USA`, `Device Type#All Device Types#switch` |
| Custom leaf | `BusinessUnit#BusinessUnit#lab`, `Stage#Stage#monitor`, `Function#Function#lab` |

GUI: **Administration > Network Resources > Network Device Groups**.

The module rejects a root that is not `root_group#root_group`, and a leaf that is not three `#` segments starting with that root. Same reason as the SGT regex in Part 3. I would rather fail `plan` than teach a mapper that turns `lab` into `BusinessUnit#BusinessUnit#lab` on the way to ERS. YAML in `policy/ndg.yaml` is the allow-list. The ISE name is the string in `environments/lab/ndg.tf`. They are not the same field, and I am not going to pretend they are.

Second bruise on the same family: a description with `<` or `>` returns an XSS 400. The module rejects those characters. Ownership notes stay in YAML.

`terraform output ndg_ids` stays in state. It does not go into `policy/` or `inventory/`.

## Allowed protocols: the parent boolean is not the object

House style held here. `AP-` plus kebab-case. Two services. `AP-wired-dot1x` is EAP. `AP-wired-mab` is host lookup. Default Network Access is not managed. I am not renaming a built-in so the prefix table looks complete.

ISE 3.3 wants `allow_5g` on every object, including the one that will never speak 5G. I had it as a YAML default of false and still had to send the attribute. Omitting it is not the same as false.

The worse one is the inner object. `allow_peap`, `allow_teap`, and `allow_eap_tls` are not enough when they are true. ERS wants the children: cryptobinding, password-change retries, inner methods. A POST that says "PEAP yes" and then stops is a partial object. The module only sets those inners when the parent is true, so the MAB service does not grow a PEAP block it did not ask for.

```hcl
name          = "AP-wired-dot1x"
allow_eap_tls = true
allow_peap    = true
allow_teap    = true
allow_5g      = false
# inner PEAP / TEAP / EAP-TLS attributes required because those parents are true
```

```hcl
name                = "AP-wired-mab"
process_host_lookup = true
allow_5g            = false
```

MAB on this service is host lookup only. No EAP methods. No PAP. I can add chaining and password-change later. This phase needed the two services a wired policy set would point at, not every knob on the page.

GUI: **Policy > Policy Elements > Results > Authentication > Allowed Protocols**.

## Conditions: attribute only, and the MAB pair is Lookup

Library conditions, not one-off rule conditions. `CND-` plus kebab-case. Two rows. No AND/OR children. No Device Admin condition resource. Network Access only.

`CND-wired-dot1x` is the obvious one: Radius `NAS-Port-Type` equals `Ethernet`.

The MAB one is the bruise. "This is a MAB authentication" is not a dictionary I get to invent. The pair that stored was Network Access `AuthenticationMethod` equals `Lookup`.

```yaml
name: CND-wired-mab
dictionary_name: Network Access
attribute_name: AuthenticationMethod
operator: equals
attribute_value: Lookup
```

AND and OR are how a real rule will combine these. That is a later module change. Putting a compound condition in this phase would have hidden whether the attribute resource worked at all. Same rule as the SGT: one question per apply.

GUI: **Policy > Policy Elements > Conditions > Library Conditions**.

## DACL before profile, and the tunnel tag is not the VLAN

`ACL-permit-all` is a temporary wide-open DACL: `permit ip any any`, type `IPV4`. The description says lab bring-up only. I do not want that string to survive into a story about least privilege. It exists so the profile has something to name.

The profile is `PR-wired-lab-access`. Access type `ACCESS_ACCEPT`. It depends on the DACL module. `dacl_name` is the ACL's name, not an id. No SGT on this profile. The SGT family had not been written into `policy/` yet, and I did not want the first profile to grow a tag binding I could not review on its own.

VLAN is the field I would have gotten wrong from the GUI. `vlan_name_id` is the string the NAD sees. For this lab that string is `20`, which is `VLAN-lab-data` in `inventory/vlans.yaml`. `vlan_tag_id` is the RADIUS tunnel tag. It is `1` on this profile. It is not the VLAN id. Sending `20` in both fields would have been a confident way to black-hole a test port.

```hcl
name         = "PR-wired-lab-access"
access_type  = "ACCESS_ACCEPT"
dacl_name    = module.acl_permit_all.name
vlan_name_id = "20"
vlan_tag_id  = 1
```

GUI: **Policy > Policy Elements > Results > Authorization**.

PermitAccess and DenyAccess stay built-ins. I am not managing them so the prefix table has a row.

## SGTs: the exception, now in `policy/`

Part 3 already paid for this one. Hyphens 400. `[A-Za-z0-9_]`, max 32. No mapper. The linter and the module regex already failed `SGT-lab-users` and `SGT_Lab_Users`.

Phase 4 is the file Part 3 left empty. Two rows:

| ISE name | Tag |
| --- | --- |
| `SGT_lab_users` | 1010 |
| `SGT_lab_devices` | 1020 |

Tags are a lab pick. Do not treat them as a site standard. `SGT_lab_bootstrap` / `1001` stays in `environments/lab/terraform.tfvars`. It is the Phase 3 proof object, not desired policy. Unknown (tag 0) is not managed.

YAML name equals ISE name. The module is the same `ise_trustsec_security_group` from Part 3. I did not widen it.

## EIGs: parent is a human name, UUID stays out

`EIG-printers` and `EIG-lab-iot`. Static groups. No endpoints. I am not importing Unknown, Profiled, Blacklist, GuestEndpoints, or RegisteredDevices so the tree looks finished.

`policy/eig.yaml` has a `parent` field. The value is the human root name, `Endpoint Identity Groups`. The module does not set `parent_endpoint_identity_group_id`. ISE hangs a new group off the root when that attribute is omitted. `system_defined` stays false.

A lookup that wrote the parent UUID into YAML would have solved a paper problem and broken the rule from Part 3: desired state is a name. The UUID is bookkeeping. It lives in state. A later phase can resolve the parent if a group ever needs a non-root parent. This phase did not need that, and I was not going to invent the lookup so `policy/` could hold a string I am not allowed to commit.

GUI: **Work Centers > Network Access > Identities > Endpoint Identity Groups**.

## What is actually on the lab node

Names only. No hostnames, no ids.

| Family | ISE names |
| --- | --- |
| NDG | `BusinessUnit#BusinessUnit`, `Stage#Stage`, `Function#Function`; leaves `Location#All Locations#USA`, `Device Type#All Device Types#switch`, `BusinessUnit#BusinessUnit#lab`, `Stage#Stage#monitor`, `Function#Function#lab` |
| Allowed protocols | `AP-wired-dot1x`, `AP-wired-mab` |
| Conditions | `CND-wired-dot1x`, `CND-wired-mab` |
| DACL / profile | `ACL-permit-all`, `PR-wired-lab-access` (VLAN string `20`, tunnel tag `1`, no SGT) |
| SGT | `SGT_lab_users` / 1010, `SGT_lab_devices` / 1020. Bootstrap stays in tfvars. |
| EIG | `EIG-printers`, `EIG-lab-iot` |

Apply was still the same ritual as Part 3. Source `.env`, then plan, then apply, from `environments/lab/` only. `terraform validate` still does not load the values. The `#` check and the SGT regex fire at `plan`.

## What Phase 4 doesn't do (a good thing)

- Policy sets, ranks, or authentication and authorization rules
- AND/OR library conditions
- An SGT on `PR-wired-lab-access`
- An EIG parent UUID lookup
- Extra PEAP/TEAP knobs (chaining, password change)
- The `PS-<bu>-*` linter join
- `ISS-`, `CAP-`, or `LP-` (logical profiles stay docs-only until that API is proven)
- A NAD module, or an import map
- `environments/stage/` or `prod/`
- A rewrite of `export_ise.py` onto `ciscoisesdk`

Those are later phases. Mixing a policy set into this post would hide the only question this phase was allowed to answer: can the blocks exist, under names ERS will store, without the planes collapsing.

Create-beside, move traffic, disable, delete is still the cutover. These objects are new. Leftover GUI names from the Phase 1 export stay where they are. I did not rename a live object in place as the first write of a family.

## Next

Part 5 is Phase 5: new standard policy sets at the locked ranks. Not an import of the non-standard sets already on the lab. Old GUI sets stay until traffic moves. Then disable. Then delete.

Still lab only. Still no `prod/`. Still no NAD module. The building blocks this post walked are the things those sets will name. They are not the sets.

## Series

- [Part 1 — repo + read-only export](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1)
- [Part 2 — naming linter](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part2)
- [Part 3 — first lab apply](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part3)
- Part 4 — policy building blocks ← this post
- Part 5 — policy sets and ranks (coming soon)

---

*Personal homelab only. Not affiliated with my employer. Hostnames, prefixes, object names, and example policy in this post and the repo are fictional lab material, not an internal standard.*
