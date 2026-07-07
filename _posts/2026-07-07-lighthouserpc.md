---
title: Lighthouse Ethereum Consensus - Phase0 rust-libp2p Networking stack deep dive 
date: 2026-07-07
categories: [ethereum]
tags: [consensus]    
pin: true
description: Code level review on the implementation of Phase0 consensus specification
---

PS - This is ongoing; I will be updating it from time to time before making the full post.

> It's been almost 3 months since I've been really going low level on geth and lighthouse and currently at the Networking stack of lighthouse. 

> So currently I'm on the phase 0 branch, which is almost a few years old, and the code has obviously changed a lot, but the foundation is still the same, and there's no better way to learn than to go to the past!

```bash
pv@arch ~/lighthouse ((v2.0.0))> git branch
* (HEAD detached at v2.0.0)
  stable
pv@arch ~/lighthouse ((v2.0.0))>
```

The thing is that by the time of writing this, there are apparently not many articles, nor is even the Eth2 book updated on the networking section, but I guess that has been the best part about this. 

Because I've been head-deep on the code and figuring out how it works under the hood, and honestly, I'm surprised with the amount of progress I've made!

Now the networking specifications from Ethereum with not much explanation might kind of seem insane at first, but it's honestly just really simple.

Yes, I'm talking about this: <https://ethereum.github.io/consensus-specs/phase0/p2p-interface/>

So before reading this, make sure to have a solid understanding of asynchronous Rust, Rust-libp2p, and how Gossipub works, because that is literally the whole thing about this codebase, and if you have a strong understanding of that, the codebase is going to feel average for real!

Here's the explanation that could get you 100% understanding of the lighthouse phase 0 RPC. I won't be going much into gossipsub and discovery, which is for another article.

Let's perform a simple ping request and see the exact process

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
            let awaiting_stream = InboundState::Idle(substream);
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

A quick note on upgrade_outbound and upgrade_inbound

> This is where the protocol versioning and ssz encoding negotiation and snappy compression is taken care of.


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

So currently each request has an expected response which should be satisfied

```
    pub fn expected_responses(&self) -> u64 {
        match self {
            InboundRequest::Status(_) => 1,
            InboundRequest::Goodbye(_) => 0,
            InboundRequest::BlocksByRange(req) => req.count,
            InboundRequest::BlocksByRoot(req) => req.block_roots.len() as u64,
            InboundRequest::Ping(_) => 1,
            InboundRequest::MetaData(_) => 1,
        }
    }

```

So currently for our ping it's one!

Looking at process_inbound_stream

```rust
async fn process_inbound_substream<TSpec: EthSpec>(
    mut substream: InboundSubstream<TSpec>,
    mut remaining_chunks: u64,
    pending_items: Vec<RPCCodedResponse<TSpec>>,
) -> InboundProcessingOutput<TSpec> {
    let mut errors = Vec::new();
    let mut substream_closed = false;

    for item in pending_items {
        if !substream_closed {
            if matches!(item, RPCCodedResponse::StreamTermination(_)) {
                substream.close().await.unwrap_or_else(|e| errors.push(e));
                substream_closed = true;
            } else {
                remaining_chunks = remaining_chunks.saturating_sub(1);
                // chunks that are not stream terminations get sent, and the stream is closed if
                // the response is an error
                let is_error = matches!(item, RPCCodedResponse::Error(..));

                substream
                    .send(item)
                    .await
                    .unwrap_or_else(|e| errors.push(e));

                if remaining_chunks == 0 || is_error {
                    substream.close().await.unwrap_or_else(|e| errors.push(e));
                    substream_closed = true;
                }
            }
```

See the lines

```rust
substream
                    .send(item)
                    .await
                    .unwrap_or_else(|e| errors.push(e));
```

It's literally just sending the response through the substream, but wait we still haven't provided a response, so how do we fill up the pending_items?

This is exactly where the send_response comes down 

handler.rs #L280

```rust
    fn send_response(&mut self, inbound_id: SubstreamId, response: RPCCodedResponse<TSpec>) {

        inbound_info.pending_items.push(response);

```
So this is exactly where a response is pushed down the stream.


