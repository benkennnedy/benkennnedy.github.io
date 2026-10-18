---
layout: ../../layouts/PostLayout.astro
title: "TCP/IP Vol 2: Chapter 3 Lab"
summary: "BGP lab from Jeff Doyle's TCP/IP Vol 2"
date: 2026-09-17
type: "Post"
category: "Routing"
tags: ["BGP", "OSPF", "iBGP", "Route Filtering"]
---

![Three-AS BGP lab topology](/images/posts/bgp-three-as-lab.png)

## Introduction
At the moment I am reading Jeff Doyle's TCP/IP Vol 2 for deep knowledge of BGP, and afterwards I intend to read the multicast section. I've decided not to lab chapters 1 and 2 because I already have good experience with BGP from my ENARSI studies. However, it has been quite some time since I configured BGP so I thought this looked like a good one to get familiar with it again, as the next chapter is about influencing path selection.

## The lab

This lab brought together three autonomous systems and several routing-policy requirements in one topology:

- AS100 uses EIGRP as its IGP.
- AS200 uses OSPF area 0 as its IGP.
- AS300 also uses OSPF area 0 and receives default-only connectivity from AS100.
- AS100 and AS200 have two eBGP links for resilience and path control.
- AS100 uses a full mesh of iBGP peerings.

The main constraint was keeping each AS clean. Internal routes needed to remain inside their own autonomous system unless they were deliberately injected into BGP. The point-to-point links used for eBGP also had to stay out of the internal routing protocols.

## What I configured

I started with the IGPs so that every router could reach the internal infrastructure and loopback addresses in its own AS. The assigned AS100 prefixes were injected into EIGRP at R3 and R4, while the AS200 prefixes were injected into OSPF at R7.

From there, I built the routing policy in stages:

1. Established eBGP between R1 and R5, and between R2 and R6.
2. Advertised the required AS100 prefixes into BGP using `network` statements on R1 and R2.
3. Redistributed the required AS200 OSPF routes into BGP on R5 and R6, using prefix lists and route maps so that no unintended routes were leaked.
4. Built a full mesh of iBGP peerings across AS100 using router loopbacks.
5. Provided AS200 with next-hop reachability without using `next-hop-self` on R5 or R6.
6. Redistributed the external routes into OSPF in a controlled way so that R7 could reach destinations outside AS200 without running BGP.
7. Configured R1 to advertise only a default route to R8 in AS300. R8 then injected that default into OSPF for R9.
8. Used aggregation and route policy on R5 and R6 to influence which AS100 edge link was preferred for different AS200 networks while retaining a backup path.

## What caught me out: `next-hop-self`

The first issue was next-hop reachability inside AS100.

When an edge router learns a prefix through eBGP and advertises it to an iBGP peer, BGP does not change the next-hop attribute by default. This meant an internal AS100 router could learn an AS200 prefix in BGP while still seeing the AS200-facing transit address as its next hop.

Because those eBGP point-to-point networks were intentionally excluded from EIGRP, the internal routers had no route to that next hop. The prefix could appear in the BGP table but would not be usable as expected.

The fix was to configure `next-hop-self` on the AS100 edge routers for their iBGP neighbours:

```text
router bgp 100
 neighbor 192.168.1.2 remote-as 100
 neighbor 192.168.1.3 update-source Loopback0
 neighbor 192.168.1.4 next-hop-self
```

R1 and R2 then advertised themselves as the next hop for the external routes they passed into AS100. Their loopbacks were already reachable through EIGRP, so the internal routers could resolve the BGP next hop correctly.

The useful lesson was that **a route being present in the BGP table does not automatically make it usable**. I also need to check whether its next hop is reachable in the routing table.

## What caught me out: `update-source Loopback0`

The second issue appeared while building the iBGP full mesh.

I configured the neighbours using loopback addresses because they provide a stable endpoint that is independent of any single physical link. However, simply entering a loopback address in the neighbour statement does not make the router source the BGP session from its own loopback.

Without `update-source Loopback0`, the router can source the TCP session from the outgoing physical interface. The remote router is expecting the configured loopback address, so the source does not match the neighbour definition and the session does not establish.

For loopback-based iBGP, both sides therefore need the correct neighbour statement and update source:

```text
router bgp 100
 neighbor <peer-loopback> remote-as 100
 neighbor <peer-loopback> update-source Loopback0
```

The loopback addresses must also be reachable through the IGP. My checklist is now:

1. Can I ping the peer loopback from my own loopback?
2. Is the peer configured against the correct local loopback address?
3. Is `update-source Loopback0` configured on both sides?
4. Does `show ip bgp summary` show the session as established?

## What went well

This lab involved a lot of route filtering, particularly when redistributing routes between different routing protocols and controlling which prefixes were included in BGP advertisements and aggregates. I was comfortable implementing this using route maps and prefix lists to precisely control route propagation.

For example, one requirement was to advertise only a default route from R1 to R8 over an eBGP session. To achieve this, I applied an outbound route map to the neighbour:

`neighbor 172.16.0.10 route-map DEFAULT_ONLY out`

The DEFAULT_ONLY route map matched a prefix list permitting only 0.0.0.0/0:

`ip prefix-list PFL1 permit 0.0.0.0/0`

This ensured that the default route was advertised to R8 while all other BGP prefixes were filtered from the outbound update.


## Verification

I used the following checks throughout the lab:

```text
show ip eigrp neighbors
show ip ospf neighbor
show ip bgp summary
show ip bgp <prefix>
show ip route <next-hop>
show ip route <prefix>
traceroute <destination>
```

I verified both the normal forwarding paths and the backup paths. For AS300, I also confirmed that R8 received only a default route from R1 and that R9 learned the resulting default through OSPF.

## Conclusion

Overall, this lab was very successful. It validated that I still have good BGP configuration and troubleshooting, as well as route filtering, but it also exposed things that I have forgot since my ENARSI studes.

The most valuable part of this lab was separating three different questions:

- Has BGP learned and selected the prefix?
- Is the BGP next hop reachable?
- Does the forwarding path follow the intended policy?

I am away this weekend but thankfully I bought the Kindle version of TCP/IP Vol 2, so I intend to begin reading chapter 4 on the plane. Looking forward to getting deep into BGP path selection!