---
title: "The Reverse Proxy That Finally Joined the Club"
date: 2026-09-03
draft: false
description: "Rebuilding Nginx Proxy Manager on Proxmox with Terraform and Ansible — including Docker, a wildcard cert, and the headache of a DNS ghost that refused to die."
categories: ["Terraform", "Ansible", "Proxmox"]
tags: ["Terraform", "Ansible", "Proxmox", "NPM", "DNS"]
cover:
  image: "proxy-by-the-lake.jpg"
  alt: "Proxy by the Lake"
  caption: "Proxy Server taking a dip"
---

# The Reverse Proxy That Finally Joined the Club

I already had a working Nginx Proxy Manager box. It did the job. Certificates renewed. Hosts forwarded. I clicked around the UI like my dog finding the toy he hid months ago. I was happy-ish.

The problem was not NPM. The problem was *me*, staring at a container I built with an installer script and zero notes.

DNS had just gone through the Terraform-then-Ansible treatment. NPM was next. Same pattern. Fewer mystery boxes, so I thought...

---

## Github Repos

**Terraform** → [stubborn-packets/proxmox-terraform](https://github.com/stubborn-packets/proxmox-terraform.git)  
**Ansible** → [stubborn-packets/ansible](https://github.com/stubborn-packets/ansible.git)

The post is a snapshot. The repos will lie to future-me more honestly than my memory will.

---

## What “done” looks like

| Piece | Choice |
|-------|--------|
| Host | Proxmox LXC `lab-npm-srv1` |
| IP | `192.168.77.9/24` |
| Software | Nginx Proxy Manager in Docker |
| TLS | Let’s Encrypt wildcard via Cloudflare DNS-01 |
| Service name | `proxy.lab.stubbornpackets.com` (UI) |
| First proxied app | `dns-manager.lab.stubbornpackets.com` → Technitium `:5380` |

Host name is for SSH and Ansible.  
Service name is what I type in a browser.  
I will die on this hill. It is a small hill.

---

## Part 1: Terraform makes the box

This was the easy chapter, which is suspicious.

Copy the DNS layout into `npm/`, point it at a new VMID, skip the VLAN tag (this NIC is untagged on `vmbr0`), inject the lab SSH key, turn nesting on because Docker is coming.

One useful argument with myself: should NPM have two NICs — management vs “the internet-facing lie I tell my homelab”?

Answer for now: **no**. One IP. DNS `A` records for every pretty name point at `192.168.77.9`. Admin UI can hide behind UFW later. Dual-homing is a treat I can give Future Me when Present Me is less busy breaking things or starting yet another project. Honestly it's like walking into a candy store with my attention span and *need* for new projects.

`terraform apply`. Container exists. SSH by IP works. I am briefly happy.

---

## Part 2: My curse of taking bad notes (aka stop messing with local DNS)

SSH by FQDN did not work.

`dig @172.16.99.5 lab-npm-srv1.lab.stubbornpackets.com` said `192.168.77.9`.  
`ping lab-npm-srv1.lab.stubbornpackets.com` said `192.168.78.1`.

🤬 That is the face a network engineer makes when two tools on the same laptop disagree about reality.

The new Technitium server was fine. macOS was still honoring a souvenir I had left in `/etc/resolver/lab.stubbornpackets.com` back when the lab DNS strategy was “please just resolve, it's just DNS and I have other things to do.” 

That file sent the whole zone to the *old* DNS server. Cache flushes did nothing. DHCP changes did nothing. The file does not care about my feelings.

Delete the file. Flush cache. Suddenly the name and the IP live in the same universe. SSH works. I add the host key and pretend this was always the plan.

**Helpful tip if you like having hair:** if `dig` and `ping` argue, look for split-DNS leftovers on the *client* before you rebuild the server.

---

## Part 3: Ansible installs clicky UI

NPM’s happy path is Docker. Nesting was already on. The role does the normal stuff that was already done for the DNS install, pretty much a copy/paste aka *"this is boring, I know there is a better way but I don't have time to rabbithole for a week"*:

1. Packages  
2. Docker Engine + Compose plugin  
3. Compose file under `/opt/nginx-proxy-manager`  
4. UFW: 22, 80, 443, and admin `:81` only from the management subnet  
5. Wait until ports actually listen  

First login is still the default NPM circus (`admin@example.com` / `changeme`). Change it immediately. Then never speak of it again.

Ansible even yelled at me that `apt_repository` is deprecated. Noted. We can be fashionable after the proxy works.

---

## Part 4: Click-ops is a lifestyle, not a strategy

The old NPM had:

- Proxy hosts I created by *hand*  
- A Cloudflare token stuffed into the SSL form. I still can't recall where it is or how I set it up.
- Zero record of either in git

New box gets the same outcome from the API.

NPM speaks REST at `:81/api`. Login gives a JWT. Certificates and proxy hosts are just JSON with opinions.

### Wildcard cert

Let’s Encrypt will not HTTP-01 a `*.lab.stubbornpackets.com` name that only exists on my shelf in my office. DNS-01 + Cloudflare will.

The first API payload failed with a very polite 400:

> `data/meta must NOT have additional properties`

Which is software for “I asked for three fields and you brought the whole junk drawer.” Newer NPM does not want `letsencrypt_email` and `letsencrypt_agree` in `meta`. Strip those. Request succeeds. Certificate ID `1`. The UI shows a wildcard that is not attached to anything yet.

Cloudflare token lives in `secrets.yml`. The example file has a fake one. Git sees only the fake one. *This is the way.*

### First proxy host

`dns-manager.lab.stubbornpackets.com` → `172.16.99.5:5380`, slap cert `1` on it, force SSL.

Re-run the playbook: existing cert is skipped, existing host is skipped. Add a line to `npm_proxy_hosts` when I get around to building the next host.

WebSockets? Only if the app actually upgrades the connection (Proxmox, dashboards, the NPM UI itself). Technitium does not need it.

---

## The mental model, one more time

| Layer | Tool | Job |
|-------|------|-----|
| LXC | Terraform | Exists, has an IP, boots, accepts my key |
| Packages / Docker / firewall | Ansible | The box is usable |
| Certs + proxy hosts | Ansible + NPM API | The names work in a browser |
| Pretty DNS names | Technitium (also Ansible) | `A` records point at NPM |

Terraform will delete a container if you remove it from config.  
Ansible will **not** delete an NPM host if you remove it from the list. Same lesson as DNS records. Desired-state-lite. Fine for a lab. Do not confuse it with Terraform.

---

## What this actually buys the homelab

- New app? A record + one YAML object + playbook. No clicking through SSL wizards at 11pm.  
- Box dies? Terraform rebuilds the LXC, Ansible rebuilds NPM, API rebuilds the hosts.  
- Secrets stay out of git.  
- The old “installer script and vibes” NPM can retire when the last host has moved.

Is it overkill for a container that forwards ports? Yes.  
Will I remember how I built it in February? Also yes (*thankfully this post exists*), and that is the entire product.

---

## Next

- fix the `apt_repository` is deprecated issues
- Keep adding hosts as services move over → I'm eyeing my CML VM next
- Point `proxy.lab.stubbornpackets.com` at NPM’s own UI (done ✅)
- Eventually retire the old NPM host once everything is moved off
- NetBox, someday, when I want IPAM to stop living in my head  

Terraform for the box. Ansible for what runs on it. API for the clicks I do not want to do twice.

---

*Lab network only. Hostnames and RFC1918 addresses are on purpose.*