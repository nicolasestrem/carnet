---
title: "Why I stopped fighting Obsidian sync"
subtitle: "Four sync strategies, one stubborn vault, and the realisation that I was solving the wrong problem."
section: carnets
type: essay
date: 2026-05-19
tags:
  - publish/carnets
  - obsidian
  - sync
  - tooling
  - essay
status: prototype
---

> [!summary]
> I tried four ways to keep an Obsidian vault in sync across three operating systems. Each one failed in a different, instructive way. This is what each tool taught me, and why I eventually paid for the option I'd been avoiding.

## The setup, briefly

Three devices. A Linux desktop, a Windows laptop, an Android phone. One vault — call it ~3,000 markdown files, a few hundred attachments, and the usual archaeological strata of plugins. The job is unglamorous: when I write something on one device, the others should know about it. Conflicts should be rare, and when they happen, I should not lose work.

I'm going to walk through four attempts in the order I made them. The lesson isn't *which tool is best* — it's that the failures had a shape, and the shape was always me reaching for the wrong abstraction.

## Attempt 1 — Git

The first instinct of anyone who's lived in a terminal: this is text, just put it in Git.

It works. For about a week. Then you forget to pull on the laptop before opening the vault, edit the same note as your phone did this morning, and now you have `<<<<<<< HEAD` markers inside what used to be a thought.

```text
<<<<<<< HEAD
Idea about cat tracker batteries
=======
Idea about LTE-M coverage in rural areas
>>>>>>> origin/main
```

Both halves are mine. Both halves are real. Neither half wants to argue with the other in YAML.

> Git wants deliberate, manual merges of source code. Notes are continuous, automatic, and personal. The abstraction doesn't fit.

I ran Git for three months. The friction was tolerable on two devices and intolerable on three. The phone (Termux) was where the wheels came off.

## Attempt 2 — Obsidian LiveSync (CouchDB)

The next move felt sophisticated. Self-hosted CouchDB on a homelab box, the [LiveSync plugin](https://github.com/vrtmrz/obsidian-livesync) on every device. Real-time chunk-level sync, peer-to-peer in spirit, my data on my hardware. What's not to love?

What I learned, the hard way: **chunk sync is a state machine, and state machines hate hurried users.**

The model is: every file is split into chunks; chunks sync to a remote CouchDB; conflicts are resolved at the chunk level. If every device finishes pushing before any device starts pulling, the world is beautiful. If you close the laptop mid-sync and pick up the phone in the elevator, the chunks have opinions about each other that no human wants to read.

My corruption log from that era — yes, I kept one — has entries like:

```
2025-08-14  vault opened in safe mode, 17 conflict markers
2025-08-22  daily note ate its own frontmatter, restored from backup
2025-09-03  chunk-not-found error, "fetch remote" hung at 78%
```

The protocol was sound. My discipline was not. The cost of "let every device finish syncing before you touch another" turned out to be higher than the value of "real-time everywhere."

## Attempt 3 — Syncthing

Syncthing is the answer most people land on after Git. It's a fundamentally different idea: peer-to-peer file replication, not version control. Newest file wins. When there's a true conflict, the loser is preserved as `filename.sync-conflict-YYYYMMDD-HHMMSS.md`, so you never *lose* anything — you just have to look at conflict files now and then.

For two months, this was the best of the four. Then I noticed two things.

First, Android. Syncthing on Android is — politely — a project. Battery optimisation kills it, the share-folder UX is hostile, and you spend more time confirming notifications than reading notes.

Second, the conflict files accumulate. Not many — a handful per week — but each one demands a decision. *Was this the version where I added the link, or the one where I deleted the typo?* The answer is usually obvious; the cost is that I'm now doing janitorial work on my own notes, and the whole point of a notes system is that it disappears.

> Syncthing didn't fail. It just didn't disappear.

## Attempt 4 — Obsidian Sync

I'd put this off for two years on principle. It's a subscription. It's not self-hosted. The principle was real.

Then I added up the time I'd spent on the previous three: Git friction, LiveSync corruption, Syncthing janitorial. Conservatively, an hour a week. Sometimes much more. At $8/month, the question stopped being *"do I want to pay for sync?"* and became *"is my hour cheaper than eight dollars?"*

I subscribed. The setup was four clicks. The phone caught up in a minute. The conflict count, six months in, is zero.

What I'd been resisting wasn't the cost. It was the admission that this was a problem I shouldn't be solving myself.

## The four options, side by side

| | **Git** | **LiveSync (CouchDB)** | **Syncthing** | **Obsidian Sync** |
| --- | --- | --- | --- | --- |
| **Real-time** | No | Yes | ~30s | Near-instant |
| **Conflict model** | Merge markers in-file | Chunk-level, complex | Newer wins, loser preserved | Built-in versioning |
| **Operational burden** | High (pull discipline) | High (sync discipline) | Medium (Android in particular) | Near zero |
| **Cost** | Free | Free + infra | Free | Paid (subscription) |
| **Self-hosted** | Optional | Yes | Yes (P2P, no central) | No |
| **Data sovereignty** | Yours | Yours | Yours | Encrypted end-to-end, not yours |
| **Mobile experience** | Painful | Adequate | Hostile | Native |
| **Failure mode** | Merge conflicts | Chunk corruption | Conflict file pile-up | Vendor risk |

## What this taught me

Three things I'd say to my younger self.

**1. "Tools for X" beat "tools used as X."** Git was designed for deliberate source-code merges; using it for notes is using a chainsaw to butter toast. LiveSync was designed for real-time replication; using it without sync discipline is leaving the chainsaw on the kitchen counter. Syncthing was designed for cross-device folder mirroring; that's actually what I wanted, but the constraints (battery, mobile UX) pushed against my actual usage.

**2. Operational burden compounds.** A 30-second sync ritual is invisible the first day, annoying the first week, and a deal-breaker by the first month. The right system is the one you stop noticing.

**3. There's a moment when self-hosting becomes a hobby, not a strategy.** I love running my own infrastructure. I run more than I probably should. But notes sync was the wrong fight. It was high-frequency, high-stakes (a corrupted note is a lost thought), and the off-the-shelf solution was *good enough that "good enough" was actually best.*

## The pattern, not the recommendation

I'm not telling you to subscribe to Obsidian Sync. The right answer depends on your devices, your tolerance for ops, your principles, and how much your hour is worth.

I'm telling you that **the friction is the signal**. Every time my sync setup made me think about my sync setup, that was a tax I was paying on every note I tried to take. Subtract that tax and the notes get noticeably better — not because the tool wrote them, but because I wasn't thinking about the tool when I did.

If you're four months into your own sync odyssey and still patching: it might be time to stop patching.

---

*Vault size at the time of writing: ~3,000 notes. Devices: Linux desktop, Windows laptop, Android phone. Sync setup: Obsidian Sync, plus a nightly Git push for archival. Yes, that's belt-and-braces. No, I'm not sorry.*
