# Tailscale on Windows kept stopping right after starting



## Symptom



After the laptop woke up from a few days asleep, Tailscale would not stay connected. `tailscale up` returned without an error, but `tailscale status` kept showing the same thing:



```

PS> tailscale down

PS> tailscale up

PS> tailscale status

# Health check:

#     - Tailscale is starting. Please wait.



unexpected state: NoState

```



Running it again and again gave the same result. The Windows service itself looked healthy:



```

PS> Get-Service Tailscale



Status   Name       DisplayName

------   ----       -----------

Running  Tailscale  Tailscale

```



So the service was up, but the connection never was.



## Environment



- `teham` - Windows 11 laptop, Tailscale 1.102.2

- The laptop had been asleep for about three days

- No other nodes affected



## Investigation



`tailscale status` only showed the symptom, so I went to the daemon logs:



```

PS> tailscale debug daemon-logs

```



The same pattern repeated all through the log, a few milliseconds apart:



```

client connected (TEHAMElyar Abedini): switching to profile (fbd4)

...

client disconnected (TEHAMElyar Abedini): disconnecting Tailscale

```



At one point it actually reached a working connection, then dropped it 10 milliseconds later:



```

01:09:15.122 Switching ipn state Starting -> Running (WantRunning=true, nm=true)

01:09:15.132 client disconnected (TEHAMElyar Abedini): disconnecting Tailscale

```



The log also showed Tailscale disconnecting when I signed in and when I locked the screen:



```

TEHAMElyar Abedini signed in to session 1: disconnecting Tailscale

TEHAMElyar Abedini locked session 1: disconnecting Tailscale

```



The words "client disconnected ... disconnecting Tailscale" were the clue. Tailscale was not failing. It was shutting down on purpose, every time something disconnected from it.



## Cause



On Windows, the Tailscale service only keeps the connection up while a program on the desktop is connected to it. Normally that program is the Tailscale tray app, the icon near the clock, and it stays connected all day.



After the laptop woke up, the tray app was not running. So every `tailscale` command I typed connected to the service, started bringing Tailscale up, then finished and exited. The service saw its only client leave and shut the connection down again. That is why `tailscale up` looked like it worked while `status` never changed.



## The part I did not expect



The very end of the log looked different. Tailscale reached `Running` and stayed there, with network checks every 20 seconds and no disconnect.



That was the moment I ran `tailscale debug daemon-logs`. That command keeps a connection open to stream the logs, so while it ran, it was the connected client, and Tailscale stayed up. The diagnostic tool fixed the problem for as long as it was running.



## Resolution



I started the Tailscale app from the Start menu. Once the tray icon was back, the connection came up and stayed up:



```

PS> tailscale status

100.119.171.84   teham               elyar.ab@  windows  -

100.94.159.116   ts-vps-01           elyar.ab@  linux    -

100.64.15.117    ubuntuservervmvbox  elyar.ab@  linux    -

```



Offline nodes omitted.



## Longer-term fix: unattended mode



The tray app has a setting called **Run unattended** (tray icon, Preferences). With it on, Tailscale keeps running even when the tray app is closed, the screen is locked, or nobody is signed in.



This is the same idea as "Start Before Logon" or always-on mode in VPN clients like Cisco AnyConnect or SonicWall. Without it, the VPN only exists while a user is signed in.



It matters for this laptop because I connect to it remotely. Without unattended mode, locking the screen takes it off the tailnet and there is no way to reach it.



The trade-off is that the laptop stays reachable, and exposed, even when I am not using it. For a machine I need to reach remotely, being reachable is the point.



## Side note from the same log



Right after waking up, Tailscale tried to request a port mapping from my home router:



```

portmapper: createOrGetMapping: write udp4 0.0.0.0:49665->192.168.88.1:5351: wsasendto: A socket operation was attempted to an unreachable host.

```



`192.168.88.1` is the router at home, and port 5351 is NAT-PMP, one of the ways Tailscale asks a router to open a port. The laptop went to sleep at home and woke up at the office, and its first move used the old network's router. A few seconds later it detected the new network and moved on. Harmless, but a good example of a device carrying state from the last network it was on.



## What I took away



- `tailscale status` shows the symptom. The daemon logs show the cause.

- "Disconnecting" in a log does not always mean failing. Here it was the system doing exactly what it was built to do.

- On Windows, a running service is not the same as a running connection. The tray app matters.

- When a customer says "tailscale up does nothing on Windows", the first question is: is the tray icon there?


