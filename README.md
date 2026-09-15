# Tailscale Homelab

A hands-on lab for learning Tailscale deeply - not just installing it, but
building it, breaking it, and diagnosing it.

## Why this exists

Reading docs is not the same as understanding a system. This repo documents a
lab where each Tailscale feature gets built, deliberately broken, and
diagnosed - because working configs teach very little, and failed ones teach
the diagnostic path.

The focus is on what happens underneath: WireGuard, NAT traversal, and why
some connections go direct while others land on a relay.

## Lab architecture

| Node | Role | Notes |
|------|------|-------|
| Windows laptop | Client | Behind home NAT |
| Ubuntu Server 26.04 LTS (VM) | Subnet router / exit node | Bridged adapter, home LAN |
| Hetzner VPS (Ubuntu 26.04 LTS) | Public node | Public IP, no NAT - used to compare direct vs relayed paths |
| Phone | Mobile client | Tested on cellular to force out-of-network paths |

Having one node behind NAT and one with a public IP is intentional: it makes
the difference between direct connections and DERP-relayed connections
observable rather than theoretical.

## Method

Every exercise follows the same loop:

1. **Predict** - write down what I expect the command or config to do
2. **Run** - execute it
3. **Compare** - record what actually happened, especially where it differed
4. **Break** - disable, misconfigure, or remove something on purpose
5. **Diagnose** - work the failure using `tailscale status`, `ping`, `netcheck`,
   `debug`, and logs before looking anything up
6. **Document** - write it up as Issue â†’ Cause â†’ Investigation â†’ Resolution

Step 4 is the point.

## Troubleshooting notebook

Entries live in [`notebook/`](notebook/). Each one documents a real failure in
this lab and how it was diagnosed.

| # | Issue | Area |
|---|-------|------|
| _(entries added as the lab progresses)_ | | |

## Roadmap

- [ ] Tailnet foundation: multi-device install, MagicDNS, admin console
- [ ] Ubuntu Server VM as a Linux node (headless, SSH access)
- [ ] Hetzner VPS joined to the tailnet
- [ ] Direct vs DERP: comparing `netcheck` across NAT'd and public nodes
- [ ] Subnet router advertising the home LAN
- [ ] Exit node routing all client traffic
- [ ] ACLs: tags, groups, and autogroups
- [ ] Tailscale SSH, with public SSH port closed at the cloud firewall
- [ ] Serve and Funnel
- [ ] Tailscale Kubernetes operator

## Concepts

Short write-ups of the mechanisms behind the lab - control plane vs data
plane, NAT traversal and hole punching, why DERP exists and why relaying
doesn't compromise end-to-end encryption - live in [`concepts/`](concepts/).

## Status

Active. Started September 2026.

