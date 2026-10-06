---
title: "Cisco ISE Policy-as-Code Migration - Part 5"
date: 2026-10-06
draft: false
description: "Phase 5 put eight policy sets on the lab, all disabled. YAML rank stayed the design band. ISE rank is a packed insert index. Enable is the cutover."
categories: ["Terraform", "Cisco", "ISE"]
tags: ["Terraform", "Cisco", "ISE", "Policy-as-Code"]
cover:
  image: "ise-pac-part5-hero.jpg"
  alt: "Homelab desk with a disabled ISE policy set list beside a YAML rank table, sticky note reading disabled is not cutover"
---

# Cisco ISE Policy-as-Code Migration - Part 5

Part 4 left the blocks a policy set would call. Network device groups, allowed protocols, library conditions, a DACL and a profile, two SGTs, two static endpoint identity groups. YAML in `policy/` was the review surface. The modules stayed thin. No policy set. That was the point of the phase.

Part 5 is the sets. Eight of them, on lab ISE, all disabled. Infra, wireless 802.1X, wireless MAB, VPN, wired 802.1X, wired MAB, then the first business-unit pair: `PS-hr-wired` and `PS-guest-wired`. Network Access only. `CiscoDevNet/ise` 0.4.1. Local state. Exit met 2026-10-05.

This series is still the same question as Part 1: *is Policy-as-Code on ISE 3.x a real operating model, or a Cisco Live slide*. Phase 4 answered "can the blocks exist as reviewed YAML." Phase 5 answers "can a policy set be created beside the old GUI sets, disabled, without pretending the rank table in the docs is the number ERS stores." A plan is a plan. The rank table was wrong. There were a few roadblocks along the way, as the plan acted like a plan, and ISE and the documentation I created disagreed. 

I used Grok AI the same way as Part 4. Direction and questions stayed mine. Each family was walked in pieces so I could argue with it — *why is this rank rejected, can the module stay one resource, what does not belong in `policy/`*. A generated tree of every ISE object in one paste is how I stop learning.

The post is a snapshot. The repo will move as later phases land.

**Note:** *This is a personal homelab project. Prefixes, object names, and example policy shapes are fictional. They came from public posts on this topic and from what I think a brownfield lab should look like so the export is interesting. They are not copied from an internal standard and they are not names used by my employer. Nothing here is employer policy, employer architecture, or employer data.*

*Opinions are mine.*

---

## Github Repos

