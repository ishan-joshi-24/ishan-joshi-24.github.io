---
layout: post
title: "Rowhammer: Breaking Computer Security With Pure Physics"
date: 2026-04-15
description: "Read the same memory address a few million times. Watch bits flip in memory you were never allowed to touch."
---

In 2014, researchers at Carnegie Mellon asked a strange question:

*What happens if you read the same memory address a few million times in a row?*

The answer broke computer security. They called it **Rowhammer**.

Your RAM is billions of tiny capacitors, each holding the charge for one bit. They're packed so densely that activating one row of memory over and over leaks charge from the rows beside it, flipping bits you never touched, in memory you were never allowed to read.

No bug. No exploit. Just physics.

Theoretically, once you can flip bits you don't own, the sky is the limit:

- Escalate a regular process into root.
- Escape a browser sandbox from a webpage.
- Break out of a VM into the hypervisor.
- Even read secrets like private keys by inferring neighbor bits from which ones flip.

In practice? Real attacks are finicky. Precise timing, specific hardware, a lot of luck. But mitigations keep falling: DDR4's "fixes" were broken in 2021, and DDR5 attacks started landing in 2025.

Still, memory isolation is the bedrock assumption of every OS, hypervisor, and browser sandbox. And Rowhammer breaks it using pure physics. No software. No code. Just capacitors bleeding into each other.

Which is honestly just cool as hell.
