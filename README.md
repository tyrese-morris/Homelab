# Home Lab — Interactive Map \& Documentation

A Windows Server 2022 lab I built from scratch, documented as an interactive map. Every
piece of it explains not just what I set up, but why I set it up that way.

[**View the live map →**](https://tyrese-morris.github.io/Homelab/)

\---

## The environment

|Layer|Build|
|-|-|
|**Physical network**|Dedicated lab segment behind its own router · TP-Link TL-SG108E 8-port managed switch|
|**Virtualization**|VMware Workstation Pro — Type 2 hypervisor|
|**Domain controller**|Windows Server 2022 Standard · AD DS, DNS, DHCP, Group Policy|
|**Domain client**|Windows 11 Enterprise joined to `lab.local`|
|**Firewall**|pfSense CE on FreeBSD — WAN/LAN interfaces, stateful rules, NAT|
|**Analysis**|Wireshark — ICMP, DNS and DHCP packet captures|

**Certifications:** CompTIA A+ · CompTIA Network+ (N10-009) · Microsoft AZ-900 ·
Security+ (SY0-701) in progress

I didn't study for Network+ and then build this. Building it *was* how I studied, and
I'd configured and broken most of the exam material before I ever saw a practice question.

\---

## VLAN segmentation — 802.1Q on a managed switch

Four VLANs on real hardware: default, management, lab, and an isolated guest segment.
Port 1 is a tagged trunk carrying all VLANs to the router; ports 2–4 are untagged access
ports, one per VLAN.

!\[802.1Q VLAN configuration showing four VLANs with tagged and untagged port members](images/vlan-802.1q-table.png)

PVIDs decide which VLAN an untagged frame lands in when it enters an access port — the
setting behind most "this port can't reach the network" problems.

!\[PVID assignments per switch port](images/vlan-pvid-assignments.png)

**What this demonstrates:** Layer 2 segmentation, 802.1Q tagging, trunk vs access port
roles, PVID assignment, and the fact that VLANs cannot route between themselves without
a Layer 3 device.

\---

## pfSense firewall — stateful filtering and NAT

pfSense CE running as a VM with two interfaces: WAN facing the untrusted side, LAN
serving the lab. A firewall needs two sides or it has nothing to separate.

!\[pfSense LAN interface configured static with a DHCP range](images/pfsense-lan-config.png)

Once it was up, both interfaces came online at gigabit — WAN pulling an address from the
untrusted side, LAN acting as the gateway for everything in the lab.

!\[pfSense dashboard showing system information and both interfaces up](images/pfsense-dashboard.png)

**What this demonstrates:** stateful vs stateless inspection, top-down rule processing
where first match wins, outbound NAT, RFC 1918 addressing, and firewall log analysis.

\---

## Packet analysis — Wireshark

**The full DHCP DORA handshake** captured by releasing and renewing a lease. Discover and
Request come from 0.0.0.0 to the broadcast address — the client has no IP yet and doesn't
know where the server is. All four packets share one Transaction ID, which is what ties
the conversation together across broadcasts.

!\[Wireshark capture of DHCP Discover, Offer, Request and Acknowledge sharing a transaction ID](images/wireshark-dhcp-dora.png)

**ICMP echo request and reply.** Requests leave with TTL 128, the Windows default; replies
return at 114. Every router hop decrements it by one, so the gap is a built-in hop counter
— the same mechanism traceroute is built on.

!\[Wireshark ICMP capture showing echo requests at TTL 128 and replies at TTL 114](images/wireshark-icmp-ttl.png)

**DNS queries and responses**, with the router acting as a forwarder — and a steady stream
of background lookups from applications nobody was actively using.

!\[Wireshark DNS capture showing queries and responses through the local forwarder](images/wireshark-dns-query.png)

**What this demonstrates:** reading the OSI model in live traffic, display filters, the
DHCP lease process, TTL and hop counting, DNS resolution and forwarding.

\---

## Active Directory \& Group Policy

Windows Server 2022 promoted to domain controller: AD DS with an integrated DNS zone,
a DHCP scope authorized in the directory, organizational units, and Group Policy objects
targeted at specific OUs.

The OUs started flat at the domain root, shown below, and were later nested under a parent
OU so policy could flow through three levels instead of two. The icons are worth noticing —
you can link a GPO to an OU you created, but not to a built-in container like Users or
Computers.

!\[Active Directory Users and Computers showing the lab.local tree with four organizational units](images/ad-ou-structure.png)

Inside an OU with its test user. This is the level a Group Policy object gets linked at,
and where a real account would live.

!\[An organizational unit containing a user object](images/ad-ou-user.png)

**What this demonstrates:** domain services, the AD/DNS dependency, DHCP scope design that
keeps static infrastructure clear of the dynamic pool, OU hierarchy, and GPO scoping.

### Group Policy

Beyond making a single setting apply: how policies resolve when several touch the same
thing, and how to prove which one won.

Policies process **Local → Site → Domain → OU**, parent OU before child, and the last one
processed overwrites the rest — so the GPO closest to the object wins. Within a single
container, link order 1 has the *highest* precedence, because the list is processed in
reverse. To make that visible I built three GPOs at three levels, each setting a wallpaper
labeled with where it was linked, so the desktop itself names the winner.

Two tools do the proving. `gpresult /r` reports what applied and, just as useful, what
didn't and why — and its output has two independent halves, one for the computer and one
for the user. `gpresult /h` writes a full HTML report attributing every setting to the GPO
that supplied it. **Group Policy Modeling** in GPMC simulates the result for any
user/computer pair without applying anything, which answers "what breaks if I move this
user to that OU" before you move them.

!\[Desktop Wallpaper group policy setting configured with a UNC path](images/gpo-wallpaper-policy.png)

The policy references the image by UNC path rather than a local one, so it resolves from
any machine in the domain instead of only the server holding the folder.

!\[A domain-joined client desktop showing the wallpaper enforced by Group Policy](images/gpo-wallpaper-applied.png)

Landing on a domain client. Labeling each wallpaper with the level it was linked at means
the desktop itself reports which GPO won — the precedence result becomes readable at a
glance instead of something you dig out of a report.

**What this demonstrates:** GPO precedence and inheritance, link order, User vs Computer
configuration, UNC-based policy paths, and RSoP diagnostics.

### The domain client

!\[Windows sign-in screen showing "Sign in to: LAB"](images/domain-join-signin.png)

A Windows 11 client joined to `lab.local`. The "Sign in to: LAB" line is the confirmation
— it only appears once a machine is domain-joined. The password prompt is the account's
must-change-at-next-logon flag, the same mechanism behind every helpdesk password reset.

\---

## Troubleshooting log

Real failures, root causes, and fixes. I keep this because a build that breaks teaches
you more than one that doesn't.

**Two DHCP servers on one segment.** After standing up pfSense, the domain controller came
up on a subnet pfSense doesn't even own, with no default gateway — and the firewall GUI was
unreachable. VMware runs its own DHCP service on host-only networks, so two servers were
answering and the hypervisor won the race. This is rogue DHCP in miniature; the real-world
version is someone plugging a consumer router into an office wall jack.

!\[Domain controller holding an address from the wrong subnet with no gateway, and the pfSense GUI failing to load](images/dhcp-conflict-wrong-subnet.png)

**What a DHCP failure looks like from the client side.** A 169.254 address, a 255.255.0.0
mask, and an empty default gateway. The blank gateway is the giveaway — nothing answered
the broadcast.

!\[ipconfig output showing an APIPA address and blank default gateway](images/apipa-dhcp-failure.png)

**A new client that could ping the DC but not find the domain.** Zero packet loss to the
domain controller, and `nslookup lab.local` returned nothing. pfSense was handing out its
own address as the DNS server — but a domain join hunts for a family of service records
that Active Directory creates in its own zone when a DC is promoted, and a firewall holds
none of them.

!\[DNS Manager showing the \_msdcs, \_sites, \_tcp and \_udp service record folders in the lab.local zone](images/ad-dns-service-records.png)

The successful ping is what isolated it: connectivity was fine, so the fault had to be
name resolution. I fixed it at the source rather than on the one machine — in pfSense's
DHCP options, so every future client gets it automatically.

!\[pfSense DHCP server LAN options with the DNS server field set to the domain controller](images/pfsense-dhcp-dns-option.png)

!\[nslookup resolving lab.local against the domain controller](images/dns-resolution-fixed.png)

**A GPO that reached nobody.** Correctly built, correctly linked, and `gpresult` reported
`Applied Group Policy Objects: N/A`. The distinguished name explained it —
`CN=Administrator,CN=Users`. Users is a container, not an OU, and you can't link a GPO to
a container, so an OU-linked policy can never reach an account living there.

!\[gpresult user settings section showing CN=Administrator,CN=Users with no applied group policy objects](images/gpresult-no-user-policy.png)

**Group Policy failing while every network test passed.** Clients couldn't read SYSVOL or
reach shares, but `Test-NetConnection` to the DC on port 445 returned true and the firewall
profile was correct. The domain controller's clock had drifted three hours after repeated VM
power cycles, and Kerberos rejects authentication beyond a five-minute skew — no warning, no
grace period. A DC is the authoritative time source for its domain, so when its clock is
wrong, nothing in the domain can authenticate.

|Problem|Root cause|
|-|-|
|Workstation lost internet after VLAN config|Port moved to a VLAN the router didn't route — segmentation working exactly as designed|
|VM pulled an address from the wrong range, no gateway|Two DHCP servers on one segment; the hypervisor's own service answered first|
|New client couldn't locate the domain|DNS pointed at the firewall, which holds none of AD's service records|
|Group Policy and shares failing domain-wide|Domain controller clock drift past the Kerberos five-minute tolerance|
|Drive mapping returned "network name cannot be found"|Folder had NTFS permissions but was never actually shared|
|Client couldn't see the domain at all|VM adapter on the wrong virtual network — IP was in a different subnet entirely|
|GPO applied to nothing|Test account lived in a container, not an OU|
|GPO applied to nothing, second time|Policy was created but never linked to anything|
|Group Policy wallpaper rendered solid black|PNG isn't a supported format for that policy — and Group Policy never validates the path|
|Switch management page unreachable|Router and switch shipped with the same default IP|
|DHCP role install threw a validation warning|A DHCP server needs a static address before it can serve leases|

\---

## Why this exists

Every node in the map answers three questions: what it is, why it's built that way, and
how it was set up or fixed. Certifications show you studied. A lab you can walk someone
through shows you understood it.

## Running it locally

No build step, no dependencies — a single self-contained HTML file.

```bash
git clone https://github.com/tyrese-morris/Homelab.git
cd homelab
open index.html    # or just double-click it
```

## Tech

Plain HTML, CSS, and vanilla JavaScript. The map layout and connector lines are
generated at runtime from a single data structure, so adding a new lab means adding one
object — no markup changes.

