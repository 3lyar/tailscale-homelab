# Why RDP lags from a Chromebook over Tailscale (open investigation)

## Symptom

Remote Desktop from a Chromebook to a Windows laptop over Tailscale ran in a repeating cycle: roughly 30 seconds usable, then about 10 seconds frozen, then usable again. SSH to the same laptop over the same tunnel was fine throughout.

No resolution yet. This entry records what was measured, what was ruled out, and what is still untested.

## Environment

- `penguin` - Chromebook, Tailscale running in the Crostini Linux container
- `strongbad` - the same Chromebook earlier, running Tailscale as the Android app
- `teham` - Windows 11 laptop at home, `192.168.88.9` on the LAN
- Tailscale 1.102.x on all nodes
- Client tested from three different networks; the target never moved

## First measurement: it is not RDP

Plain ICMP to the laptop showed the same pattern, which rules out RDP itself as the cause:

```
64 bytes from 100.119.171.84: icmp_seq=7  time=59.7 ms
64 bytes from 100.119.171.84: icmp_seq=18 time=25.2 ms
```

Sequence numbers 8 through 17 are missing. Ten consecutive packets lost, then recovery. One run finished with 50% packet loss, another with 10%. Latency swung between 9ms and 87ms on a link that should be far more stable.

ICMP carries no display data and no protocol overhead, so the loss is in the path, not in the application.

## Isolating the variable: three client networks, same target

The useful move was changing only the client's network and leaving both endpoints alone.

**Work Wi-Fi**

```
* UDP: true
* IPv4: yes, <WORK_IP>:52902
* MappingVariesByDestIP: false
* PortMapping: 
* Nearest DERP: Toronto (16.2ms)
```

`MappingVariesByDestIP: false` means the router uses the same public port regardless of destination, so the address discovered through STUN is usable by a peer. Combined with working UDP, hole punching should succeed. It did, and `tailscale status` showed a direct path:

```
100.119.171.84  teham  elyar.ab@  windows  active; direct <HOME_IP>:1036
```

But it did not hold. Later attempts returned to the relay, and `tailscale ping` gave up after ten tries:

```
pong from teham (100.119.171.84) via DERP(tor) in 24ms
... eight more DERP responses ...
direct connection not established
```

When it did re-establish, the port had changed from 1036 to 1037, which means a new NAT mapping was allocated rather than an existing one being reused. NAT table entries for UDP typically expire after tens of seconds of idle time, so an idle session loses its mapping and the next attempt has to punch a new hole.

**Phone hotspot**

```
* UDP: true
* IPv4: yes, <HOTSPOT_IP>:63368
* MappingVariesByDestIP: true
* PortMapping: 
```

`MappingVariesByDestIP: true` means the carrier allocates a different public port for every destination. The port STUN reported belonged to the STUN conversation only, so a peer sending there hits a mapping that is not theirs and the router drops it. A direct connection here is not slow, it is impossible. Every attempt relayed, at 46ms to 89ms, the worst of the three.

This was still a useful test: it proved work Wi-Fi was never the blocker.

**Home, measured from `teham` itself**

```
* UDP: true
* IPv4: yes, <HOME_IP>:61300
* MappingVariesByDestIP: false
* PortMapping: UPnP
* Nearest DERP: Toronto (6.5ms)
```

The router answered UPnP discovery and Tailscale picked it up, meaning it can request an explicit port forward rather than relying on hole punching alone. So the home side is healthy, and my earlier theory that the home NAT was the problem was wrong. Worth stating plainly, because measuring it is what killed the theory.

| Client network | Same port for every destination? | Port mapping | Result |
|---|---|---|---|
| Work Wi-Fi | yes | none | Direct, then decays |
| Phone hotspot | no | none | Relay only, permanently |
| Home (target side) | yes | UPnP | Healthy |

## The platform constraint

Tailscale issue [#432](https://github.com/tailscale/tailscale/issues/432) describes the underlying problem. On ChromeOS, both the Crostini Linux container and the Android environment have their own network layers, each NATted through ChromeOS. ChromeOS assigns those internal interfaces addresses inside `100.64.0.0/10`, the same range Tailscale uses for tailnet addresses.

Tailscale filters out `100.x` addresses when reporting local endpoints, on the assumption that they belong to Tailscale. On ChromeOS that filter also discards the container's real local address, so the control server never learns where the node actually is. The issue notes that Tailscale running in both layers on the same device can only reach itself through DERP for this reason.

That issue is now marked closed, but the workaround is still present in current Tailscale code: `ChromeOSVMRange()` carves out `100.115.92.0/23` so Tailscale never assigns addresses from the range ChromeOS uses. Closed does not appear to mean removed.

A related and still-current problem is worth noting alongside it. Issues [#12090](https://github.com/tailscale/tailscale/issues/12090) and [#19488](https://github.com/tailscale/tailscale/issues/19488) describe `cros-garcon`, the ChromeOS container agent, crashing on cold boot when the `tailscale0` interface is present, taking the whole Crostini container down with it. The workaround is running tailscaled with `--tun=userspace-networking`, which keeps Tailscale out of the kernel interface list. Google ships that component as a closed binary, so an upstream fix is on an indefinite timeline.

One correction worth making, because the evidence does not support the stronger claim: none of this forces every connection to relay. Direct paths from Crostini to an external peer did establish, repeatedly. What the overlap appears to cost is reliability, not possibility. Endpoint discovery is degraded, so hole punching is slow to succeed and quick to fail.

It is also worth being precise that issue #432 describes Android and Crostini talking to each other on the same device, which is not this case. The relevance is the shared root cause, not the scenario.

## A second, separate finding

RDP connected successfully using the Microsoft Windows App but failed from Remmina, with the same target, the same credentials, and the same network. Two different RDP implementations, one works and one does not, so this is a client-side problem and not a network one. The most likely cause is the space in the Windows username, which the Microsoft client handles and FreeRDP may not.

Worth separating this out. For most of the session it looked like one problem. It is two.

## Why SSH was unaffected

SSH over the same tunnel, to the same laptop, in the same relay conditions, worked without complaint throughout. SSH is low-bandwidth and text-based, so a few hundred milliseconds of relay jitter is invisible. RDP streams bitmap updates and input events continuously, so the same jitter shows up as a frozen screen.

A useful thing to hold onto: "the VPN is slow" and "this one application is slow over the VPN" are different reports, and testing a second protocol separates them in under a minute.

## Still untested

- **RDP over UDP.** The Windows Firewall rule "Remote Desktop - User Mode (UDP-In)" was never verified on the laptop. RDP over TCP freezes the entire screen on packet loss, while RDP over UDP degrades more gracefully. Given the loss measured here, this may matter more than anything else on the list.
- **MTU.** The `tailscale0` interface runs at 1280. If the RDP client assumes 1500, fragmentation inside the ChromeOS layers is plausible.
- **Userspace networking in Crostini.** Running `tailscaled --tun=userspace-networking` avoids the kernel interface conflicts described above. It does not bypass the relay, so it is a stability fix rather than a latency fix.
- **The Remmina username format**, which is the likeliest explanation for that client failing.

## What this was worth

The measurements ruled out more than they confirmed, which is most of the value. The home network was cleared, work Wi-Fi was cleared, the hotspot was identified as symmetric NAT with a definite and unfixable cause, RDP was separated from the transport, and one client was separated from another.

What is left is a documented platform limitation and four specific things to test. That is a reasonable place for an investigation to sit.