---
layout: post
title: "Every reCAPTCHA We Solved Trained an AI (For Free)"
date: 2026-04-13
description: "$6.1 billion in unpaid human labor. The trick is how Google verified our answers without knowing the answer itself."
---

Every reCAPTCHA image grid we solved trained an AI.

But if the AI doesn't know the answer, how did it know ours was right?

We've all seen those grids, *"select all traffic lights"*, *"click every crosswalk."* Google shows us an image grid and asks us to pick the right ones. Get it wrong and we don't get in.

But here's the thing. It doesn't fully know the answer either.

Some images in the grid already have a known answer. Those are the test. If we get those right, Google trusts our answer on the ones it doesn't know yet.

Then it shows the same unknown image to multiple people. If enough humans agree, that becomes the new ground truth.

That's it. That's how it learns. We weren't just proving we're human. **We were labeling data.**

Street signs. Crosswalks. Hydrants. Buses. Traffic lights. Sourced from Google Street View. All labeled by us. For free.

A 2023 preprint estimated that over 13 years, reCAPTCHA consumed **819 million hours of human labor**. Worth roughly **$6.1 billion in wages**.

If Google ever asks me to select traffic lights again, I'm sending an invoice.