**cisco-ise** → [stubborn-packets/cisco-ise](https://github.com/stubborn-packets/cisco-ise)

*The post is a snapshot and the repos are live and will be updated as I progress through the phases. Things you see in the repos may have items that have been updated since this post.*

Main changes in this phase:

- `policy/policy-sets.yaml` filled. Rank there is still the old design band.
- Thin modules under `terraform/modules/{policy-set,authentication-rule,authorization-rule}/`
- Lab calls in `environments/lab/{policy-sets,authn-rules,authz-rules}.tf`
- The conditions, profiles, allowed protocols, NDGs, and EIGs those sets name

The cloneable artifact is those modules plus the lab calls. Not a generated tree. There is no `for_each` over YAML. A reviewer reads the YAML. Terraform writes one object per module call. The lab file is still copied. That copy is the drift this phase did not close.

`it`, `PS-<bu>-vpn`, and `PS-<bu>-wireless` are not on that list. A lab with nothing pointed at it does not gain a pattern from a copy.

---

## Where Part 4 stopped

[Part 1](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1) was the repo and the read-only export. [Part 2](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part2) was the linter. [Part 3](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part3) was the first write. [Part 4](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part4) was the building blocks.

- Three planes that stay split: `policy/` (desired), `inventory/` (reference), `exports/` (observed, gitignored)
- One lab ISE under `environments/lab/`. No `prod/` folder. No `stage/` folder unless a second eval node shows up.
- `{PREFIX}-` plus kebab-case, except SGTs. SGT is `SGT_` plus lowercase snake_case, max 32. YAML name equals the ISE name. No mapper.
- `scripts/lint_ise.py` reads `policy/` and `inventory/` only. No HTTP client. No Terraform.
- Provider pin is `CiscoDevNet/ise` 0.4.1. Credentials stay in repo-root `.env`. Both the exporter and the provider read `ISE_URL`.
- State is local: `environments/lab/terraform.tfstate`, gitignored. The id from `terraform output` stays in that file.

Phase 4 did not create a policy set. Phase 5 is the first time a YAML row means *this set should exist, and it should stay disabled*.

## YAML is the review surface

I did not teach Terraform to read `policy/policy-sets.yaml`. `terraform/modules/policy-set/` is one `ise_network_access_policy_set`. The lab root calls it once per set I am willing to own. Required arguments are the name and the allowed-protocols service name. `is_proxy` is false. I do not set `default`. Rank 99 Default is built-in. I am not renaming it so the table has a row.

The condition on the set is a reference, not an inline attribute. The lab call passes `condition_id` from the condition module output. That id stays in state. It does not go into `policy/` or `inventory/`.

Authentication and authorization rules are separate resources, keyed by the policy set id from that module. `ise_network_access_authentication_rule` wants the name, the policy set id, and the three `if_*` actions. `ise_network_access_authorization_rule` wants the name, the policy set id, and a set of profile names. Not profile ids. I do not set `default` on either rule, and I do not set `security_group` on the first authorization rule.

That is slower than `for_each`. It is also the point. A generated address for every row would hide the first-write subset inside a loop. I wanted the PR to show eight policy-set module calls, not "and whatever else is in the file."

After the guest and HR sets, the linter still only reads `policy/` and `inventory/`:

```text
lint_ise: ok (57 object(s))
```

Fifty-seven is the reviewed set. It is not "everything on the node."

## Rank is an insert index, not a band

The paper model was a spacing convention. Infra in 1–9. VPN at 10. Wireless at 20 and 30. Wired at 40 and 50. Business-unit VPN at 60, wireless at 70, wired at 80. Default at 99. Lower number, evaluated first, with gaps so a later set could land between them.

ISE does not store those numbers. Policy-set rank is a packed insert index. The first apply posted `PS-global-wired-8021x` at rank 40. The node rejected it. The legal range on this lab started at 0. There were not forty sets to insert in front of. Default cannot be moved to 99. I am not going to manage the built-in so the YAML can keep a row called 99.

YAML rank stayed the old band. The lab call sends the index ERS will accept. Those are not the same field, and I am not going to pretend they are. The mismatch is the next phase, not a bug to hide in this post.

```hcl
name         = "PS-global-wired-8021x"
rank         = 6
state        = "disabled"
service_name = module.ap_wired_dot1x.name
condition_id = module.cnd_wired_dot1x_framed.id
is_proxy     = false
```

`policy/policy-sets.yaml` still says rank 40 for that set. The 6 is the packed index after the later inserts. A reviewer who only reads the YAML will think the set is evaluated at 40. A reviewer who only reads `terraform output` will think it is evaluated at 6. Neither number is the GUI order. That is the next hurdle.

## AuthenticationMethod is illegal on a policy set

Part 4 stored `CND-wired-mab` as Network Access `AuthenticationMethod` equals `Lookup`. That pair is a fine authentication-rule condition. It is not a policy-set condition. The set POST rejected it.

The wired set conditions are Radius attributes the packet already has. `NAS-Port-Type` plus `Service-Type`. Wired 802.1X is `Ethernet` and `Framed`. Wired MAB is `Ethernet` and `Call Check`. VPN is `NAS-Port-Type` equals `Virtual`, which can stay a single attribute. Wireless port type is `Wireless - IEEE 802.11`.

`CND-wired-mab` stays the authentication-rule condition on `AN-wired-mab`. The set points at `CND-wired-mab-call-check`. Same idea, different dictionary, because the policy set is matched before authentication has a method to report.

GUI: **Policy > Policy Sets**.

## Device Type is not a Network Access attribute

The infra set needed "this NAD is a load balancer." I typed the dictionary the way the GUI label reads: Network Access, Device Type. That pair is illegal on a library condition.

The NAD group condition uses dictionary `DEVICE`. The value is not the bare leaf. Device Type is `All Device Types#load-balancer`. BusinessUnit is `BusinessUnit#global`, not `global`. Same `#` shape Part 4 already paid for on the network device group name. I had hoped the condition value could be the short token. ERS stores the hierarchy.

```hcl
condition_type  = "ConditionAttributes"
dictionary_name = "DEVICE"
attribute_name  = "Device Type"
attribute_value = "All Device Types#load-balancer"
```

```hcl
condition_type  = "ConditionAttributes"
dictionary_name = "DEVICE"
attribute_name  = "BusinessUnit"
attribute_value = "BusinessUnit#global"
```

The review YAML still has the bare leaf on some of those rows. The lab call does not. I am not going to call the YAML the thing ERS stored until those strings match.

## Insert at 0 shifts everything already there

A new set does not land in a gap. It lands at an index, and every managed set at that index or after it moves down. The first VPN apply inserted `PS-global-vpn` at 0. The wired sets that were already there shifted. Infra, wireless, guest, and HR did the same thing later. Each insert was a renumber of `environments/lab/policy-sets.tf` in the same change. `depends_on` chains the inserts so they do not race.

YAML rank did not move. `PS-hr-wired` is still 80 in `policy/policy-sets.yaml`. On the node it is index 0, because it was the last insert at the front. `PS-guest-wired` is 81 in YAML and 1 on the node. VPN is 10 in YAML and 5 on the node. Wired 802.1X is 40 in YAML and 6 on the node.

| Name | YAML rank | ISE index |
| --- | --- | --- |
| `PS-hr-wired` | 80 | 0 |
| `PS-guest-wired` | 81 | 1 |
| `PS-infra-health-checks` | 1 | 2 |
| `PS-global-wireless-8021x` | 20 | 3 |
| `PS-global-wireless-mab` | 30 | 4 |
| `PS-global-vpn` | 10 | 5 |
| `PS-global-wired-8021x` | 40 | 6 |
| `PS-global-wired-mab` | 50 | 7 |

That table is the bruise. The right-hand column is what this lab accepted. The left-hand column is what the docs still call the design band. Closing the gap means editing `rank` on the sets that shift, in the YAML, and stopping the second hidden number. Not in this post.

## Unmanaged sets still occupy slots

The lab already had non-standard GUI policy sets. Create-beside is still the cutover from Part 1. I did not import them. I did not rename them in place.

Terraform rank output does not show them. The eight indexes above are the managed inserts. They are not "the first eight rows a packet hits." GUI order is the evaluation order. Old sets still sit between managed sets. A plan that says rank 0 is not a promise that the set is first.

I would rather have a disabled standard set with a leftover GUI set above it than import a name I am not ready to own. Import is a later phase, and only for objects I am ready to own. A non-standard policy set is not one of them.

## Guest and HR have to match before global MAB

A global wired MAB set whose condition is only `Ethernet` plus `Call Check` matches an HR NAD and a guest NAD. Those sets never get a turn. Guest and HR need their own business unit on the policy-set condition, and they need to be inserted in front.

The global wired conditions now require `DEVICE` `BusinessUnit` `BusinessUnit#global`. HR requires `BusinessUnit#hr`. Guest requires `BusinessUnit#guest`. Without that child, the business-unit set is decorative.

Authentication on those two sets still searches `Internal Endpoints`. Unknown MAC is `CONTINUE`, not `REJECT`. Authorization is the EIG check. `AZ-hr-wired` returns `PR-hr-wired-mab` when `EIG-hr` matches. Guest returns `PR-guest-wired-mab`. VLAN string `30` for HR, `40` for guest, tunnel tag `1` on both. The VLAN id is still the inventory number. It is not the tunnel tag. No SGT.

```text
AN-hr-wired     Internal Endpoints    REJECT / DROP / CONTINUE
AZ-hr-wired     EIG-hr                PR-hr-wired-mab
AN-guest-wired  Internal Endpoints    REJECT / DROP / CONTINUE
AZ-guest-wired  EIG-guest             PR-guest-wired-mab
```

802.1X and VPN still search `Internal Users` and reject if the user is not found. Wired and wireless MAB continue. The global authorization rules return `PR-wired-lab-access`: VLAN string `20`, `ACL-permit-all`, no SGT. That profile is a lab bring-up result. It is not a story about least privilege.

`EIG-hr` and `EIG-guest` exist. No endpoint was put in either group. An empty group is the object. A populated group is a cutover.

## ConditionAttributes is the child type

AND showed up in this phase because a policy-set condition is two or three attributes, not one. The module default for a single-attribute call is `LibraryConditionAttributes`. A call that omits `condition_type` gets that default. That is the right type for `CND-vpn` and for `CND-wired-mab`.

Inside an AND block the child type is `ConditionAttributes`. Not the library type. The parent is `LibraryConditionAndBlock`. Sending the library type on the child, or the child type on the parent, is a different object than the one the GUI draws.

```hcl
name           = "CND-wired-dot1x-framed"
condition_type = "LibraryConditionAndBlock"
children = [
  {
    condition_type  = "ConditionAttributes"
    dictionary_name = "Radius"
    attribute_name  = "NAS-Port-Type"
    attribute_value = "Ethernet"
  },
  {
    condition_type  = "ConditionAttributes"
    dictionary_name = "Radius"
    attribute_name  = "Service-Type"
    attribute_value = "Framed"
  },
  {
    condition_type  = "ConditionAttributes"
    dictionary_name = "DEVICE"
    attribute_name  = "BusinessUnit"
    attribute_value = "BusinessUnit#global"
  },
]
```

GUI: **Policy > Policy Elements > Conditions > Library Conditions**.

## A timer is not a timer by itself

`PR-global-vpn` returns Class `ou=GP-global-vpn` and a session timeout of `28800`. No VLAN. The Class string is what the concentrator reads as the group policy. I am not managing the group policy. I am sending the attribute.

A reauth timer without `reauthentication_connectivity` does not store. The lab value is `DEFAULT`. The module only sends the pair when the timer is set, so the wired profile does not grow a reauth block it did not ask for.

```hcl
name                          = "PR-global-vpn"
access_type                   = "ACCESS_ACCEPT"
dacl_name                     = module.acl_permit_all.name
asa_vpn                       = "ou=GP-global-vpn"
reauthentication_timer        = 28800
reauthentication_connectivity = "DEFAULT"
```

Infra is the other new profile. `PR-infra-permit` is access accept only. No VLAN, no DACL, no SGT, no Class. I did not point the health-check rule at the built-in PermitAccess profile so the prefix table could skip a row. PermitAccess stays a built-in. The set uses `AP-infra-health`, PAP only. The condition matches Device Type `All Device Types#load-balancer` and a lab username. The user was not created. A condition that names a user is not the user.

GUI: **Policy > Policy Elements > Results > Authorization > Authorization Profiles**.

## Disabled is not the cutover

Every managed set is `state = "disabled"`. Enable is the cutover. It is not part of this phase.

Nothing is pointed at these sets yet. The F5 user was not created. No NAD was placed in `BusinessUnit#global`, `BusinessUnit#hr`, `BusinessUnit#guest`, or `All Device Types#load-balancer`. No endpoint was put in an EIG. A disabled set with an empty group is a name I can review. An enabled set with no device behind it is a story I cannot test.

`it` is on the BusinessUnit allow-list. There is no `PS-it-wired`. Business-unit VPN and business-unit wireless were not written. Copying the wired pair onto a service this lab does not speak would have made the tree look finished. It would not have taught me anything the HR set had not already taught.

## What is actually on the lab node

Names only. No hostnames, no ids.

| Family | ISE names |
| --- | --- |
| Policy sets | `PS-hr-wired`, `PS-guest-wired`, `PS-infra-health-checks`, `PS-global-wireless-8021x`, `PS-global-wireless-mab`, `PS-global-vpn`, `PS-global-wired-8021x`, `PS-global-wired-mab`. All disabled. |
| Authentication | `AN-hr-wired`, `AN-guest-wired`, `AN-wireless-mab`, `AN-wired-mab` (`Internal Endpoints`, `REJECT` / `DROP` / `CONTINUE`). `AN-infra-f5-health`, `AN-wireless-dot1x`, `AN-vpn`, `AN-wired-dot1x` (`Internal Users`, `REJECT` / `DROP` / `REJECT`). |
| Authorization | `AZ-hr-wired` → `PR-hr-wired-mab`. `AZ-guest-wired` → `PR-guest-wired-mab`. `AZ-infra-f5-health` → `PR-infra-permit`. `AZ-vpn` → `PR-global-vpn`. Wireless and wired `AZ-` rules → `PR-wired-lab-access`. No SGT. |
| Conditions added | `CND-wired-dot1x-framed`, `CND-wired-mab-call-check`, `CND-vpn`, `CND-wireless-dot1x-framed`, `CND-wireless-mab-call-check`, `CND-infra-f5-health`, `CND-hr-wired-mab-call-check`, `CND-guest-wired-mab-call-check`, `CND-hr-eig`, `CND-guest-eig` |
| Profiles added | `PR-global-vpn`, `PR-infra-permit`, `PR-hr-wired-mab` (VLAN string `30`), `PR-guest-wired-mab` (VLAN string `40`) |
| Also named | `AP-infra-health`, `AP-vpn`, `AP-wireless-dot1x`, `AP-wireless-mab`, `EIG-hr`, `EIG-guest`, `BusinessUnit#BusinessUnit#global`, `BusinessUnit#BusinessUnit#hr`, `BusinessUnit#BusinessUnit#guest`, `Device Type#All Device Types#load-balancer` |

Inventory only: `VLAN-lab-data` / 20, `VLAN-hr-data` / 30, `VLAN-guest-data` / 40 in `inventory/vlans.yaml`. Those are not ISE objects.

Apply was still the same ritual as Part 3. Source `.env`, then plan, then apply, from `environments/lab/` only. `terraform validate` still does not load the values.

## What Phase 5 doesn't do (a good thing)

- Enable any managed set
- Create the F5 user, place a NAD, or put an endpoint in an EIG
- `it`, `PS-<bu>-vpn`, or `PS-<bu>-wireless`
- Import the old GUI sets, or move Default to 99
- An SGT on any profile
- Manage PermitAccess
- `environments/stage/` or `prod/`
- A NAD module, or an import map

Those are later phases, or they are not this lab. Mixing an enable into this post would hide the only question this phase was allowed to answer: can the sets exist, disabled, under names ERS will store, with the rank bruise still visible.

## Next

The YAML rank and the ISE index disagree. That split is the next phase. The numbers in `policy/policy-sets.yaml` are still the old bands. The lab file holds the packed index. I am not going to paper over that in the post and call the table done.

Sets stay disabled. Enable is the cutover. Still lab only. Still no `prod/`. Still no NAD module. The sets this post walked are names. They are not traffic.

## Series

- [Part 1 — repo + read-only export](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part1)
- [Part 2 — naming linter](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part2)
- [Part 3 — first lab apply](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part3)
- [Part 4 — policy building blocks](https://stubbornpackets.com/blog/cisco-ise-policy-as-code-part4)
- Part 5 — policy sets, disabled ← this post

---

*Personal homelab only. Not affiliated with my employer. Hostnames, prefixes, object names, and example policy in this post and the repo are fictional lab material, not an internal standard.*
