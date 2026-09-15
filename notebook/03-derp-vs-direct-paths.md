# Comparing direct and relayed paths across three networks

## What I was testing

Tailscale connects peers directly where it can and falls back to a DERP relay where it cannot. I wanted to see both happen, understand what decides which one you get, and measure what the difference actually costs.

## Environment

- `teham` - Windows laptop, home network behind NAT
- `els26ultra` - Android phone, tested on both cellular and home Wi-Fi
- `ts-vps-01` - Ubuntu 26.04 LTS on a Hetzner VPS in Nuremberg, public IP, cloud firewall with no inbound rules
- All three nodes in one tailnet under a single account

## Baseline: what netcheck reports

`tailscale netcheck` profiles the network in front of a node. It works the same way STUN does in VoIP - it asks an external server "what address do you see me coming from?", because a machine behind NAT cannot discover its own public mapping by looking at itself.

Laptop, on the home network:

```
* UDP: true
* IPv4: yes, <HOME_IP>:61113
* IPv6: no, but OS has support
* MappingVariesByDestIP: false
* PortMapping: UPnP
* Nearest DERP: Toronto
* DERP latency:
    - tor: 15.9ms  (Toronto)
    - ord: 26.8ms  (Chicago)
    - nyc: 30.1ms  (New York City)
    - fra: 96.1ms  (Frankfurt)
    - nue: 99.9ms  (Nuremberg)
```

VPS, in Nuremberg:

```
* UDP: true
* IPv4: yes, <VPS_IP>:46317
* IPv6: yes
* MappingVariesByDestIP: false
* PortMapping:
* Nearest DERP: Frankfurt
* DERP latency:
    - fra: 12.2ms  (Frankfurt)
    - nue: 20.6ms  (Nuremberg)
    - nyc: 91.7ms  (New York City)
    - tor: 99.8ms  (Toronto)
```

Three things stand out.

**The two DERP maps are mirror images.** Each node's nearest relay is whichever one is geographically closest to it. If these two ever fall back to a relay, they have to agree on a single region, so one of them will be far from it. That asymmetry is worth remembering - it explains why a relayed connection can feel fine for one person and slow for another on the same tailnet.

**`MappingVariesByDestIP: false` on both.** This is the line that decides whether a direct connection is even possible. False means the NAT keeps the same public port mapping no matter which destination the traffic is going to. That predictability is what lets two peers tell each other where to send packets. If it were true - symmetric NAT - the router would allocate a different port per destination, the address discovered through STUN would be useless to a peer, and a direct connection would be impossible rather than just slow.

**`PortMapping: UPnP` on the laptop but not the VPS.** My home router supports UPnP, which gives Tailscale a way to request an explicit port forward rather than relying on hole punching alone. The VPS has no NAT in front of it at all, so there is nothing to map.

## Test 1 - Laptop to VPS

Prediction: direct. Both ends reported working UDP and non-varying NAT mapping, and the VPS has a public IP.

```
> tailscale ping ts-vps-01
pong from ts-vps-01 (100.94.159.116) via <VPS_IP>:1026 in 97ms
```

Direct on the first attempt - the response came back via the VPS's real public address, with no DERP in the path. 97ms is ordinary transatlantic latency for Toronto to Nuremberg.

`tailscale ping` stops as soon as it gets a reply, unlike regular ping. To watch the full behaviour, `-c 10` forces ten attempts.

## Test 2 - Laptop to phone on cellular

Prediction: DERP. Carrier CGNAT is often symmetric, and I have seen the equivalent problem before - a user on cellular unable to bring up an IPSec VPN to a SonicWall, because the carrier's NAT would not hold a usable mapping.

```
> tailscale ping -c 10 els26ultra
pong from els26ultra (100.87.100.18) via DERP(tor) in 28ms
pong from els26ultra (100.87.100.18) via DERP(tor) in 54ms
pong from els26ultra (100.87.100.18) via DERP(tor) in 28ms
pong from els26ultra (100.87.100.18) via 208.98.222.74:40345 in 35ms
```

This is the most useful result of the session, because it caught the upgrade happening live.

The first three replies came back through the Toronto relay. The fourth came back directly, via the carrier's public address, and the test stopped there because it had succeeded.

Tailscale does not wait for NAT traversal to finish before giving you a connection. It relays immediately so traffic flows right away, and works on hole punching in the background. When that succeeds, it moves the connection to the direct path with no interruption. Here it took about three seconds.

So my prediction was half right. The relay did come first - but the carrier's NAT turned out to be better behaved than expected, and hole punching eventually worked.

The latency is the part worth noticing: **DERP was 28ms and direct was 35ms.** The relay was faster. Toronto DERP sits close to both endpoints, while the direct path takes whatever route the carrier gives it. Direct is not automatically better.

## Test 3 - Laptop to phone on home Wi-Fi

Same two devices, different network.

```
> tailscale ping -c 10 els26ultra
pong from els26ultra (100.87.100.18) via DERP(tor) in 54ms
pong from els26ultra (100.87.100.18) via 192.168.88.10:56675 in 8ms
```

Direct again, but notice the address: `192.168.88.10` is a private address on my own LAN. Both devices are on the same network, so Tailscale connected them over the local link. No NAT traversal, no public internet, no relay.

It still relayed the first packet before switching. Tailscale does not assume a LAN path exists - it discovers it, then upgrades.

## Results

| Path | How it connected | Latency |
|---|---|---|
| Laptop â†’ VPS (Nuremberg) | Direct, public IP | 97ms |
| Laptop â†’ phone on cellular | DERP (Toronto) | 28ms |
| Laptop â†’ phone on cellular | Direct, carrier NAT | 35ms |
| Laptop â†’ phone on Wi-Fi | Direct, LAN | 8ms |

## What I took away

**Tailscale prioritises working immediately over working optimally.** Relay first so there is connectivity, optimise to direct in the background, upgrade transparently. For support work this reframes a common question - "my connection is using DERP" may just mean it has not upgraded *yet*. The real question is whether it is *stuck* on the relay, and `netcheck` on both ends is how you tell.

**DERP is not automatically the slow option.** A nearby relay beat a direct path across a carrier network in this test.

**The 100.x address is a stable identity, not a route.** The phone moved from cellular to Wi-Fi and kept the same Tailscale address and the same session while the underlying path changed completely. Nothing in a traditional client VPN behaves this way - a SonicWall SSL VPN client that changes networks drops and reconnects.

**`MappingVariesByDestIP` is the first thing to check** when a connection will not go direct. It is the difference between "hole punching is hard here" and "hole punching is impossible here."

