# Tailscale Homelab

A hands-on lab for learning Tailscale deeply - not just installing it, but building it, breaking it, and diagnosing it.

## Why this exists

Reading docs is not the same as understanding a system. This repo documents a lab where each Tailscale feature gets built, deliberately broken, and diagnosed - because working configs teach very little, and failed ones teach the diagnostic path.

The focus is on what happens underneath: WireGuard, NAT traversal, and why some connections go direct while others land on a relay.

## Lab architecture

| Node | Role | Notes |
|------|------|-------|
| Windows laptop (`teham`) | Client, VM host | Moves between home and office networks |
| Ubuntu Server 26.04 LTS VM (`ubuntuservervmvbox`) | Subnet router for the home LAN | VirtualBox, bridged adapter |
| Hetzner VPS, Ubuntu 26.04 LTS (`ts-vps-01`) | Exit node, tagged `tag:server` | Public IP, no NAT, no public SSH port |
| Android phone (`els26ultra`) | Mobile client | Tested on cellular to force out-of-network paths |
| Chromebook (`penguin`, `strongbad`) | Client | Crostini Linux and Android app, see entry 05 |

Having one node behind NAT and one with a public IP is intentional: it makes the difference between direct connections and DERP-relayed connections observable rather than theoretical.

## Method

Every exercise follows the same loop:

1. **Predict** - write down what I expect the command or config to do
2. **Run** - execute it
3. **Compare** - record what actually happened, especially where it differed
4. **Break** - disable, misconfigure, or remove something on purpose
5. **Diagnose** - work the failure using `tailscale status`, `ping`, `netcheck`, `debug`, and logs before looking anything up
6. **Document** - write it up as Issue → Cause → Investigation → Resolution

Step 4 is the point.

## Troubleshooting notebook

Each entry documents a real failure in this lab and how it was diagnosed.

| # | Issue | Area |
|---|-------|------|
| 01 | [Commands failing in the Hetzner browser console](notebook/01-console-keyboard-layout-mangling-commands.md) | Linux console |
| 02 | [Device showing as expired and unable to connect](notebook/02-expired-node-key.md) | Node keys |
| 03 | [Comparing direct and relayed paths across three networks](notebook/03-derp-vs-direct-paths.md) | NAT traversal, DERP |
| 04 | [A subnet route needs two separate things to be true](notebook/04-subnet-router-two-conditions.md) | Subnet routing |
| 05 | [Why RDP lags from a Chromebook over Tailscale (open investigation)](notebook/05-rdp-lag-chromeos-cgnat-overlap.md) | ChromeOS, NAT |
| 06 | [Tailscale on Windows kept stopping right after starting](notebook/06-windows-tailscale-wont-stay-connected.md) | Windows client |
| 07 | [Exit node approved, but not available to my laptop](notebook/07-exit-node-approved-but-not-available.md) | Exit nodes, policy |

## Roadmap

- [x] Tailnet foundation: multi-device install, MagicDNS, admin console
- [x] Hetzner VPS joined to the tailnet
- [x] Direct vs DERP: comparing `netcheck` across NAT'd and public nodes
- [x] Ubuntu Server VM as a Linux node (headless, SSH access)
- [x] Subnet router advertising the home LAN
- [x] Tailscale SSH, with public SSH port closed at the cloud firewall
- [x] Exit node routing all client traffic
- [ ] Grants: tags, autogroups, and a restrictive policy (in progress)
- [ ] Host firewall: does ufw on a node actually filter tailnet traffic?
- [ ] Password logins disabled on the VM, so Tailscale SSH is the only way in

### Later

- [ ] MagicDNS resolution issues on Ubuntu
- [ ] A cloned VM showing up with a duplicate node key
- [ ] Peer Relays on the VPS
- [ ] Serve and Funnel
- [ ] Tailscale Kubernetes operator

## Status

Active. Started September 2026.
