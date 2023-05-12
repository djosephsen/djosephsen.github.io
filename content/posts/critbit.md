---
title: "Critbit Tries"
date: 2022-11-08T20:36:53-07:00
draft: true
---

As I write this, the fastest line-cards running in the fastest backbone routers on the internet operate at 101-Gigabit speeds, and the IEEE is hard at work on [400 and 800 gig-e standards](https://en.wikipedia.org/wiki/Terabit_Ethernet). It's common, I think, to hear about a network speed like 100Gig-e and form a mental picture of a *massive* point-to-point connection -- of untold thousands of jiggerbits from an ocean-sized fire-hose spewing haphazardly from point-a to point-b at the speed of
light.

![pointab](/pointab.png)

That's true, I suppose, in the case of two hosts directly wired together with a cable, but that's not what happens inside a router. Routers... **route**, which is not a trivial process. Given an ethernet frame containing an IP, the router must search for a longest-prefix match for that IP from amoung the collection of all routes it knows about, while the route table itself is constantly being modified.

By "longest prefix", we mean _the most specific route_, because for an address like `8.8.8.8`, we may have several routes that potentially match:
    * 0/32 -- matches everything, routes to transit
    * 8/24 -- more specific, routes to a google ixp peer
    * 8.8/16 -- even more specific, routes to an igp best-path (you get the idea)

## Magical Monoliths
That's an IPv4 address with simple cidr boundaries. But if you consider real-world subnetting in IPv6 address-space, the length of the key, along with the number of potential matches explodes. Similarly, the number of available public routes continues to [grow at non-linear rates](https://blog.apnic.net/2022/01/06/bgp-in-2021-the-bgp-table/). Then consider on top of that, the aforementioned 100Gig-e input speed, potentially feeding the system billions of routing decisions per second.

Billions of lookups per second against a million-object data-store is a distributed-systems-sized problem. And yet these magic monoliths we call "routers" do it all day every day. So how does a router pull this off? How do you design a single box that's capable of performing billions of longest-prefix searches per second on an IPv6-length search key against a data-set of a million routes that's constantly being modified beneath you?

Very few engineers know the answer to this question. When asked, most networky-flavored engineers will wave their hand and mutter something about _ASICs_, which isn't untrue, but sort of glosses over the _code_ burnt into the ASICS. 

## Critbit
The real answer is complex, but a huge component is a data-structure called a "bitwise trie". There are many flavors, but in this article I want to dive into [critbit trees](https://cr.yp.to/critbit.html) with you. Critbit is a simple, solid, and fast implementation that's not only used in hardware, but also operating systems, and oodles of open-source projects that deal with low-level network operations like [gobgp]() and [cilium]().

We use critbit trees in several of our internal software-defined-networking projects at [Fastly](https://fastly.com). I've never had to implement critbit myself, but I wanted to have a more than passing familiarity with how the critbit library we use actually works, so I spent a day digging into it out of curiosity. What follows is not a mathematically rigorous exploration on the topic, but something more like *rigoresque* tourism. If you're interested in writing your own implementation, you'll need to have a look
at the same material I referenced in my own exploration:

### The screed
DJB's [article](http://cr.yp.to/critbit.html), which I'll refer to as "the screed" is the prototypical Bernstein introduction to the topic, intended for CS adepts who already understand nuance like trie compression and node augmentation.

### The Spec
Adam Langley's [reference implemenation](https://github.com/agl/critbit), which I'll refer to as "the spec" is a well-annotated C implementation of the screed, and is your best bet for signal to noise ratio.

### The Magic
The [critbit-go](https://github.com/agl/critbit) library, which I'll refer to as "The Magic" is the undocumented, under-commented, anti-idiomatic critbit package that other go projects tend to import and rely on. The point of this excercise for me, was unraveling enough of its mystery to feel safe importing it myself.

## Compressed Bitwise Tries
