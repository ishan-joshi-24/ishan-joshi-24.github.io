---
layout: post
title: "How Git Fits 1.4 Million Commits of Linux Into 6 GB"
date: 2026-04-08
description: "Git stores every commit as a full snapshot, not a diff. So how is the entire Linux kernel history only 6 GB?"
---

The Linux kernel. 1.4 million commits. Two decades of history.

How much storage would you guess that takes?

100 GB? 50 GB?

The source code alone is 2 GB. The entire Git repo, with every commit, every version of every file since 2005, is about 8 GB.

Two decades of history compressed into 6 GB. How?

Because Git doesn't store what you'd expect.

Most people assume Git stores diffs, the lines you added, the lines you deleted. It doesn't. Every commit is a full snapshot of your entire project. A photograph of everything at that moment.

Sounds like it should take up a lot of space. It doesn't.

Say your project has 93,000 files and you change one. Git stores the complete new version of that file, not the diff, the whole thing. But the other 92,999 files that didn't change? Git just points to the copies it already has. Same content = same hash = stored once.

Then Git goes further. It packs objects into **packfiles**, delta-compressing similar blobs against each other. Then zlib compresses everything on top of that.

So yes, every commit is technically a full snapshot. But in practice, it's a snapshot made almost entirely of pointers to things Git already knows.

That's how 1.4 million commits fit into 6 GB.
