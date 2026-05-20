---
layout: post
title: "How One Engineer in Pakistan Took Down YouTube for the Planet"
date: 2026-03-20
description: "BGP is how the internet routes itself. It has zero authentication. The consequences keep happening."
---

In 2008, one engineer in Pakistan accidentally took down YouTube for the entire planet.

Here's how.

**BGP**, Border Gateway Protocol, is how networks talk to each other. Each one announces "I can reach these IP addresses, send traffic to me." That's how your request finds its way from your laptop to a server on the other side of the world.

It has **zero authentication**. No certificate. No password. No verification.

If a network announces "I own Google's IPs", every other network just believes it and starts sending traffic there.

Pakistan tried to block YouTube locally. An engineer made a BGP announcement that leaked beyond Pakistan's borders. Within minutes, YouTube traffic from around the world was routed to Pakistan instead of YouTube's servers.

YouTube went down globally. For 2 hours. Because one engineer made an announcement that nobody verified.

It keeps happening.

- In **2018**, attackers hijacked Amazon's Route 53 DNS and stole $152,000 in Ethereum by redirecting wallet traffic to a fake server.
- In **2021**, Facebook vanished from the internet for 6 hours. A maintenance command withdrew all their BGP routes. Engineers couldn't even badge into the building to fix it.
- In **2025**, eight root DNS servers were BGP-hijacked simultaneously for 90 minutes.

There is a fix. **RPKI**, basically certificates for BGP announcements. Adoption is at 51%. Government networks? Under 1%.

The protocol that routes all internet traffic was built in 1989 on the assumption that nobody would lie.

People lie all the time.
