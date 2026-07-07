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

(This is for the RPC behaviour implementation)

rpc/mods.rs #154-165

```rust
    pub fn send_request(
        &mut self,
        peer_id: PeerId,
        request_id: RequestId,
        event: OutboundRequest<TSpec>,
    ) {
        self.events.push(NetworkBehaviourAction::NotifyHandler {
            peer_id,
            handler: NotifyHandler::Any,
            event: RPCSend::Request(request_id, event),
        });
    }


```

Here the OutBoundRequest being the ping request with the sequence number.

```rust
pub enum OutboundRequest<TSpec: EthSpec> {
    Status(StatusMessage),
    Goodbye(GoodbyeReason),
    BlocksByRange(BlocksByRangeRequest),
    BlocksByRoot(BlocksByRootRequest),
    Ping(Ping),
    MetaData(PhantomData<TSpec>),
}
```

Here we're going to make an outbound request

In rpc/handler.rs#L492-435

```rust

    fn inject_event(&mut self, rpc_event: Self::InEvent) {
        match rpc_event {
            RPCSend::Request(id, req) => self.send_request(id, req),
            RPCSend::Response(inbound_id, response) => self.send_response(inbound_id, response),
            RPCSend::Shutdown(reason) => self.shutdown(Some(reason)),
        }
    }
```

This is exactly where the request we made from the behaviour reaches the RPC handler.

```rust
    fn send_request(&mut self, id: RequestId, req: OutboundRequest<TSpec>) {

        match self.state {
            HandlerState::Active => {
                self.dial_queue.push((id, req));
            }
```

The request will be added to a dial queue which the handler will pick up on polling.

In hander.rs #L881-895 we can see that

```rust
        // establish outbound substreams
        if !self.dial_queue.is_empty() && self.dial_negotiated < self.max_dial_negotiated {
            self.dial_negotiated += 1;
            let (id, req) = self.dial_queue.remove(0);
            self.dial_queue.shrink_to_fit();
            return Poll::Ready(ProtocolsHandlerEvent::OutboundSubstreamRequest {
                protocol: SubstreamProtocol::new(
                    OutboundRequestContainer {
                        req: req.clone(),
                        fork_context: self.fork_context.clone(),
                    },
                    (),
                )
                .map_info(|()| (id, req)),
            });
        }

```
Now looking at the libp2p documentation for protocols handler we can see this

```rust
///   1. Dialing by initiating a new outbound substream. In order to do so,
///      [`ProtocolsHandler::poll()`] must return an [`ProtocolsHandlerEvent::OutboundSubstreamRequest`],
///      providing an instance of [`libp2p_core::upgrade::OutboundUpgrade`] that is used to negotiate the
///      protocol(s). Upon success, [`ProtocolsHandler::inject_fully_negotiated_outbound`]
///      is called with the final output of the upgrade.
///
```

First the libp2p stream is given to the upgrade_outbound where the protocol negotiations happens, if the particlar stage is passed the stream is handled to the handler!

handler.rs #373 

```rust
    fn inject_fully_negotiated_outbound(

  };
            if self
                .outbound_substreams
                .insert(
                    self.current_outbound_substream_id,
                    OutboundInfo {
                        state: awaiting_stream,
                        delay_key,
                        proto,
                        remaining_chunks: expected_responses,
                        req_id: id,
                    },
                )
                .is_some()
            {

```

Here we can see that the outbound stream is saved, now for the inbound stream same the upgrade_inbound is where protocol negotiations happen if that stage is pssed then 
    fn inject_fully_negotiated_inbound( is called!

```rust
            self.inbound_substreams.insert(
                self.current_inbound_substream_id,
                InboundInfo {
                    state: awaiting_stream,
                    pending_items: vec![],
                    delay_key: Some(delay_key),
                    protocol: req.protocol(),
                    remaining_chunks: expected_responses,
                },

```

Again we can see that the inbound substream is saved!

(Here i haven't gone much into upgrade_inbound and upgrade_outbound since it's not doing that much for real, just a few lines of code)

The handler poll function is really simple where first it checks the delays of each items in the inbound and outbound streams and is removed and a timeout is issued.

Now here comes the interesting part how is the response given?

handler.rs #L606 - 

```rust
    let mut substreams_to_remove = Vec::new(); // Closed substreams that need to be removed
        for (id, info) in self.inbound_substreams.iter_mut() {
            loop {
                match std::mem::replace(&mut info.state, InboundState::Poisoned) {
                    InboundState::Idle(substream) if !deactivated => {
                        if !info.pending_items.is_empty() {
                            let to_send = std::mem::take(&mut info.pending_items);
                            let fut = process_inbound_substream(
                                substream,
                                info.remaining_chunks,
                                to_send,
                            )
```
