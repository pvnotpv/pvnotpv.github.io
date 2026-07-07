---
title: Lighthouse Ethereum Consensus - Phase0 rust-libp2p Networking stack deep dive 
date: 2026-07-07
categories: [ethereum]
tags: [consensus]    
pin: true
description: Code level review on the implementation of Phase0 consensus specification
---

PS - This is ongoing , I will be updating it time to time before making the full post.

> It's been almost 3 months since I've been really going low level on geth and lighthouse and currently at the Networking stack of lighthouse. 

> So currently I'm on the phase0 branch which is almost a few years old and the code have obviously changed a lot but the foundation is still the same and no better way to learn than go to the past!

```bash
pv@arch ~/lighthouse ((v2.0.0))> git branch
* (HEAD detached at v2.0.0)
  stable
pv@arch ~/lighthouse ((v2.0.0))>
```

The thing is that by the time of writing this there is apparently not much articles nor even the eth2 book is not updated of the networking section but I guess that had been the best part about this 
because I've been head deep on the code and figuring it out how it works under the hood and honestly suprised with the amount of progress I've made!

Now the networking specifications from ethererum with not much explanations might kinda seem insane at first but it's honestly just really simple

Yes I'm talking about this: https://ethereum.github.io/consensus-specs/phase0/p2p-interface/

So before reading this make sure to have solid understanding of asynchronus rust, rust-libp2p and how gossipub works, because that is literally the whole thing about this codebase and if you have a strong understanding of that , the codebase is going to feel average for real!

Here's the explanation that could get you 100% understanding of the lighthouse phase0 rpc, I won't be going much into gossipsub and discovery , which is for another article.

Let's perform a simple ping request and see the exact 
