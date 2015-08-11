---
layout: post
title:  "Learning Cap'n Proto RPC"
tags:
 - Raft
 - Rust
 - CSC466
 - CSC462
---

Awhile ago, I wrote a [First Look at Cap'n Proto](/2015/02/11/capn-proto-in-rust/). Unfortunately I didn't cover how to utilize it's RPC capabilities. In Rust, this is via the [`capnp-rpc-rust`](https://github.com/dwrensha/capnp-rpc-rust) crate.

Let's do that!

## What's RPC?
Remote Procedure Calls (RPC) are basically what they say on the box. You issue one from some client to some server and the server responds (or not!) with some response.

Most protocols have some form of language-agnostic schema, not unlike SQL I suppose, which they use to describe interchange data (`struct`s and `fn`s). A tool is generally used to create a **stub**, which the programmer then architects.

> One of the orgininal Remote Procedure Call papers can be found [here](http://research.cs.wisc.edu/areas/os/Qual/papers/rpc.pdf). Cap'n Proto has it's own [RPC Spec](https://capnproto.org/rpc.html).


So if you want to `foo()` on `bar` you can do that. But how is this different than just shuffling off packets to one another?

**Networks are unreliable and unordered.** Your UDP packets might get dropped, reordered, or mishandled. *"Ha! I'll use TCP!"* You say? Sure, okay, this solves part of the issue. Are you willing to pay the round-trips? What happens when the other host goes down? What happens when the router dies? How do you respond to your client? A protocol like Cap'n Proto has more well defined failure modes.

**You don't need to keep track of so many things.** Say we're using UDP, that means we need to label packets and track which ones we haven't recieved responses for. Okay, sure, what happens when we have a couple get lost? Do you "garbage collect" them? Do they leak forever? Using TCP? You're no much better off, you need to either pay the handshake cost *every call* or track all those connections!

**It implements a more understandable interface.** You write code once, then you read it many more times. Then you rewrite it, then cycle repeats. How often do you want to spend trying to trace packets through your application? Does it scale? Do you really expect contributors to understand your mess? RPCs look and feel more like Local Procedure Calls (LPC) and can make things easier to comprehend.

## Our Example

Today we'll be implementing mock calls for the [Raft](http://ramcloud.stanford.edu/raft.pdf) protocol. **On page 4** you'll find the calls that we'll be implementing.

In short:

* `AppendEntries` which appends entries to a replicated log on all other Raft nodes in the cluster.
* `RequestVote` which a `Candidate` uses to request the vote of other nodes so it may become a `Leader`.

There is also another call specified, but that is left as practice for the reader. (Feel free to share your results!)

## In the Schema

Cap'n Proto uses `interface` to declare calls. Let's take a look what will be `src/schema/raft.capnp`:

    @0xf64213cd3ccb41d5;
    # unique file ID, generated by `capnp id`

    interface Raft {
        appendEntries @0 (term :UInt64,
                          leaderId :UInt64,
                          prevLogIndex :UInt64,
                          prevLogTerm :UInt64,
                          entries :List(Text),
                          leaderCommit :UInt64)
                          -> (term :UInt64, success :Bool);
        requestVote @1 (term :UInt64,
                        candidateId :UInt64,
                        lastLogIndex :UInt64,
                        lastLogTerm :UInt64)
                        -> (term :UInt64, voteGranted :Bool);
    }

Just like in the structs we declared in the previous article, these are also decorated with `@n`. Again, it's probably a good idea to keep them in order.

One neat thing is **multiple returns**. This means you can return more than just a single struct or value from a call.

One gotcha is that, as in the previous article, the data types don't translate directly into familiar interfaces. For example, I was not able to successfully create a `:List(Text)` from a vector, some adaptation is required.

> If you don't have Cap'n Proto set up yet (or it's out of date) you can follow the instructions [here](http://www.hoverbear.org/2015/02/12/capn-proto-in-rust/#capnproto).

From here, you can generate your `.rs` file.

    capnp compile -o rust src/schema/*.capnp

You should now see `src/schema/raft_capnp.rs` and it will contain the generated implementation. In RPC terms, this are your **stubs**.

*Don't cross your eyes too much at that code.* The just of it is that Cap'n Proto is mostly driven by `Reader` and `Builder` implementations. You also might notice that `List` types need to be constructed a bit differently.

> This all might feel a bit weird because Cap'n Proto lays things out in memory such that they are **serialized at rest, before being sent**. [Read more on the encoding...](https://capnproto.org/encoding.html)

## Building Off Stubs

Before we even get into Rust code make sure you have the following packages in your `Cargo.toml`:

    [dependencies]
    capnp = "*"
    capnpc = "*"
    capnp-rpc = "*"

In `src/main.rs` of our test project we'll need to have some imports:

    extern crate capnp;
    extern crate capnpc;
    extern crate "capnp-rpc" as capnp_rpc;
    mod raft_capnp {
        include!("./schema/raft_capnp.rs");
    }
    use std::thread;
    use raft_capnp::raft as Raft;
    use capnp::capability::{FromServer, Server};
    use capnp_rpc::capability::{InitRequest, LocalClient, WaitForContent};
    use capnp_rpc::ez_rpc::{EzRpcServer, EzRpcClient};

*Note:* Because `capnp-rpc` is not a valid crate name we need to alias it. We'll also use `mod raft_capnp { include!(...) }` because of some [scoping issues with the Cap'n Proto implementation](https://github.com/dwrensha/capnpc-rust/issues/5). Since our `raft_capnp::raft` would normally be capitalized if it were a native Rust implementation I've gone ahead and done that as well.

Next, in order to make our stubs something more than just *nothing* we'll go ahead and implement them. In our case, `RaftImpl` is an empty struct since we're just fiddling. But this is a good place to put stateful things since the handlers will recieve a `&mut self`.

    struct RaftImpl;
    impl Raft::Server for RaftImpl {
        fn append_entries(&mut self, mut context: Raft::AppendEntriesContext) { }
        fn request_vote(&mut self, mut context: Raft::RequestVoteContext) { }
    }

## Working with Context

So you might have looked at the parameters to our calls and realized *they don't look anything like the ones we wrote in the schema*. They show up in `mut context`.

> **Why?** Because, like `struct`s in Cap'n Proto, RPC calls are also laid out in memory in creative ways. So we'll need to use getters to access them.

Let's work through getting the parameters inside of our `AppendEntries` call.


    let (params, mut results) = context.get();

So `context` breaks down into the `params`, and the `results`. The idea is that you read from the params and write to the (mutable) results.

    let term = params.get_term();
    let leader_id = params.get_leader_id();
    let prev_log_index = params.get_prev_log_index();
    let prev_log_term = params.get_prev_log_term();
    let leader_commit = params.get_leader_commit();

Accessors look pretty standard here, the parameters come out as types we'd expect (In this case, `u64`). For a List things are a bit different.

    let entries = {
        let target = params.get_entries();
        let size = target.len();
        let mut entries = Vec::with_capacity(size as usize);
        for i in 0..size {
            entries.push(target.get(i).to_string());
        }
        entries
    };

Unfortunately, I wasn't able to figure out a painless way of extracting a full set of values from the list without doing a manual walk like this. However, if you're working with 'real' code you might be able to avoid such things. **The scope is important here, note how we limit it.**

At the end of the call we can set the results and close the context.

    results.set_term(1u64);
    results.set_success(true);

    context.done();

After, Cap'N Proto will go and return our results. It should be noted that some cases requests can be *piplined* into other requests. I haven't dug too deeply into this yet but you can explore more [in this example](https://github.com/dwrensha/capnp-rpc-rust/tree/master/examples/calculator). Once I have a more firm understanding of things I'll probably write about pipelining.

## Erecting a Server

The server part of our application will consume our `RaftImpl` and listen on a specific address. In order to simplify testing, we'll kick it off into a new thread because once `.serve()` is called the thread will block.

    thread::spawn(move || {
        let rpc_server = EzRpcServer::new("localhost:8080").unwrap();
        let raft_server = Box::new(Raft::ServerDispatch { server : Box::new(RaftImpl)}) as Box<Server+Send>;
        rpc_server.serve(raft_server);
    });


In the RPC repository's example, [@dwrensha](https://github.com/dwrensha) mentiones that there should be a better way to create `raft_server` but I've not been able to find any myself. If you have suggestions please [speak your peace](https://github.com/dwrensha/capnp-rpc-rust/issues)!

## Working with a Client

Creating a client is fairly straightforward, the only thing I would note is that the `::new("$ADDRESS")` parameter is the **server**'s address.

    let mut rpc_client = EzRpcClient::new("localhost:8080").unwrap();
    let raft_client: Raft::Client = rpc_client.get_main();

Finally, creating a request involves using the `Builder` implementation from the Rust code we generated from the Schema. Again, creating a `List` is a bit different than you might be used to.

    println!("Issuing append_entries.");
    {
        let mut request = raft_client.append_entries_request();
        let mut builder = request.init();

        builder.set_term(0u64);
        builder.set_leader_id(0u64);
        builder.set_prev_log_index(0u64);
        builder.set_prev_log_term(0u64);
        // Do entries in a sec.
        builder.set_leader_commit(0u64);
        // Entry creation is scoped.
        {
            let mut entries = builder.init_entries(2u32);
            entries.set(0, "Foo");
            entries.set(1, "Bar");
        }
        let mut promise = request.send();

        let response = promise.wait().unwrap();
        let term = response.get_term();
        let success = response.get_success();
    }

## Further Exploration

These are unsolved problems that I've been working on which I'm hoping the community can offer suggestions on. I'll update this post and credit any bright ideas!

**Issuing calls to many servers at the same time**. Simply attempting to `.wait()` in a loop is wholly inadequate for the implementation of `RequestVote`. How can we do better? Perhaps with multiple clients and a 'barrier'? [Issue here](https://github.com/dwrensha/capnp-rpc-rust/issues/5).

    // TODO

**Having other events occuring on the server thread.** In our implementation of [Raft](https://github.com/hoverbear/raft) we'd ideally like to keep the requirements of a `RaftNode` down to a single thread, but we need to have timer events. I'm curious if this might be able to integrate with [MIO](https://github.com/carllerche/mio) to do this.

    // TODO

**Pipelining multiple requests.** Cap'n Proto has a feature called "Pipelining" which allows you to make multiple RPC calls in only one network trip (with some limitations). I have yet to come up with an understandable, simple example for this, suggestions?

	// TODO

## Closing Thoughts

As before, one of the biggest stumbling blocks to using Cap'n Proto in Rust is the lack of documentation. There are only a few examples and the majority of the code does not have `rustdoc` comments, so exploring the API is usually a best-effort attempt. If you have an interest in writing documentation I'd highly suggest this project!

I think that Cap'n Prot's RPC mechanisms offer something better then just shuffling around packets. In addition to allowing for more comprehension, it can also improve compatability and speed.

> This post is discussed on [Reddit](https://www.reddit.com/r/rust/comments/2yp7cb/learning_capn_proto_rpc/).

> You can play with the full code [here](https://gist.github.com/Hoverbear/7099220c233ecdf7fe8b).

> Thanks to [@dwrensha](http://github.com/dwrensha/) for proofreading this!