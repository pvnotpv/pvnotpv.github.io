---
title: Lighthouse Networking - LifeCycle of a message from LibP2P to BeaconProcessor to the BeaconChain
date: 2026-08-23
categories: [consensus]
tags: [ethereum, lighthouse, core, networking]     
description: Mapping out the entire Network stack of Lighthouse from top to bottom.
published: true
pin: true
---

So again, I'm really at the end phase of my journey, deep into Lighthouse phase 0, and kind of regret not making notes of some other parts of the codebase. I decided not to make the mistake on the networking part and am now going to make a diagrammatic explanation because the amount of context you'd have to keep in your head is kind of insane considering the vast size of the codebase...

Honestly, it's my first time writing such a huge article, so I'm trying my best to make it count!

Here is the complete mapping of the NetworkStack:

![diagram1](/images/lighthousenet.png)

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

---

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

This event will be picked up by the libp2p handler.

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

Now just like how libp2p works, sending a ProtocolsHandlerEvent::OutboundSubstreamRequest request will initiate a connection.

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

These lines are where the magic happens:

```rust
                substream
                    .send(item)
                    .await
                    .unwrap_or_else(|e| errors.push(e));
```

It's literally sending a response to the substream!

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

Now if the expected responses are matched, then:

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

The lifecycle has been completed now (at the libp2p level), where the sender has gotten a response now.

---

## Behaviour

![diagram2](/images/6.png)

Behaviour is the main, core behaviour that provides the high-level abstraction for all other behaviours. 

Let's take a look at the Behaviours' processing of the RPC events.

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

Whereas for the other requests, the request needs stuff from the beacon chain, so it's sent further up the stack.

Now guess it's time for us to go to an even higher level where we'd see how this works.

---

## Service

(eth2_lib2p/src/service.rs)

> I might be a bit vague right here on this part since there is not much going on other than the default libp2p stuff.


![diagram2](/images/7.png)

Here's the more in-depth view:

![diagram2](/images/8.png)

The "Service" struct is the final abstraction of the LibP2P layer, where the behaviour, networking and the main LibP2P "swarm" are configured.

- The usual Libp2p network configurations, and then we importantly have the swarm being built.

```rust
                SwarmBuilder::new(transport, behaviour, local_peer_id)
                    .notify_handler_buffer_size(std::num::NonZeroUsize::new(7).expect("Not zero"))
                    .connection_event_buffer_size(64)
                    .connection_limits(limits)
                    .executor(Box::new(Executor(executor)))
                    .build(),
                bandwidth,
            )
```

Nothing is too specific other than the usual libp2p configurations.

Again, here are the two important methods being send_request and send_response:

```rust
    /// Sends a request to a peer, with a given Id.
    pub fn send_request(&mut self, peer_id: PeerId, request_id: RequestId, request: Request) {
        self.swarm
            .behaviour_mut()
            .send_request(peer_id, request_id, request);
    }
        /// Sends a response to a peer's request.
    pub fn send_response(&mut self, peer_id: PeerId, id: PeerRequestId, response: Response<TSpec>) {
        self.swarm
            .behaviour_mut()
            .send_successful_response(peer_id, id, response);
    }
```

Alright, now time to hit the most important function, which is the polling.

```rust
    pub async fn next_event(&mut self) -> Libp2pEvent<TSpec> {
        loop {
            match self.swarm.select_next_some().await {
                SwarmEvent::Behaviour(behaviour) => {
                    // Handle banning here
                    match &behaviour {
                        BehaviourEvent::PeerBanned(peer_id) => {
                            self.swarm.ban_peer_id(*peer_id);
                        }
                        BehaviourEvent::PeerUnbanned(peer_id) => {
                            self.swarm.unban_peer_id(*peer_id);
                        }
                        _ => {}
                    }
                    return Libp2pEvent::Behaviour(behaviour);

```

This is exactly where the swarm sends events outside it. Meaning we're basically at the end of Libp2p.

---

## NetworkService

> Hitting the network/ folder now!

![diagram2](/images/9.png)

The NetworkService is the middleman between the low-level networking and the high-level message processing. 