Now going back to the outbound side

handler.rs #746

```rust
                OutboundSubstreamState::RequestPendingResponse {
                    mut substream,
                    request,
                } => match substream.poll_next_unpin(cx) {
                    Poll::Ready(Some(Ok(response))) => {
                        if request.expected_responses() > 1 && !response.close_after() {
                            let substream_entry = entry.get_mut();
                            let delay_key = &substream_entry.delay_key;
                            // chunks left after this one
                            let remaining_chunks = substream_entry
                                .remaining_chunks
                                .map(|count| count.saturating_sub(1))
                                .unwrap_or_else(|| 0);
                            if remaining_chunks == 0 {
                                // this is the last expected message, close the stream as all expected chunks have been received

```

The substream is read, checked for expected response!


It's honestly this simple (Assuming here the peer is dialed) 

1. The request is send to the dial queue
2. The poll picks it up
3. Protocol negotiations happens.
4. The substreams are saved awaiting responses.
5. A response is send through the inbound stream
6. The outbound stream gets it


We still haven't talked about the most important part the driving factor , meaning when a request is recieved how is the request being sent?

handler.rs poll() we can see this:

```rust
    fn poll(
        &mut self,
        cx: &mut Context<'_>,
    ) -> Poll<
        ProtocolsHandlerEvent<
            Self::OutboundProtocol,
            Self::OutboundOpenInfo,
            Self::OutEvent,
            Self::Error,
        >,
    > {
        // return any events that need to be reported
        if !self.events_out.is_empty() {
            return Poll::Ready(ProtocolsHandlerEvent::Custom(self.events_out.remove(0)));
        } else {
            self.events_out.shrink_to_fit();
        }
```


Where the request is sent out , which is sent out through the inject_event() function:

behaviour/mod.rs , we can see that:

```rust
impl<TSpec: EthSpec> NetworkBehaviourEventProcess<RPCMessage<TSpec>> for Behaviour<TSpec> {
    fn inject_event(&mut self, event: RPCMessage<TSpec>) {
        let peer_id = event.peer_id;

            Ok(RPCReceived::Request(id, request)) => {
                let peer_request_id = (handler_id, id);
                match request {
                    /* Behaviour managed protocols: Ping and Metadata */
                    InboundRequest::Ping(ping) => {
                        // inform the peer manager and send the response
                        self.peer_manager.ping_request(&peer_id, ping.data);
                        // send a ping response
                        self.pong(peer_request_id, peer_id);
                    }
                    InboundRequest::MetaData(_) => {
                        // send the requested meta-data
                        self.send_meta_data_response((handler_id, id), peer_id);
                    }
                    InboundRequest::Goodbye(reason) => {
                        // queue for disconnection without a goodbye message
                        debug!(
                            self.log, "Peer sent Goodbye";
                            "peer_id" => %peer_id,
                            "reason" => %reason,
                            "client" => %self.network_globals.client(&peer_id),
                        );
                        // NOTE: We currently do not inform the application that we are
                        // disconnecting here. The RPC handler will automatically
                        // disconnect for us.
                        // The actual disconnection event will be relayed to the application.
                    }
                    /* Protocols propagated to the Network */
                    InboundRequest::Status(msg) => {
                        // inform the peer manager that we have received a status from a peer
                        self.peer_manager.peer_statusd(&peer_id);
                        // propagate the STATUS message upwards
                        self.propagate_request(peer_request_id, peer_id, Request::Status(msg))
                    }
                    InboundRequest::BlocksByRange(req) => self.propagate_request(
                        peer_request_id,
                        peer_id,
                        Request::BlocksByRange(req),
                    ),
                    InboundRequest::BlocksByRoot(req) => {
                        self.propagate_request(peer_request_id, peer_id, Request::BlocksByRoot(req))
                    }
                }
            }

```
For ping, metadata and goodbye the request is handled there itself!

Since status, blockbyrange and blocksbyrange requires data from the beacon chain they're handled differently... let's see how that goes...

