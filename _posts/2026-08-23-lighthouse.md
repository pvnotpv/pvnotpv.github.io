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

Messages supported in phase0:

```rust
pub enum InboundRequest<TSpec: EthSpec> {
    Status(StatusMessage),
    Goodbye(GoodbyeReason),
    BlocksByRange(BlocksByRangeRequest),
    BlocksByRoot(BlocksByRootRequest),
    Ping(Ping),
    MetaData(PhantomData<TSpec>),
}
```

## LibP2P

First let's start with Libp2p itself, First the diagram then the explanation! 

![diagram2](/images/2.jpg)

I cannot give an explanation better than this on how the requests work out between a sender and a receiver:

![diagram2](/images/3.png)


```bash
/lighthouse/beacon_node/eth2_libp2p/src/rpc/mod.rs
```

We have the RPC events defined for libp2p:

```rust
/// RPC events sent from Lighthouse.
pub enum RPCSend<TSpec: EthSpec> {
    Request(RequestId, OutboundRequest<TSpec>),
    Response(SubstreamId, RPCCodedResponse<TSpec>),
}

/// RPC events received from outside Lighthouse.
pub enum RPCReceived<T: EthSpec> {
    Request(SubstreamId, InboundRequest<T>),
    Response(RequestId, RPCResponse<T>),
}
```


(Removed the other stuffs to focus on the main stufff...)

So the entry points to the RPC behaviour is through the functions send_request() and send_response() defined in /rpc/mod.rs

Starting from a request:

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

This event will be picked up the libp2p handler.

```rust
    fn inject_event(&mut self, rpc_event: Self::InEvent) {
        match rpc_event {
            RPCSend::Request(id, req) => self.send_request(id, req),
            RPCSend::Response(inbound_id, response) => self.send_response(inbound_id, response),
            RPCSend::Shutdown(reason) => self.shutdown(Some(reason)),
        }
    }
```
```rust
    fn send_request(&mut self, id: RequestId, req: OutboundRequest<TSpec>) {

        match self.state {
            HandlerState::Active => {
                self.dial_queue.push((id, req));
            }
            _ => self.events_out.push(Err(HandlerErr::Outbound {
                error: RPCError::HandlerRejected,
                proto: req.protocol(),
                id,
            })),
        }
    }
```

The request will be put into a dial queue:

```rust
    fn send_request(&mut self, id: RequestId, req: OutboundRequest<TSpec>) {

        match self.state {
            HandlerState::Active => {
                self.dial_queue.push((id, req));
            }
            _ => self.events_out.push(Err(HandlerErr::Outbound {
                error: RPCError::HandlerRejected,
                proto: req.protocol(),
                id,
            })),
        }
    }
```

Now just like how libp2p works , sending a ProtocolsHandlerEvent::OutboundSubstreamRequest request will initiate a connection.

We can see these lines in rpc/handler.rs L881-L895 in the poll function:

```rust
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

Once a stream is opened, the first step is the protocol negotiation part, which happens at upgrade_outbound and upgrade_inbound for the sender and receiver, respectively. Now, if an agreement is done, then a request is sent to the next function, which we'll see.

![diagram2](/images/4.png)

Here we can see that other than the encoding much not much is happening.

Now let's take a look at the receiver side inbound_upgrade:

![diagram2](/images/5.png)

Again if the encoding matched the request is accepted.

From libp2p docs , after the protocol negotiation part the next function that'd be called is "inject_fully_negotiated_inbound" function for reciever and the outbound function for receiver. And we can see it defined in the rpc handler!

```rust
    fn inject_fully_negotiated_inbound(
        &mut self,
        substream: <Self::InboundProtocol as InboundUpgrade<NegotiatedSubstream>>::Output,
        _info: Self::InboundOpenInfo,
    ) {
        // only accept new peer requests when active
        if !matches!(self.state, HandlerState::Active) {
            return;
        }

        let (req, substream) = substream;
        let expected_responses = req.expected_responses();

        // store requests that expect responses
        if expected_responses > 0 {
            // Store the stream and tag the output.
            let delay_key = self.inbound_substreams_delay.insert(
                self.current_inbound_substream_id,
                Duration::from_secs(RESPONSE_TIMEOUT),
            );
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
            );
        }

        // If we received a goodbye, shutdown the connection.
        if let InboundRequest::Goodbye(_) = req {
            self.shutdown(None);
        }

        self.events_out.push(Ok(RPCReceived::Request(
            self.current_inbound_substream_id,
            req,
        )));
        self.current_inbound_substream_id.0 += 1;
    }
```

Now a lot is going on in this function; the expected response is nothing but the number of responses each request is expecting... For example, the status function just requires one response, whereas the blocksbyroot/range requires multiple responses.

Most importantly, the stream is passed by libp2p, where both the clients can send/receive stuff.

In RPCHandler we can see the inbound_streams array (hashmap, actually) where all the inbound requests are pushed into, and later during polling it's processed.

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
            );
```

The stream given by libp2p from the low level from the protocol negotiations that has happened is saved here. Initially the state is idle.

For outbound requests, again, nothing much other than the function "inject_fully_negotiated_outbound" pushes the substream to the array.

