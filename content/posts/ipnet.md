---
title: "The exasperated engineers guide to IPs in Golang"
date: 2023-01-04T13:51:26-06:00
---

If you've ever needed to store an IP address in Go, you will have come across either [net.IP](https://pkg.go.dev/net#IP) or [netip.Addr](https://pkg.go.dev/net/netip#Addr), or both. If you aren't sure about any of the following:

 * Why are there two of them? (historical reasons)
 * Is one objectively better than the other? (netip.Addr is always better)
 * Which should I use? (netip.Addr)
 * What's the difference? (see below)

Or if you need to understand the implementation details well enough to translate between them, then this article is for you. If you found this article by angrily googling about working with IP addresses and subnets in Go, after discovering that such a simple and obvious thing which should be easy is not, you may rest assured dear friend, I'm kind of salty about it too. As someone who swims in this swamp daily for my day job, I carry around a **silly** amount of hogwash-context relating to the lovely mess that is IP addressing in Go. So read on if you dare as I unburden myself of some cognitive load, and hopefully help make this whole `net.IP` vs. `netip` swamp a little less murky.

### package net

The first data types intended to represent IPs and subnets were built-in to the [net](https://pkg.go.dev/net) package. [net.IP](https://pkg.go.dev/net#IP) and [net.IPNet](https://pkg.go.dev/net#IPNet) were created to hold IP's and subnet addresses respectively. The underlying implementation is a byte-slice, which, at first glance probably makes some sense to you (it did to the net lib creators). _What is an IP address, after all, but a series of dot-separated byte values?_ (Spoiler alert: An IP address is **not** a series of dot-separated byte values.)

```
// a net.IP is just a byte slice
type IP []byte

// a net-mask is the same thing
type IPMask []byte

// So a net.IPNet just wraps an IP and an IPMask
type IPNet struct {
	IP   IP     // network number
	Mask IPMask // network mask
}
```
This implementation is handy because it can model both v4 and v6 addresses (4 bytes for v4, 16 for v6),  but working with it turns out to be kind of a pain in the neck. In fact if you examine any of the instance methods defined on these types you'll immediately encounter a conditional that looks something like:

```
if ip4 := ip.To4(); ip4 != nil {

```

One of the problems with net.IP and net.IPNet is that, in real life, v4 and v6 addresses almost always wind up needing slightly different handling in some way, and the net primitives don't provide us a very handy means of telling them apart once our value is packed into the slice. So working with net.IP becomes an exercise in incessantly writing these clumsy _does it convert to a v4 address?_ conditionals every time you need to tell them apart (which is always). That probably sounds whiny, but writing these over and over begins to feel inhumane more quickly than you'd expect. 

A more serious problem with the type is that it isn't [comparable](https://go.dev/ref/spec#Comparison_operators) (doesn't support `==`) and therefore can't be used as a map key. Also, being a slice, it carries with it mutability, and a zero value that equates to a very inconvenient address (0.0.0.0). 

Further, for all of these problems, the type isn't particularly _good_ at anything. For any human-esque operations like printing, parsing and presenting we wish to perform, we need to convert up to decimal or hex formatted strings. For any math-esque operations we may want to perform, like computing the nth address of a given subnet, or detecting whether address x is contained within subnet y, we need to convert down to bitwise operations on sliced-out bytes. And worst of all, for any computery operations we want to do, like creating a map of net.IP to struct, the correct answer is undefined, so everyone does something unique and surprising.

If we need to fiddle around with the type format for EVERY operation we perform, perhaps the byte-slice is not an ideal abstraction for network addresses? And given binary operations are kind of the lowest common denominator in this use case, maybe we should just keep the binary value of the address in a pair of uint64's instead. Using a uint64 buys us a comparable, immutable base-type, and handily aligns all the Bit-twiddling operations to a pair of 64-bit registers (or a single register for v4 addresses) which is computationally cheap. Whatever extra cost we incur converting uint64 to hex or decimal strings will be more than made up for by all the time we're saving by not having to glean the size of the byte-slice all the time to tell what kind of address we have, because with an int we can use a bitwise operation to separate v4 from v6. In fact, it turns out EVERYTHING we ever want to do computationally with an IP address reduces to a small number of bitwise operations, because that was _entirely the point_ of IP in the first damn place.

This brings me back to my earlier rhetorical: _what is an ip address if not a dot-separated series of byte values_? and of course, the correct answer is: _a binary number_

### package net/netip

