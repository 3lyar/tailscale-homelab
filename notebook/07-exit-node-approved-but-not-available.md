# Exit node approved, but not available to my laptop



## What I was building



Using the VPS in Nuremberg as an exit node, so all of my laptop's internet traffic leaves from Germany.



Normally Tailscale only carries traffic meant for the tailnet, and everything else goes out the normal internet connection. An exit node changes that: all internet traffic goes through one chosen node and leaves from there. It is the same difference as split tunnel versus full tunnel on a traditional VPN client.



## Environment



- `ts-vps-01` - Ubuntu 26.04 LTS on a Hetzner VPS in Nuremberg, tagged `tag:server`

- `teham` - Windows 11 laptop, on the office network

- Tailscale 1.102.x



The policy at the start was already restrictive, from an earlier session:



```

{"src": ["autogroup:member"], "dst": ["autogroup:self"], "ip": ["*"]},

{"src": ["autogroup:member"], "dst": ["tag:server"], "ip": ["tcp:22"]},

```



My own devices can reach each other on anything, and anything tagged as a server only on port 22.



## Baseline



```

PS> curl.exe ipinfo.io

{

&#x20; "ip": "<OFFICE_IP>",

&#x20; "region": "Ontario",

&#x20; "country": "CA",

&#x20; "org": "AS577 Bell Canada"

}

```



## The node



IP forwarding first, because Linux drops packets that are not addressed to itself unless told otherwise, and an exit node is a router for internet traffic:



```

tee /etc/sysctl.d/99-tailscale.conf <<EOF

net.ipv4.ip_forward = 1

net.ipv6.conf.all.forwarding = 1

EOF



sysctl -p /etc/sysctl.d/99-tailscale.conf

```



Then advertise it:



```

tailscale set --advertise-exit-node

```



No health warning this time. On the subnet router I advertised first and got a warning about forwarding. Doing it in this order avoided it.



## The console



The Machines page showed an orange **Exit Node** badge with an info icon, meaning advertised but waiting for approval:



![Exit node advertised, waiting for approval](../screenshots/exitnode-machines-list-badge.png)



Approved under Edit route settings:



![Approving the exit node](../screenshots/exitnode-approval-dialog.png)



The badge turned blue.



One thing worth noting: approving it changed nothing on any device. My laptop and phone kept using their own internet. Approval only makes an exit node available. Each device still has to choose it.



## The problem



On the laptop, the Tailscale menu under Exit nodes showed:



![No exit node available](../screenshots/exitnode-not-available.png)



The VPS was configured. The console had approved it. There was no error anywhere. The option simply did not exist on the laptop.



## Cause



The policy. Grants do not only decide what traffic can pass. They also decide what each device is told about in the first place.



My two grants covered my own devices and SSH to servers. Neither one allowed internet access through an exit node. So the control plane never offered the VPS to my laptop as an exit node at all.



Under the default policy, which allows everything, this condition is quietly met, so it is easy to never notice it exists. Restricting the policy is what made it visible.



## Resolution



One more grant:



```

{"src": ["autogroup:member"], "dst": ["autogroup:internet"], "ip": ["*"]},

```



`autogroup:internet` means the internet, reached through an exit node.



After saving, the VPS appeared in the laptop's exit node list:



![Exit node available after adding the grant](../screenshots/exitnode-available-after-grant.png)



With it selected:



```

PS> curl.exe ipinfo.io

{

&#x20; "ip": "<VPS_IP>",

&#x20; "city": "Nuremberg",

&#x20; "region": "Bavaria",

&#x20; "country": "DE",

&#x20; "org": "AS24940 Hetzner Online GmbH"

}

```



And from the command line:



```

100.94.159.116  ts-vps-01  tagged-devices  linux  active; exit node; direct <VPS_IP>:41641, tx 5612352 rx 23004320

```



`exit node` in the status column confirms which exit node is in use. `direct` means the connection goes straight to the VPS without a relay.



Setting the exit node back to None returned the laptop to the office connection, and the status line changed:



```

100.94.159.116  ts-vps-01  tagged-devices  linux  active; offers exit node; direct <VPS_IP>:41641

```



`exit node` means this device is using it. `offers exit node` means it is available but not in use. The traffic counters also stopped growing, which confirms nothing was flowing through it anymore. When a customer pastes their `tailscale status`, that one word tells you whether they are actually on the exit node or only have one available.



## Why websites saw the VPS address



The VPS rewrites the laptop's traffic so it appears to come from the VPS itself. This is source NAT, the same thing an office firewall does for every device on the way out to the internet. Here the office is one laptop and the firewall is a server in Germany.



## Why the cloud firewall needed no change



The Hetzner firewall has no inbound rules for this. Exit traffic leaves the VPS outbound, and the replies come back because they belong to connections the VPS started. A stateful firewall lets that return traffic through without any rule.



## Three conditions, and only one of them speaks



| Condition | Where it lives | What you see when it is missing |

|---|---|---|

| IP forwarding, advertised | On the node | A health warning in `tailscale status` |

| Approval | Admin console | An orange badge, only if you look |

| Policy grant | Policy file | Nothing. The option does not appear |



The first one tells you what is wrong. The second one is visible only in the console. The third one is completely silent.



## What I took away



- "I approved my exit node but it does not show up on my laptop" is a policy question first. Check for access to `autogroup:internet`.

- Approving an exit node does not route anyone through it. Each device has to select it.

- A restrictive policy exposes conditions the default policy hides. Subnet routes have the same hidden condition: the policy has to allow the subnet.

- `tailscale status` shows whether a device is using an exit node or only has one available, which is the first thing to confirm when someone says their traffic is not going where they expected.