(Here I won't be going deep into the exact working of the beacon state processor but tracing the execution path.)

```rust
    InboundRequest::Status(msg) => {
                        // inform the peer manager that we have received a status from a peer
                        self.peer_manager.peer_statusd(&peer_id);
                        // propagate the STATUS message upwards
                        self.propagate_request(peer_request_id, peer_id, Request::Status(msg))
                    }
```

Here again remember that behaviour/mod.rs is anotha behaviour!

```rust
    fn propagate_request(&mut self, id: PeerRequestId, peer_id: PeerId, request: Request) {
        self.add_event(BehaviourEvent::RequestReceived {
            peer_id,
            id,
            request,
        });
    }
```

Now it's sends out its own event.

Looking at network/service.rs

```rust
 BehaviourEvent::RequestReceived{peer_id, id, request} => {
                                let _ = service
                                    .router_send
                                    .send(RouterMessage::RPCRequestReceived{peer_id, id, request})
                                    .map_err(|_| {
                                        debug!(service.log, "Failed to send RPC to router");
                                    });
                            }
```

The message is passed through the router.

router/mod.rs

```rust
    fn handle_message(&mut self, message: RouterMessage<T::EthSpec>) {
        match message {
            // we have initiated a connection to a peer or the peer manager has requested a
            // re-status
            RouterMessage::PeerDialed(peer_id) | RouterMessage::StatusPeer(peer_id) => {
                self.processor.send_status(peer_id);
            }
            // A peer has disconnected
            RouterMessage::PeerDisconnected(peer_id) => {
                self.processor.on_disconnect(peer_id);
            }
            RouterMessage::RPCRequestReceived {
                peer_id,
                id,
                request,
            } => {
                self.handle_rpc_request(peer_id, id, request);
            }
```

```rust
   fn handle_rpc_request(&mut self, peer_id: PeerId, id: PeerRequestId, request: Request) {
        if !self.network_globals.peers.read().is_connected(&peer_id) {
            debug!(self.log, "Dropping request of disconnected peer"; "peer_id" => %peer_id, "request" => ?request);
            return;
        }
        match request {
            Request::Status(status_message) => {
                self.processor
                    .on_status_request(peer_id, id, status_message)
            }
            Request::BlocksByRange(request) => self
                .processor
                .on_blocks_by_range_request(peer_id, id, request),
            Request::BlocksByRoot(request) => self
                .processor
                .on_blocks_by_root_request(peer_id, id, request),
        }
    }
```

The request goes through the processor.

Now how the processor working is really interesting. From beacon_processor/mod.rs

```
//! The purpose of the `BeaconProcessor` is to provide two things:
//!
//! 1. Moving long-running, blocking tasks off the main `tokio` executor.
//! 2. A fixed-length buffer for consensus messages.
```

Now here comes the voilia moment!

```
-        self.send_beacon_processor_work(BeaconWorkEvent::status_message(peer_id, status))
-            self.beacon_processor_send
            .try_send(work)

(here the work is sent to the channel)
```

worker/rpc_methods.rs

```rust

    pub fn process_status(&self, peer_id: PeerId, status: StatusMessage) {
        match self.check_peer_relevance(&status) {
            Ok(Some(irrelevant_reason)) => {
                debug!(self.log, "Handshake Failure"; "peer" => %peer_id, "reason" => irrelevant_reason);
                self.goodbye_peer(peer_id, GoodbyeReason::IrrelevantNetwork);
            }
            Ok(None) => {
                let info = SyncInfo {
                    head_slot: status.head_slot,
                    head_root: status.head_root,
                    finalized_epoch: status.finalized_epoch,
                    finalized_root: status.finalized_root,
                };
                self.send_sync_message(SyncMessage::AddPeer(peer_id, info));
            }
            Err(e) => error!(self.log, "Could not process status message"; "error" => ?e),
        }
    }


```

Here the response is set!

Mspc is channel where the asynchronus threads communicate with!

So this is how the lighthouse handles the consensus specification 