The newish **netip** package follows this logic, reimplementing ips and subnets using a "uint128" struct type that wraps a pair of uint64's. NetIP began as an alternate, external project, but made SO much sense, it was gravitationally slurped into standard lib as [net/netip](https://pkg.go.dev/net/netip). Thus today we have four super-confusingly named types for ips and subnets, all awkwardly cohabitating like frenemies in a 90's sitcom in the standard lib. The [netip.Addr](https://pkg.go.dev/net/netip#Addr) type replaces net.IP and holds IPs, and the [netip.Prefix](https://pkg.go.dev/net/netip#Prefix) type replaces net.IPNet and holds subnet addresses. Super obvious. What could go wrong?

```
// the uint128 base-type wraps two uint64's
// it comes with a rich assortment of handy instance methods 
// to help us do math on it
type uint128 struct {
	hi uint64
	lo uint64
}

// netip.Addr replaces net.IP
// It ships with an intern.Value to give it a distinct nil value
// and make v4/v6 checks even faster than a mask op 
type Addr struct {
	addr uint128      // the address
	z *intern.Value   // a metadata tuple
}

// netip.Prefix replaces net.IPNet
// Instead of carrying the mask bytes like net.ipnet
// we wrap an Addr and a CIDR style mask length integer. 
type Prefix struct {
	ip Addr
	bits int16 // really a uint8 with an extra bit to encode validity
}
```

This implementation fixes all of the problems I listed above with net lib, giving us an immutable, comparable type, that can wrap both v4 and v6 addresses and whose validity is distinct from its zero-value. As a bonus, it's quite efficient, saves heap allocations (more on that later), and having been given the benefit of hindsight, ships with tons of handy instance methods to help us do things like easily select v4 vs v6, present human-readable formats, and compute the sorts of things we're likely to want to compute like subnet memberships and etc..

With the net pkg, it's pretty common to reach "inside" the type to directly access the wrapped slices, whereas netip exports nothing BUT instance methods. That may sound awkward, but I find the uniformity quite nice. With net.IP, I would sometimes forget a method even existed and reflexively start writing code against the byte-slice where, by comparison, EVERYTHING you do with a netip.Addr begins with a method invocation, so you tend to build better muscle memory with the API. To me, package net feels like a suitcase you never finish unpacking, and netip, more like a tool you just pick up and use.

IMHO you should always choose netip.Addr in green-field projects and everywhere else for that matter. When you come across legacy code that only supports net.IP, I think the "right" thing to do is add ipnet.Addr support and deprecate net.IP. Some people say I'm a dreamer, but I'm not the only one.


### Don't pass netip pointers
Another problem with the package net types is that they will always be allocated on the heap, because the compiler doesn't know how much memory to reserve for a byte-slice ahead of time. Netip however, with its concrete underlying int types can be stored on the stack at compile time, saving you allocation and garbage collection, which is a big win if you're dealing with zillions of ips per picosecond like I do. You can undermine the compiler however, by passing pointers to netip types.
around...

```
func doCoolStuff(prefix *netip.Prefix){  // wrong. passing a ref might result in a heap alloc 

func doCoolStuff(prefix netip.Prefix){  // correct. The compiler will stack this prefix
```


### Translating net to netip

If you have a `net.IP` and you want a `netip.Addr`, use the `netip.AddrFromSlice()` function:

```
// convert net.IP to netip.Addr
func netip2Addr(ip net.IP) (netip.Addr, error) {
	if addr, ok := netip.AddrFromSlice(ip); ok {
		return addr, nil
	}
	return netip.Addr{}, errors.New("Invalid IP")
}
```
I'm bothering to parse the ok value and return an error above, but it's also common to ignore `ok` and return an invalid or uninitialized Addr in the event of an error, because the type carries a validity flag that the caller can check with `Addr.IsValid()`.

If you have a `net.IPNet`, and you want a `netip.Prefix`, you'll need to reach-in to the net.IPNet, glean the address and CIDR length, and use those to build a Prefix with `netip.PrefixFrom()`. The Addr you can create by using `netip.AddrFromSlice()` like the last example, and the CIDR length you can get from a pkg/net method call against the `net.IPNet.Mask` value.

```
// convert net.IPNet to netip.Prefix
func ipnet2Prefix(ipn net.IPNet) netip.Prefix {
    addr, _ := netip.AddrFromSlice(ipn.IP)
    cidr, _ := ipn.Mask.Size()
    return netip.PrefixFrom(addr, cidr)
}
```

### Translating netip to net

If you have a netip.Addr, and you want a net.IP, no worries, you can use the `Addr.AsSlice()` method, and simply type-assert the returned byte-slice into a `net.IP`.

```
// convert a netip.Addr to net.IP
func addr2NetIP(addr netip.Addr) net.IP {
    return addr.AsSlice()
}
```
If you want to dummy proof this, check the validity of the Addr before you make the conversion...

```
// more safely convert a netip.Addr to net.IP
func addr2NetIP(addr netip.Addr) (net.IP, error) {
    if addr.IsValid(){
        return addr.AsSlice(), nil
    }
    return net.IP{}, errors.New('Invalid IP')
}
```

And finally, if you have a netip.Prefix, and you want a net.IPNet, things get a little complicated, but the idea is to obtain the Addr and prefix length from the Prefix, convert these to byte-slices and manually build a net.IPNet struct with them.

```
// convert a netip.Prefix to a net.IPNet
func prefix2IPNet(prefix netip.Prefix) net.IPNet {
    addr := prefix.Addr() // extract the address portion of the prefix
    pLen := 128 // plen is the total size of the subnet mask
    if addr.Is4() {
        pLen = 32
    }
    ones := prefix.Bits() // ones is the portion of the mask that's set
    ip := net.IP(addr.AsSlice()) // convert the address portion to net.IP
    mask := net.CIDRMask(ones, pLen) // create a net.IPMask 
    return net.IPNet{  // and construct the final IPNet
        IP:   ip,
        Mask: mask,
    }
}
```

Anyway, take it easy.

--dave
