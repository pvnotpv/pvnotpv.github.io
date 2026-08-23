---
title: Lighthouse - LifeCycle of a message from LibP2P to BeaconProcessor to the BeaconChain
date: 2026-08-23
categories: [consensus]
tags: [ethereum, lighthouse, core, networking]     
description: Mapping out the entire Network stack of Lighthouse from top to bottom.
published: true
---

So again, I'm really at the end phase of my journey, deep into Lighthouse phase 0, and kind of regret not making notes of some other parts of the codebase. I decided not to make the mistake on the networking part and am now going to make a diagrammatic explanation because the amount of context you'd have to keep in your head is kind of insane considering the vast size of the codebase...

Honestly, it's my first time writing such a huge article, so I'm trying my best to make it count!

So where to start?

Oh yeah, I guess I'd just start with the commit since the version of Lighthouse is pretty old:

```bash
pv@arch ~/lighthouse ((v2.0.0))> git show HEAD

commit 7c88f582d955537f7ffff9b2c879dcf5bf80ce13 (HEAD, tag: v2.0.0)
Author: Michael Sproul <michael@sigmaprime.io>
Date:   Tue Oct 5 03:53:18 2021 +0000
```
Again, you need to have an understanding of how asynchronous programming works in Rust and how streams are handled. Also, we won't be going into how libp2p works, but we will be going into how libp2p is implemented on Lighthouse (behaviour, handlers, events) and all that stuff. Make sure you know about it.

Here's a really high-level overview of the entire network stack of Lighthouse: here the beacon chain is sitting at the highest level, where all the state changes are made and stored, and at the lowest, libp2p and gossipsub handle the actual networking stuff.

![diagram1](/images/1.png)
