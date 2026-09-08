# Blocking LG smart-TV telemetry at the network level

Notes from measuring what my own LG webOS TV actually sends home — and how I stopped
it — after reading yet another round of headlines about smart TVs uploading audio and
scanning home networks.

**Every privacy setting in the TV was already switched off.** Live Plus (ACR) off, ad
tracking limited, user agreements for voice and personalised advertising withdrawn. It
made no measurable difference to the traffic. That is the whole point of this write-up:
if you want a smart TV to stop talking, the TV's own settings are not where you do it.

---

## What I measured

Method: DNS query logs from a Pi-hole that all LAN clients use, plus the router's
connection-tracking table. No packet capture, no TLS interception.

**3,351 DNS queries from one TV in a single day.**

### 1. It enumerates your LAN

The TV issued reverse-DNS lookups (`<n>.<x>.<y>.<z>.in-addr.arpa`) for **30 distinct
internal addresses**, alongside mDNS discovery. It is walking your local address range
and trying to resolve a name for every host it finds.

It did **not** open connections to those hosts. This is inventory, not intrusion — but
it is inventory of your home, built without asking.

### 2. It talks to telemetry and ad endpoints, constantly

| Domain | Queries/day | What it is |
|---|---|---|
| `eic.api.lgtviot.com` | 398 | LG IoT API |
| `eic.lgtviot.com` | 52 | same family |
| `nl.nextlgsdp.com` | 50 | LG Smart Data Platform |
| `nl.rdx2.nextlgsdp.com` | 14 | same family |
| `nl.lgeapi.com` | 22 | LG Electronics API |
| `eic.tv.wiselg.com` | 14 | LG analytics |
| `nl.info.lgsmartad.com` | 8 | **LG Smart Ad** |
| `www.ueiwsp.com` | 64 | Universal Electronics (remote-control database) |

Replace `nl.` with your own country prefix.

Also noisy, and worth knowing about:

- `discovery.meethue.com` — the TV polling for a Philips Hue bridge. I do not own one.
- `nrdp.logs.netflix.com` / `nrdp.push.prod.netflix.com` — the Netflix app keeps a push
  channel open and ships logs **even while you are watching something else entirely**.

### 3. It tries to route around your DNS filter

The TV sends DNS directly to hardcoded public resolvers (`1.1.1.1`, `8.8.8.8`) rather
than the resolver handed out by DHCP. This matters more than the blocklist itself:

```
dig eic.api.lgtviot.com @<your-pihole>   ->  0.0.0.0        (blocked)
dig eic.api.lgtviot.com @1.1.1.1         ->  18.239.50.37   (real address)
```

A DNS blocklist alone is therefore **not** a control. Any device that ships its own
resolver address walks straight past it.

---

## What I could not determine

**Whether audio is being uploaded.** All of this traffic is TLS. DNS logs and
connection tracking show *who* a device talks to and *how often* — never *what* is in
the payload. Anyone claiming to have proven audio exfiltration from network metadata
alone is overstating their evidence, and I am not going to do that here.

What is demonstrable is the volume, the destinations, the LAN enumeration, and the
deliberate bypass of local DNS. That was enough for me to act.

---

## The fix

Three layers. The second one is the one people skip, and it is the one that matters.

### Layer 1 — Block the telemetry domains

Wildcards, so tomorrow's subdomain is covered too:

```bash
for d in lgtviot.com nextlgsdp.com lgsmartad.com wiselg.com ueiwsp.com; do
  pihole deny --wild "$d"
done
```

Optional extras, blocked as **exact** names so you do not break anything else:

```bash
pihole deny discovery.meethue.com     # only if you have no Hue bridge
pihole deny nrdp.logs.netflix.com     # Netflix logging; playback unaffected
```

**Do not blanket-block `lgeapi.com`.** It appears to carry the app store and firmware
updates. Blocking telemetry is good; blocking your own security updates is not.

Likewise, do not wildcard `netflix.com`. `nrdp.logs` is telemetry;
`nrdp.push.prod.netflix.com` is the push channel that makes the app work.

### Layer 2 — Force the TV through your resolver

Without this, layer 1 is decoration. Allow the TV to reach *only* your own resolver on
port 53, and drop DNS-over-TLS:

```bash
TV=192.168.x.x        # the TV
DNS=192.168.x.x       # your Pi-hole / resolver

iptables -t raw -I PREROUTING 1 -s $TV ! -d $DNS -p udp --dport 53 -j DROP
iptables -t raw -I PREROUTING 1 -s $TV ! -d $DNS -p tcp --dport 53 -j DROP
iptables -I FORWARD 1 -s $TV -p udp --dport 853 -j DROP
iptables -I FORWARD 1 -s $TV -p tcp --dport 853 -j DROP
```

