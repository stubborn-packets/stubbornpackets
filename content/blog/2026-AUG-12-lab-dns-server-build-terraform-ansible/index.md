---
title: "Building my Homelab DNS Server with Terraform and Ansible on Proxmox"
date: 2026-08-13
draft: true
description: "Replacing a *hand-built* Technitium DNS server with a Proxmox LXC managed by Terraform and configured by Ansible."
categories: ["Terraform", "Ansible", "Proxmox"]
tags: ["Terraform", "Ansible", "Proxmox"]
cover:
  image: "dns-server-lost-in-space.jpg" 
  alt: "DNS in Space"
  caption: "Giving the DNS server some Space"
---

# Building Homelab DNS with Terraform and Ansible

Replacing a *"hand-built"* Technitium DNS server with a Proxmox LXC managed by Terraform and configured by Ansible.

---

For a while my lab DNS was a Technitium instance I had stood up with an auto-installer script inside an LXC and Proxmox. It worked... but it lived outside any real infrastructure-as-code workflow. The long-term goal is simple:

- **Terraform** owns the VM/LXC lifecycle  
- **Ansible** owns the software and configuration  
- Later, **NetBox** can become the source of truth for IPs and documentation  

This post covers the first concrete step: rebuilding Technitium as a Terraform-managed LXC and driving install plus config with Ansible.

---
## Github Repos
Below are the two GitHub repos with all the code used in this post. These Repos are also where you can find the latest code updates for anything with my home lab, so there might be more recent changes than what's in this post.

