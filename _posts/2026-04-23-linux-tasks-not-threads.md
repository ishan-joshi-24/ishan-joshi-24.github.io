---
layout: post
title: "Linux Doesn't Know the Difference Between a Process and a Thread"
date: 2026-04-23
description: "You think in processes and threads. The Linux kernel sees one thing: a task. The difference is a bitmask."
---

You think in processes and threads.

The Linux kernel doesn't know the difference.

The textbook says they're two things. A process is heavy and isolated. A thread is lightweight and shares memory.

To Linux, it's one thing: a **task**. Every running thing on your machine is the same struct, `task_struct`. The scheduler just picks one to run.

So where do those two categories come from?

One syscall. `clone()`. It takes a bitmask of flags telling the kernel what the new task should share with its parent:

- `CLONE_VM`: share memory
- `CLONE_FILES`: share file descriptors
- `CLONE_SIGHAND`: share signal handlers
- `CLONE_THREAD`: join the same thread group

None of them: the task gets its own memory, FDs, handlers. That's `fork()`. We call it a process.

All of them: the task shares everything with its parent. That's `pthread_create()`. We call it a thread.

Same syscall. Same struct. Different flags.

And the flags are mostly independent. A task that shares memory but not FDs. A task that shares FDs but has its own memory. Linux lets you.

"Process" and "thread" are just two presets on the same dial. Userspace picked them and gave them names.

Next time someone tells you threads are lightweight processes, take it literally. Linux did.