**Note the `raw` table — this is the part that cost me an evening.** My first attempt put
the DNS rules in `FORWARD`, verified they were loaded, and moved on. Days later the router's
own traffic log still showed the TV talking to `8.8.8.8`, while my rules sat at **zero
packets matched**.

The reason: the router had the TV enrolled in its own DNS-interception feature. Client DNS
was being DNAT'd in `nat/PREROUTING` to a resolver running on the router itself — so the
packets were consumed *before* they ever reached `FORWARD`, and my rules were downstream of
the thing they were supposed to catch. Worse, that built-in resolver **resolves independently
of the blocklist**: querying it directly returned live addresses for every domain I thought
I had blocked.

```
# on the router, against its own interception resolver:
dig eic.api.lgtviot.com @127.0.0.1 -p 1053
  -> 18.239.50.37   # not blocked at all
```

So the TV had a fully working escape hatch that *looked* closed from every angle I had
checked: the blocklist was correct, the firewall rules were present, the connection table
showed no traffic to the telemetry addresses at that moment. Only the packet counters
staying at zero gave it away.

The `raw` table runs before connection tracking and before NAT, so a rule there fires ahead
of the interception. After the change: **51 packets dropped in the first 45 seconds** — the
TV had been leaning on that path constantly — and its queries reappeared in the resolver's
log, correctly denied.

**On a consumer router/firewall, express this as a native policy instead** — raw `iptables`
usually does not survive a reboot or a config re-provision. Scope the policy as *"from the
TV, to the **internet zone**, port 53 and 853 → block"*. Do not block a specific public
resolver address: this TV used two, and blocking one just moves the traffic to the other.
Targeting the zone leaves the LAN path to your own resolver open and needs no maintenance
when the next hardcoded address shows up.

And whichever way you do it — **verify with counters, not with the presence of the rule.**
A loaded rule that never matches looks identical to a working one.

### Layer 3 — Optional: keep it off your LAN

Putting the TV on an isolated VLAN with internet access but no path to the rest of the
LAN stops the enumeration entirely. It also breaks casting and local device
integrations, so it is a genuine trade-off rather than a free win.

---

## Verifying it works

Blocklist is biting — you want to see `denied` lines for the TV's address:

```bash
grep -E "lgtviot|nextlgsdp|lgsmartad|wiselg|ueiwsp" /var/log/pihole/pihole.log | tail
```

```
query[A] www.ueiwsp.com from 192.168.x.x
regex denied www.ueiwsp.com is 0.0.0.0
```

**Check the rule counters** — a rule at zero packets is either doing nothing or sitting
downstream of something that already handled the traffic:

```bash
iptables -t raw -L PREROUTING -n -v
iptables -L FORWARD -n -v
```

Traffic actually stopped (not just DNS) — resolve one of the endpoints and check the
router's connection table for that address. Zero sessions is the goal.

Confirm you did not break the things you use:

```bash
dig www.netflix.com @<your-pihole>              # must still resolve
dig nrdp.push.prod.netflix.com @<your-pihole>   # must still resolve
```

---

## Caveats, honestly

- **DNS-over-HTTPS is not covered.** DoH rides on port 443 and is indistinguishable
  from normal web traffic without deeper inspection. If a device moves to DoH, the
  tell is traffic continuing to the endpoint's IP addresses while your DNS log falls
  silent. The answer then is blocking by address, not by name.
- **Raw firewall rules on appliance routers are usually not persistent.** Convert them
  to a native rule in the vendor's own configuration.
- **Your router may be intercepting client DNS itself**, which both hides the traffic from
  rules placed too late in the chain and answers queries from a resolver your blocklist does
  not control. Check for a DNAT of port 53 to a local address before trusting any of this.
- **The blocked device keeps trying.** That is expected and harmless; it gets
  `0.0.0.0` and gets nowhere.
- **Endpoints change.** Country prefixes differ, and vendors add hostnames. Re-check
  your DNS logs occasionally rather than assuming a one-time fix holds forever.

---

## The takeaway

The interesting finding was not that a smart TV phones home — everyone assumes that.
It was that **the privacy switches in the TV did not change the behaviour**, and that
the device **carries its own DNS resolver specifically so a network-level blocklist
does not apply to it**. Those two facts together are why this has to be handled on the
network, and why blocking domains without also forcing DNS through your own resolver
gives you a false sense of having fixed it.

The third fact is the one I did not expect: **my own router was quietly undermining the
fix**, by intercepting client DNS into a resolver that ignored the blocklist. Every
individual piece of the setup looked correct. The only thing that exposed it was a packet
counter that should not have been zero.

---

*Measured on an LG webOS TV in September 2026. Numbers are from one device on one
network; your endpoints will differ by region and firmware.*