It drives the whole NetworkStack completely, driving the whole high-level and the low-level layers.

![diagram2](/images/10.png)

```rust
/// Service that handles communication between internal services and the `eth2_libp2p` network service.
pub struct NetworkService<T: BeaconChainTypes> {
    /// A reference to the underlying beacon chain.
    beacon_chain: Arc<BeaconChain<T>>,
    /// The underlying libp2p service that drives all the network interactions.
    libp2p: LibP2PService<T::EthSpec>,
    /// An attestation and subnet manager service.
    attestation_service: AttestationService<T>,
    network_recv: mpsc::UnboundedReceiver<NetworkMessage<T::EthSpec>>,
    router_send: mpsc::UnboundedSender<RouterMessage<T::EthSpec>>,

    store: Arc<HotColdDB<T::EthSpec, T::HotStore, T::ColdStore>>,
}
```

The router_send is the channel where the NetworkService sends messages to the Router.


```rust
        let router_send = Router::spawn(
            beacon_chain.clone(),
            network_globals.clone(),
            network_send.clone(),
            executor.clone(),
            network_log.clone(),
        )?;
```

The router_send is:

```rust
        let (handler_send, handler_recv) = mpsc::unbounded_channel();
```

The network channel is where the NetworkService receives responses from the higher stack.

```rust
impl<T: BeaconChainTypes> NetworkService<T> {
    #[allow(clippy::type_complexity)]
    pub async fn start(
        beacon_chain: Arc<BeaconChain<T>>,
        config: &NetworkConfig,
        executor: task_executor::TaskExecutor,
    ) -> error::Result<(
        Arc<NetworkGlobals<T::EthSpec>>,
        mpsc::UnboundedSender<NetworkMessage<T::EthSpec>>,
    )> {
        let network_log = executor.log().clone();
        // build the network channel
        let (network_send, network_recv) = mpsc::unbounded_channel::<NetworkMessage<T::EthSpec>>();

        // router task
        let router_send = Router::spawn(
            beacon_chain.clone(),
            network_globals.clone(),
            network_send.clone(),
            executor.clone(),
            network_log.clone(),
        )?;
    }

```

When a new NetworkService is spawned:

```rust
fn spawn_service<T: BeaconChainTypes>(
    executor: task_executor::TaskExecutor,
    mut service: NetworkService<T>,
) {
    let mut shutdown_sender = executor.shutdown_sender();

    // spawn on the current executor
    executor.spawn(async move {

        let mut metric_update_counter = 0;
        loop {
            // build the futures to check simultaneously
            tokio::select! {

```

Here the NetworkService constantly polls the service, and if a new event is received from libp2p, it's sent to the router for processing.


![diagram2](/images/11.png)

```rust
impl<T: BeaconChainTypes> Router<T> {
    /// Initializes and runs the Router.
    pub fn spawn(
        beacon_chain: Arc<BeaconChain<T>>,
        network_globals: Arc<NetworkGlobals<T::EthSpec>>,
        network_send: mpsc::UnboundedSender<NetworkMessage<T::EthSpec>>,
        executor: task_executor::TaskExecutor,
        log: slog::Logger,
    ) -> error::Result<mpsc::UnboundedSender<RouterMessage<T::EthSpec>>> {
        let message_handler_log = log.new(o!("service"=> "router"));
        trace!(message_handler_log, "Service starting");

        let (handler_send, handler_recv) = mpsc::unbounded_channel();

        // Initialise a message instance, which itself spawns the syncing thread.
        let processor = Processor::new(
            executor.clone(),
            beacon_chain,
            network_globals.clone(),
            network_send,
            &log,
        );

        // generate the Message handler
        let mut handler = Router {
            network_globals,
            processor,
            log: message_handler_log,
        };

        // spawn handler task and move the message handler instance into the spawned thread
        executor.spawn(
            async move {
                debug!(log, "Network message router started");
                UnboundedReceiverStream::new(handler_recv)
                    .for_each(move |msg| future::ready(handler.handle_message(msg)))
                    .await;
            },
            "router",
        );

        Ok(handler_send)
    }
```