---

Now we're at the stage where both the sender and the receiver has a stream opened between them to send a receive stuffs.

When an inbound request is received for the the receiver, in "inject_fully_negotiated_inbound" , we can see these lines:

```rust
        self.events_out.push(Ok(RPCReceived::Request(
            self.current_inbound_substream_id,
            req,
        )));
```

This is the actual part where the libp2p behaviour (during polling) lets the top-level stack know that it received a request.

Just for now I'm going to put a black box around the actual processing of requests that happens at the higher level; we'd go into it when we look at the higher levels, but now at the libp2p level all it cares about is sending and receiving...

So the above lines will let the higher levels know a request has been received—it has processed it somehow and calls "send_response" in rpc/mod.rs, which injects an event into the handler and calls the "send_response" function in the handler.

```rust
    fn send_response(&mut self, inbound_id: SubstreamId, response: RPCCodedResponse<TSpec>) {
        // check if the stream matching the response still exists
        let inbound_info = if let Some(info) = self.inbound_substreams.get_mut(&inbound_id) {
            info
        } else {
            if !matches!(response, RPCCodedResponse::StreamTermination(..)) {
                // the stream is closed after sending the expected number of responses
                trace!(self.log, "Inbound stream has expired, response not sent";
                    "response" => %response, "id" => inbound_id);
            }
            return;
        };

        // If the response we are sending is an error, report back for handling
        if let RPCCodedResponse::Error(ref code, ref reason) = response {
            self.events_out.push(Err(HandlerErr::Inbound {
                error: RPCError::ErrorResponse(*code, reason.to_string()),
                proto: inbound_info.protocol,
                id: inbound_id,
            }));
        }

        if matches!(self.state, HandlerState::Deactivated) {
            // we no longer send responses after the handler is deactivated
            debug!(self.log, "Response not sent. Deactivated handler";
                "response" => %response, "id" => inbound_id);
            return;
        }
        inbound_info.pending_items.push(response);
    }
```

It takes the particular substream and pushes the response to the pending_items; the pending_items here are the expected responses the requests are expecting.

(Currently we're at the receiver side.)

handler.rs #L610-L#624, which is from the poll function.

```rust
                    InboundState::Idle(substream) if !deactivated => {
                        if !info.pending_items.is_empty() {
                            let to_send = std::mem::take(&mut info.pending_items);
                            let fut = process_inbound_substream(
                                substream,
                                info.remaining_chunks,
                                to_send,
                            )
                            .boxed();
                            info.state = InboundState::Busy(Box::pin(fut));
                        } else {
                            info.state = InboundState::Idle(substream);
                            break;
                        }
                    }
```

We can see it checks the pending items and send the response to the stream using "process_inbound_substream" function,

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
        } else if matches!(item, RPCCodedResponse::StreamTermination(_)) {
            // The sender closed the stream before us, ignore this.
        } else {
            // we have more items after a closed substream, report those as errors
            errors.push(RPCError::InternalError(
                "Sending responses to closed inbound substream",
            ));
        }
    }
    (substream, errors, substream_closed, remaining_chunks)
}
```

These lines is where the magic happens:

```rust
                substream
                    .send(item)
                    .await
                    .unwrap_or_else(|e| errors.push(e));
```

It's literally sending response to the substream!

Now let's take a look at the sender side that is expecting the response.

hander.rs#L746

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

```

Now if the expected responses are matched then:

```rust

                        let received = match response {
                            RPCCodedResponse::StreamTermination(t) => {
                                Ok(RPCReceived::EndOfStream(id, t))
                            }
                            RPCCodedResponse::Success(resp) => Ok(RPCReceived::Response(id, resp)),
                            RPCCodedResponse::Error(ref code, ref r) => Err(HandlerErr::Outbound {
                                id,
                                proto,
                                error: RPCError::ErrorResponse(*code, r.to_string()),
                            }),

```

The lifecyle has been completed now (at the libp2p level) , where the sender has gotten a response now.

## Behaviour

![diagram2](/images/6.png)

Behaviour is the main, core behaviour that provides the high level abstraction for all other behaviours. 

Let's take a look at the Behaviours' processing of the rpc events.

```rust
// RPC
impl<TSpec: EthSpec> NetworkBehaviourEventProcess<RPCMessage<TSpec>> for Behaviour<TSpec> {
    fn inject_event(&mut self, event: RPCMessage<TSpec>) {
        let peer_id = event.peer_id;

        if !self.peer_manager.is_connected(&peer_id) {
            debug!(
                self.log,
                "Ignoring rpc message of disconnecting peer";
                "peer" => %peer_id
            );
            return;
        }
```

From L-899

```rust
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
```

We can see that for ping requests the request is directly handled from the behaviour itself.

Where as for the other requests the request needs stuff fromt the beacon chain, so it's sent further high up the stack,

Now guess it's time for us to go even higher level where we'd see how this works.

## Service

(eth2_lib2p/src/service.rs)

> I might be a bit vague right here on this part since there is not much going on other than the default libp2p stuffs.


![diagram2](/images/7.png)

Here's the more in-depth view:

![diagram2](/images/8.png)



