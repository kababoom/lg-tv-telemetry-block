# Blocking LG smart-TV telemetry at the network level

Notes from measuring what my own LG webOS TV actually sends home — and how I stopped it.

The trigger was a Dutch report that LG sets appear to listen in and scan home networks:
[LG smart-tv's lijken gebruikers af te luisteren en netwerken te scannen](https://tweakers.net/nieuws/251832/lg-smart-tvs-lijken-gebruikers-af-te-luisteren-en-netwerken-te-scannen.html)
(Tweakers, in Dutch — itself a summary of earlier reporting). I have not verified that
research and this write-up does not repeat its central claim: **nothing here demonstrates
audio being uploaded**, and I explain below why network measurement cannot show that either
way. What I could check was the rest of it — and the network scanning turned out to be real
and easy to observe.

## TL;DR

- One LG webOS TV made **3,351 DNS queries in a day**, ran **2,169 reverse-DNS lookups**
  enumerating the local network, and talked to LG telemetry, ad, and **ACR** endpoints —
  with every privacy setting in the TV switched off. Those settings changed nothing.
- **This does not prove audio is being uploaded.** All of it is TLS. Network measurement
  shows *who* a device talks to, never *what* it sends. Anyone claiming otherwise from
  metadata is overstating it.
- **A DNS blocklist alone is not a control.** The TV reached public resolvers directly. You
  have to force its DNS to your resolver at the firewall.
- **Block exact hostnames, not parent domains.** A wildcard on `nextlgsdp.com` broke app
  updates; one on `wiselg.com` silently blocked a firmware CDN. LG mixes telemetry and
  software delivery under the same domains.
- **Check what your own DHCP hands out.** Half my "the TV is bypassing me" story was the
  network offering a public resolver as secondary DNS.
- **Turn off your router's built-in DNS filtering** if you run your own resolver. On UniFi
  (*CyberSecure → Content Filtering*) it hijacks the whole subnet's port-53 traffic to a
  resolver that ignores your blocklist — and it had rewritten 703,000 packets here.
- **Verify with packet counters, not with quiet moments.** A rule sitting downstream of that
  hijack matches nothing and looks exactly like a rule that works. I got fooled by this
  twice in one evening; both times are documented below.

---

## Quick fix

Order matters: **step 2 first**, or the rest is decoration.

### 1. Pi-hole — block the telemetry hosts

```bash
pihole deny \
  eic.api.lgtviot.com eic.lgtviot.com eic.tv.wiselg.com \
  nl.info.lgsmartad.com www.ueiwsp.com \
  prov-lg.alphonso.tv nl.elastic.lgwebostv.com \
  eic.nudge.lgtvcommon.com nl.lgrecommends.lgappstv.com

pihole deny --wild logs.netflix.com    # Netflix logging; numbered hosts, so wildcard
pihole deny discovery.meethue.com      # only if you own no Hue bridge
```

⚠️ `nl.` and `eic.` are **region prefixes** — yours will differ. Read your resolver's query
log for the TV rather than copying this list blindly.

⚠️ **Do not** block `nextlgsdp.com`, `lgeapi.com`, anything `ngfts.*`, or
`nrdp.push.prod.netflix.com`. That is the app store, the firmware CDN, and the Netflix push
channel. Blocking them breaks updates — quietly, and you find out months later.

### 2. UniFi — stop the gateway hijacking DNS

*Settings → CyberSecure → Content Filtering* → **delete the rule** (mine was "Default /
Ad Block / Always").

While this exists, the gateway rewrites the whole subnet's port-53 traffic to its own
resolver, which ignores your blocklist **and** makes every firewall policy below unmatchable.
Pi-hole already does ad blocking, better.

### 3. UniFi — check what DHCP hands out

*Settings → Networks → \<your LAN\> → DHCP Name Server*. If there is a public resolver in
there as secondary, every client may use it whenever it likes, and your filtering is optional.
Point it at your own resolver only — or at a second *filtering* one, never a public one.

### 4. UniFi — force the TV through your resolver

*Settings → Policy Engine → Create Policy*:

| Field | Value |
|---|---|
| Source Zone | Internal → **Device** → your TV |
| Action | **Block** |
| Destination Zone | External → **App** → `DNS`, `DNS over HTTPS`, `DNS over TLS` |

Scope the source to the **device**. Applied to everything, this also cuts your own resolver's
upstream lookups and takes the network's DNS down with it.

### 5. Verify it actually fires

```bash
iptables-save -c | grep UBIOS_policy_src_clients     # leading [packets:bytes]
```

A non-zero counter is proof. An empty firewall log is not a failure — once the device settles
on the resolver that answers, there is nothing left to block.

---

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
| `prov-lg.alphonso.tv` | 4 | **Alphonso — an ACR vendor** |
| `nl.elastic.lgwebostv.com` | 4 | log shipping |
| `eic.nudge.lgtvcommon.com` | 4 | promotional "nudges" |
| `nl.lgrecommends.lgappstv.com` | 4 | recommendation service |

Replace `nl.` with your own country prefix.

The low-volume entries matter more than their counts suggest. **Alphonso is an
automatic-content-recognition company** — the business of identifying what is on the screen.
Four queries a day is not a lot of traffic, but it is not a service you want reachable, and
it is the one I nearly missed by sorting the list by volume.

Also noisy, and worth knowing about:

- `discovery.meethue.com` — the TV polling for a Philips Hue bridge. I do not own one.
- `nrdp.logs.netflix.com` / `nrdp.push.prod.netflix.com` — the Netflix app keeps a push
  channel open and ships logs **even while you are watching something else entirely**.

### 3. It reaches public resolvers instead of the filtering one

The TV sent DNS straight to `1.1.1.1` and `8.8.8.8` rather than to the filtering resolver.
This matters more than the blocklist itself:

```
dig eic.api.lgtviot.com @<your-pihole>   ->  0.0.0.0        (blocked)
dig eic.api.lgtviot.com @1.1.1.1         ->  18.239.50.37   (real address)
```

A DNS blocklist alone is therefore **not** a control. Any device that reaches another
resolver walks straight past it.

**Check your own DHCP before blaming the device.** I spent a while treating both addresses
as hardcoded firmware behaviour. Then I read back my own DHCP configuration: the network
was handing out the filtering resolver as primary and **`1.1.1.1` as secondary**. One of the
two "bypasses" was not the TV going around me at all — it was the network formally offering
it a way out, and the TV taking it. Only the second address was genuinely the device's own.

A secondary DNS server pointing anywhere but your filter defeats the filter, because clients
are free to use it whenever they like. If you run a filtering resolver, it should be the
only one your DHCP advertises. The trade-off is real and worth stating: no fallback means no
DNS at all while that resolver is down.

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

### Layer 1 — Block the telemetry hosts

**Block exact hostnames, not parent domains.** This is the correction I most want to pass on,
because I got it wrong twice in one evening and both mistakes broke something.

LG mixes telemetry and essential infrastructure under the same parent domains. A wildcard on
`nextlgsdp.com` looks safe — "Smart Data Platform" reads like pure analytics — but the app
store leans on those endpoints, and blocking them **broke app updates**. A wildcard on
`wiselg.com` quietly took out `eic-ngfts.tv.wiselg.com`, which is a firmware and app download
CDN sitting on the same domain as an analytics host. Neither failure was visible until
something stopped working, and the second would only have surfaced at the next update.

So, exact hosts:

```bash
pihole deny \
  eic.api.lgtviot.com \
  eic.lgtviot.com \
  eic.tv.wiselg.com \
  nl.info.lgsmartad.com \
  www.ueiwsp.com \
  prov-lg.alphonso.tv \
  nl.elastic.lgwebostv.com \
  eic.nudge.lgtvcommon.com \
  nl.lgrecommends.lgappstv.com
```

Optional extras:

```bash
pihole deny discovery.meethue.com     # only if you have no Hue bridge
pihole deny nrdp.logs.netflix.com     # Netflix logging; playback unaffected
```

**Leave these alone** — they are the delivery path for the software on the device:

- `nextlgsdp.com` — the Content Store depends on it. Blocking it breaks app updates.
- `ngfts.*` on any parent (`ngfts.lge.com`, `eic-ngfts.tv.wiselg.com`,
  `ngfts.nextlgsdp.com`) — firmware and app download CDN.
- `lgeapi.com` — app store and update API.
- `nrdp.push.prod.netflix.com` — the push channel the Netflix app needs.

Blocking telemetry is good; blocking your own security updates is not. The way to find out
which is which is to read your resolver's query log for that device and look up each host,
rather than reaching for the parent domain.

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
TV, to the **internet zone**, DNS/DoT/DoH → block"*. Do not block a specific public resolver
address: this TV used two, and blocking one just moves the traffic to the other. Targeting
the zone leaves the LAN path to your own resolver open and needs no maintenance when the
next address shows up.

**But a zone-based policy cannot see traffic that has already been redirected.** That was
the sting in the tail here. The interception was switched on by an *ad-blocking / content
filtering* option on the network — a feature that works by capturing client DNS. With it
enabled, the destination is rewritten to a local address before the policy is evaluated, so
a rule saying "destination zone: external" never matches, and the firewall log cheerfully
reports **Allow** for a connection to a public resolver. The log is not lying; the policy
genuinely did not match.

So if you run your own filtering resolver, turn the router's built-in DNS filtering **off**.
It duplicates what your resolver already does, and while it is on it quietly outranks it.

#### On UniFi specifically

This was a UniFi OS gateway (Cloud Gateway Max, Network 10.6.101), and the details are worth
spelling out because the setting is not where you would look for it.

**Where it lives:** *Settings → CyberSecure → Content Filtering*. Not under Security, not
under the network itself — and the rule that caused all this was a single row named
"Default", scope **Ad Block**, schedule Always. It had been enabled for over a year.

**What it does under the hood**, visible over SSH to the gateway:

```bash
ipset list dnsfilter          # → the entire LAN subnet, e.g. 192.168.x.0/24
iptables -t nat -L DNSFILTER -n -v
#  DNAT  udp  match-set dnsfilter src  udp dpt:53  to:127.0.0.1:1053
```

Every client in that subnet has its outbound port-53 traffic rewritten to a resolver on the
gateway. Ask that resolver directly and you can see it ignores your blocklist entirely:

```bash
dig <a-domain-you-blocked> @127.0.0.1 -p 1053    # → a real address
```

In my case that one rule had rewritten **703,000 DNS packets**. Deleting it removes the ipset
and the DNAT chain outright — verify with the two commands above, they should come back empty.

**Verifying a UniFi policy actually matches.** Policies compile down to iptables rules that
match on ipsets, so the destination in the UI never appears literally in `iptables-save`.
Find yours by the client set and read its counters:

```bash
iptables-save -c | grep UBIOS_policy_src_clients
#  [7:203] -A UBIOS_LAN_WAN_USER -m set --match-set UBIOS_policy_src_clients_15 src \
#          -m dpi32 --cat-app 9,61 --cat-app 20,199 --cat-app 20,197 -j DROP
```

The leading `[packets:bytes]` is the honest answer to "is this rule doing anything". Note the
`dpi32` match: a policy scoped to *App → DNS / DNS over HTTPS / DNS over TLS* is a deep-packet
match on application signatures, which is how it reaches DoH on 443 — and also why it depends
on the traffic being classified first.

One more UniFi-specific trap: the **Insights → Flows** list shows recent history, not live
state. A destination you blocked ten minutes ago still sits there marked *Allow* from before
the change. Check the gateway's connection table for live sessions instead.

Scoping the source matters as much as the destination: a policy like this applied to *every*
client would also cut off your own resolver's upstream lookups, and take the whole network's
DNS down with it.

And whichever way you do it — **verify with counters over time, not with a quiet moment.**
I made exactly this mistake while writing this up: I removed my working rules to test the
policy, watched for 75 seconds, saw no connections, and concluded the policy had taken over.
It had not. I had flushed the connection table just before looking, and the device was in
its retry backoff. Sampling again over two minutes showed the bypass alive and growing —
73, then 80, then 131 packets. A rule at zero and a device that happens to be quiet look
identical for exactly as long as you are willing to be fooled.

**How it ended.** Once the router's DNS interception was switched off, the policy did work.
The proof was not another quiet window — it was finding the rule the policy compiles down to
and reading its packet counter:

```
-A ..._LAN_WAN_USER -m set --match-set ..._policy_src_clients_15 src \
   -m dpi32 --cat-app 9,61 --cat-app 20,199 --cat-app 20,197 -j DROP
                                    ^ 7 packets dropped
```

Seven packets, then nothing — because a device that finds one resolver dead settles on the
one that answers. That is also why the firewall log stays almost empty afterwards, which
looks like the rule is doing nothing. It is not: it is doing its job so well there is
nothing left to log.

Note also that the rule matches on **DPI application categories**, not on port numbers. That
covers DoH on 443, which a port rule cannot — but it means the traffic has to be classified
before it can be matched, so treat it as a strong filter rather than a hard wall.

### Layer 3 — Optional: keep it off your LAN

Putting the TV on an isolated VLAN with internet access but no path to the rest of the
LAN stops the enumeration entirely. It also breaks casting and local device
integrations, so it is a genuine trade-off rather than a free win.

---

## Verifying it works

Blocklist is biting — you want to see `denied` lines for the TV's address:

```bash
grep -E "lgtviot|lgsmartad|wiselg|ueiwsp|alphonso" /var/log/pihole/pihole.log | tail
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
router's **connection table** for that address. Zero live sessions is the goal.

Do that in the connection table rather than the firewall's flow list. A flow list shows
recent history, so a destination you blocked minutes ago still appears there, marked
*Allow*, from before the change — which reads alarmingly like the block failing. Live
sessions are the honest measure of now.

Confirm you did not break the things you use:

```bash
dig www.netflix.com @<your-pihole>              # must still resolve
dig nrdp.push.prod.netflix.com @<your-pihole>   # must still resolve
dig ngfts.lge.com @<your-pihole>                # firmware CDN
dig nl.nextlgsdp.com @<your-pihole>             # app store
```

Then actually **open the app store on the TV and install an update**. A blocklist that breaks
software delivery will pass every DNS test you can think of and still leave the device
unpatched — and you will not find out for months.

---

## Caveats, honestly

- **DNS-over-HTTPS is only partly covered.** DoH rides on port 443 and is indistinguishable
  from normal web traffic without deeper inspection. A firewall with application signatures
  can match the well-known DoH providers — worth enabling — but that is a list of known
  endpoints, not a closed door: anything not in it still passes. If a device moves to DoH, the
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

*Measured on an LG webOS TV behind a UniFi Cloud Gateway Max with a Pi-hole resolver,
September 2026. Numbers are from one device on one network; your endpoints will differ by
region and firmware.*