**Terraform** → [stubborn-packets/proxmox-terraform](https://github.com/stubborn-packets/proxmox-terraform.git)
**Ansible** → [stubborn-packets/ansible](https://github.com/stubborn-packets/ansible.git)

---

## End state

| Piece | Choice |
|-------|--------|
| Host | Proxmox LXC on `pve` |
| Hostname | `lab-dns-srv1.lab.stubbornpackets.com` |
| IP | `172.16.99.5/24` (DMZ VLAN) |
| DNS software | [Technitium DNS Server](https://technitium.com/dns/) |
| Forwarders | Cloudflare + Quad9 over DNS-over-HTTPS |
| Zones | `lab.stubbornpackets.com` + reverse zones for lab subnets |
| Blocking | Hagezi Pro, OISD Big, URLhaus |
| Web UI access | `dns-manager.lab.stubbornpackets.com` → Nginx Proxy Manager |
| Direct / SSH | `lab-dns-srv1.lab.stubbornpackets.com` → container IP |

Splitting the **host** name from the **service** name keeps SSH and management on the real box while the browser UI goes through NPM for certificates.

---

## Part 1: Terraform — the LXC

I used the [bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest/docs) provider. The important bits for an LXC are:

- Ubuntu 24.04 template (downloaded with `pveam` once permissions were sorted)
- Static IP, gateway, DNS, VLAN tag
- SSH public key injected via cloud-init style initialization 
- `start_on_boot = true` (not `on_boot` — that argument name does not exist on this resource)

API tokens need real rights. A token without enough privileges fails with HTTP 403 on create/power operations. Assigning an appropriate role on the token fixed that.

SSH keys in the lab were worth doing first. Passwordless root login from my laptop is a huge time-saver, and I could use it to run Ansible into the new container.

Project layout stays simple: one area for this DNS host (and room later for NPM, CML, etc.), with secrets in `terraform.tfvars` (gitignored) and a `.example` file committed.

After `terraform apply`, the container existed, answered on its IP, and accepted the SSH key. Infrastructure side: **done**.

---

## Part 2: Ansible — install and harden

Ansible is new to me, so the role was built in layers.

### Role layout

```text
roles/technitium/
├── defaults/main.yml
├── handlers/main.yml
├── meta/main.yml
├── tasks/
│   ├── main.yml        # includes the rest
│   ├── packages.yml
│   ├── resolved.yml
│   ├── install.yml
│   ├── firewall.yml
│   ├── configure.yml   # Technitium HTTP API
│   └── verify.yml
└── README.md
```

Playbooks stay thin:

```yaml
- name: Configure Technitium DNS servers
  hosts: dns
  become: true
  vars_files:
    - "{{ playbook_dir }}/../secrets.yml"
  roles:
    - technitium
```

### What the install path does

1. Apt update + packages (`curl`, `ca-certificates`, `libicu74`, `ufw`)
2. Stop/disable `systemd-resolved` so port 53 is free
3. Point `/etc/resolv.conf` at `127.0.0.1`
4. Run the official Technitium installer only if the binary is not already present
5. Enable and start the `dns` service
6. UFW: allow 22/tcp, 53/tcp, 53/udp, 5380/tcp, default deny incoming. ICMP is allowed by default
7. Verify service active and port 5380 listening

`roles_path = roles` in `ansible.cfg` matters when the playbook lives under `playbooks/` and the role lives at the repo root. Without it, Ansible looks for roles next to the playbook and fails.

---

## Part 3: Configure via the Technitium API

Once the service was up, configuration moved to the HTTP API so it stays in git instead of manual mouse clicks.

### Auth

Create a non-expiring API token in the UI (or via `/api/user/createToken`). Store it in a local `secrets.yml` that is **gitignored**:

```yaml
# secrets.yml (not committed)
technitium_api_token: "..."
```

```yaml
# secrets.yml.example (committed)
technitium_api_token: "paste-token-here"
```

The playbook loads `secrets.yml` with `vars_files`. Defaults reference the variable; nothing secret is committed.

### Forwarders

DoH to Cloudflare and Quad9, concurrent forwarding enabled. The `uri` module needs a careful form body (a multiline folded scalar introduced a space into the domain name and broke record creates — single-line form bodies were more reliable).

### Zones and records

Primary zones:

- `lab.stubbornpackets.com`
- Reverse for `172.16.99.0/24`, `192.168.5.0/24`, `192.168.77.0/24`, `192.168.78.0/24`

The API accepts CIDR for reverse zones and creates the correct `in-addr.arpa` names. Primary reverse zones are appropriate when you own those subnets in a lab.

Records (examples):

| Name | Target | Purpose |
|------|--------|---------|
| `lab-dns-srv1.lab.stubbornpackets.com` | `172.16.99.5` | Host / SSH |
| `dns-manager.lab.stubbornpackets.com` | NPM IP | Web UI via proxy |

PTR for the host record is created with the A record when `ptr=true`.

### Blocking

Three lists, balanced for home + lab:

- [Hagezi Multi Pro](https://github.com/hagezi/dns-blocklists)
- [OISD Big](https://oisd.nl/)
- [URLhaus](https://urlhaus.abuse.ch/)

Enabled via `enableBlocking` and `blockListUrls` on the settings API. Start moderate; allowlist anything that breaks after clients move over.

---

## Terraform vs Ansible mental model

Terraform reconciles managed objects: remove from config → destroy remote.

These Ansible tasks are mostly **ensure present / set to this value**:

- Removing a record from `technitium_records` does **not** delete it on the server
- Changing an existing name with `overwrite=true` updates it
- Settings like forwarders and block lists are replaced by whatever the API call sends

For a small lab zone that is fine: add and update in Ansible, delete rarely used names by hand or with a one-off API call.

---

## Cutover plan

No big-bang cutover.

1. Point lab / IoT DHCP at `172.16.99.5` and soak for a few days  
2. Move home SSID/scopes the same way  
3. When nothing important still uses the old server, decommission it  

That keeps blast radius small and makes allowlisting straightforward if a block list is too aggressive. Also prevents me from hearing everyone in my house complain that they can't get to Instagram or watch Netflix.

---

## Repo hygiene

- `.gitignore`: `secrets.yml`, `.env`, `*.tfvars` (in the Terraform repo), `.DS_Store`, Ansible retry files  
- Example secret files committed; real secrets only on disk  
- README documents structure, secrets, and how to run the playbook  

Same idea as Terraform `tfvars` + example and Python `dotenv` + `.env-example`.

---

## What is next

- Keep soaking lab traffic on the new DNS  
- Add remaining host records as needed  
- Ansible role for Nginx Proxy Manager  
- Eventually NetBox for IPAM and docs  

The important part was establishing the pattern: **Terraform for the box, Ansible for what runs on it**, with secrets out of git and config driven by the Technitium API instead of only the web UI.

---

*Lab network only; no public exposure of internal services. Hostnames and RFC1918 addresses in examples are intentional for a homelab write-up.*