Here we can see the initialization of a router where a new handler channel is created to receive messages from the NetworkService. The "handler_send" is returned to the NetworkService to send messages to the Router.

The router folder also contains a Processor struct, which acts like an abstraction for the BeaconProcessor.

![diagram2](/images/12.png)

```rust
/// Processes validated messages from the network. It relays necessary data to the syncing thread
/// and processes blocks from the pubsub network.
pub struct Processor<T: BeaconChainTypes> {
    chain: Arc<BeaconChain<T>>,
    /// A channel to the syncing thread.
    sync_send: mpsc::UnboundedSender<SyncMessage<T::EthSpec>>,
    /// A network context to return and handle RPC requests.
    network: HandlerNetworkContext<T::EthSpec>,
    beacon_processor_send: mpsc::Sender<BeaconWorkEvent<T>>,
}
```

It has 3 channels being defined:

1. "network"—The network channel is passed from the NetworkService, which is passed to the BeaconProcessor to send stuff back to the NetworkService.
2. "beacon_processor_send" - A channel to send events to the BeaconProcessor.


```rust
impl<T: BeaconChainTypes> Processor<T> {
    /// Instantiate a `Processor` instance
    pub fn new(
        executor: task_executor::TaskExecutor,
        beacon_chain: Arc<BeaconChain<T>>,
        network_globals: Arc<NetworkGlobals<T::EthSpec>>,
        network_send: mpsc::UnboundedSender<NetworkMessage<T::EthSpec>>,
        log: &slog::Logger,
    ) -> Self {
        let sync_logger = log.new(o!("service"=> "sync"));
        let (beacon_processor_send, beacon_processor_receive) =
            mpsc::channel(MAX_WORK_EVENT_QUEUE_LEN);

```

A channel is created, and a new BeaconProcessor is created with "beacon_processor_receive" to send events to.

```rust
        BeaconProcessor {
            beacon_chain: Arc::downgrade(&beacon_chain),
            network_tx: network_send.clone(),
            sync_tx: sync_send.clone(),
            network_globals,
            executor,
            max_workers: cmp::max(1, num_cpus::get()),
            current_workers: 0,
            log: log.clone(),
        }
        .spawn_manager(beacon_processor_receive, None);
```

---

## BeaconProcessor

![diagram2](/images/13.png)

Lots of paths so far and finally we've come to where the actual changes are read and written to the BeaconChain.

Here we have a WorkEvent struct:

```rust
pub struct WorkEvent<T: BeaconChainTypes> {
    drop_during_sync: bool,
    work: Work<T>,
}
```

where the Work being:

```rust
pub enum Work<T: BeaconChainTypes> {
    GossipAttestation {
        message_id: MessageId,
        peer_id: PeerId,
        attestation: Box<Attestation<T::EthSpec>>,
        subnet_id: SubnetId,
        should_import: bool,
        seen_timestamp: Duration,
    },
    
    ,
    
        Status {
        peer_id: PeerId,
        message: StatusMessage,
    },
    BlocksByRangeRequest {
        peer_id: PeerId,
        request_id: PeerRequestId,
        request: BlocksByRangeRequest,
    },
    BlocksByRootsRequest {
        peer_id: PeerId,
        request_id: PeerRequestId,
        request: BlocksByRootRequest,
    },
```


The event_rx channel is where the Processor sends the beacon processor the work, just like I said above.

The BeaconProcessor spawns a manager that is constantly being looped, checking for events happening.

```rust
        let manager_future = async move {
            let mut inbound_events = InboundEvents {
                idle_rx,
                event_rx,
                reprocess_work_rx: ready_work_rx,
            };

            loop {
                let work_event = match inbound_events.next().await {
                    Some(InboundEvent::WorkerIdle) => {
                        self.current_workers = self.current_workers.saturating_sub(1);
                        None
                    }
                    Some(InboundEvent::WorkEvent(event))
                    | Some(InboundEvent::ReprocessingWork(event)) => Some(event),
                    None => {
                        debug!(
                            self.log,
                            "Gossip processor stopped";
                            "msg" => "stream ended"
                        );
                        break;
                    }
                };
```

Here the idle channel is for use by workers when they've finished a task.

InboundEvents is a stream that sends the manager when a new work is received through a channel.


```rust
/// Combines the various incoming event streams for the `BeaconProcessor` into a single stream.
///
/// This struct has a similar purpose to `tokio::select!`, however it allows for more fine-grained
/// control (specifically in the ordering of event processing).
struct InboundEvents<T: BeaconChainTypes> {
    /// Used by workers when they finish a task.
    idle_rx: mpsc::Receiver<()>,
    /// Used by upstream processes to send new work to the `BeaconProcessor`.
    event_rx: mpsc::Receiver<WorkEvent<T>>,
    /// Used internally for queuing work ready to be re-processed.
    reprocess_work_rx: mpsc::Receiver<ReadyWork<T>>,
}

impl<T: BeaconChainTypes> Stream for InboundEvents<T> {
    type Item = InboundEvent<T>;

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        // Always check for idle workers before anything else. This allows us to ensure that a big
        // stream of new events doesn't suppress the processing of existing events.
        match self.idle_rx.poll_recv(cx) {
            Poll::Ready(Some(())) => {
                return Poll::Ready(Some(InboundEvent::WorkerIdle));
            }
            Poll::Ready(None) => {
                return Poll::Ready(None);
            }
            Poll::Pending => {}
        }
```

When a new Work is received first it checks if there is an available worker.

```rust
                    }
                    // There is a new work event and the chain is not syncing. Process it or queue
                    // it.
                    Some(WorkEvent { work, .. }) => {
                        let work_id = work.str_id();
                        let toolbox = Toolbox {
                            idle_tx: idle_tx.clone(),
                            work_reprocessing_tx: work_reprocessing_tx.clone(),
                        };

                        match work {
                            _ if can_spawn => self.spawn_worker(work, toolbox),
                            Work::GossipAttestation { .. } => attestation_queue.push(work),
                            // Attestation batches are formed internally within the
                            // `BeaconProcessor`, they are not sent from external services.
                            Work::
```

can_spawn being:

```rust
                let can_spawn = self.current_workers < self.max_workers;
```

If a Worker is not available it's pushed into a FIFO queue.

Now suppose a Worker is available right now, then a new worker is spawned for their respective work.

```rust
    /// Spawns a blocking worker thread to process some `Work`.
    ///
    /// Sends an message on `idle_tx` when the work is complete and the task is stopping.
    fn spawn_worker(&mut self, work: Work<T>, toolbox: Toolbox<T>) {
        let idle_tx = toolbox.idle_tx;
        let work_reprocessing_tx = toolbox.work_reprocessing_tx;

```

Depending on the Work:

```rust
                     */
                    Work::Status { peer_id, message } => worker.process_status(peer_id, message),
                    /*
                     * Processing of range syncing requests from other peers.
                     */
                    Work::BlocksByRangeRequest {
                        peer_id,
                        request_id,
                        request,
                    } => worker.handle_blocks_by_range_request(peer_id, request_id, request),
                    /*
                     * Processing of blocks by roots requests from other peers.
                     */
                    Work::BlocksByRootsRequest {
                        peer_id,
                        request_id,
                        request,
                    } => worker.handle_blocks_by_root_request(peer_id, request_id, request),

```

The work is matched and sent to their respective functions.

![diagram2](/images/14.png)

We have 

![diagram2](/images/16.png)

All these functions are defined for rpc methods.

For example, here is the BlocksByRange request:

![diagram2](/images/15.png)

Voila! We've finally hit the actual BeaconChain!

Now sending response is through the network channel created at the NetworkService 

```rust
    pub fn send_response(
        &self,
        peer_id: PeerId,
        response: Response<T::EthSpec>,
        id: PeerRequestId,
    ) {
        self.send_network_message(NetworkMessage::SendResponse {
            peer_id,
            id,
            response,
        })
    }
```

![diagram2](/images/17.png)

Which is received by the NetworkService struct.

Here I'm again providing the diagram


![diagram2](/images/18.png)

In NetworkService we can see the channel being polled constantly:

![diagram2](/images/19.png)

Which directly sends the responses through libp2p!


---

