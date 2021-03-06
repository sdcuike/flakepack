# FlakePack: A decentralized, k-ordered id generation MessagePack service in Scala

FlakePack is a port of [Flake](https://github.com/boundary/flake) to Scala.

FlakePack produces 128-bit, k-ordered ids (read time-ordered lexically). Run one on each node in your infrastructure and they will generate conflict-free ids on-demand without coordination.

Read the original [post](http://blog.boundary.com/2012/01/12/flake-a-decentralized-k-ordered-unique-id-generator-in-erlang/) on the Boundary blog.

FlakePack uses a MessagePack RPC interface through Netty to send it's ids.

After building / getting the right versions for msgpack-rpc (Java) and msgpack-scala, you can run the service through SBT:

	sbt run

This will start the service on your "localhost" ethernet device, with 6000 as default port.
You can change the default ethernet device and port while starting up:

	sbt "run en0 17000"

You can run a simple test suite to check if everything is working correctly:
	
	sbt test


# Deployment

FlakePack is a standalone application. Request ids with a <tt>generateID</tt> through a MessagePack interface.
If you want to the service to generate multiple ids in one batch, use the generateIDs method and pass the number of ids as a int.

# Anatomy

FlakePack ids are 128-bits wide described here from most significant to least significant bits.

* 64-bit timestamp - milliseconds since the epoch (Jan 1 1970)
* 48-bit worker id - MAC address from a configurable device
* 16-bit sequence # - usually 0, incremented when more than one id is requested in the same millisecond and reset to 0 when the clock ticks forward

# Frequently Asked Questions

**How does this differ from snowflake developed at Twitter?**

The differences stem primarily from the fact that Twitter snowflake ids are 64-bits wide. This means that additional coordination is required to pick a worker id (twitter does this via a ZooKeeper ensemble) and a new epoch must be defined so that timestamps can fit into a small number of bits. Their scheme works great when your ids must fit into 64-bits. However this comes at the cost of additional coordination among nodes and a system that is generally a little more difficult to reason about. It is a fine system though and we were able to learn from it in our efforts.

**How is FlakePack different from rearranging the bits of a UUID-1 id?**

First, successive UUID versions aim to make ids increasingly _opaque_ in nature through various means. We have actually found a great deal of utility in structurally transparent unique ids and that has motivated much of this work.  The most transparent variant is UUID-1 (for which it has received a fair bit of criticism) and thus the nearest relative to a flakepack id. There are some important differences though.

UUID-1 is an odd beast. First, the timestamp is based on the number of 100 nanosecond intervals since October 15, 1582. This is not how most of us familiar with Unix timestamps reason about time. If that isn't bad enough, the timestamp is an odd 60-bits in length with the most significant bits shifted to the least significant bits of the UUID. This property makes lexical ordering essentially meaningless. The remaining bits contain a clock id (initially set to a random number) and a node id (usually the MAC address).

The first problem is the timestamp. We could rearrange the bits to get some k-ordering love, but reasoning on timestamps of this nature makes reasoning about the resulting ids more complex than it needs to be. This is why flakepack uses a standard 64-bit Unix timestamp, unaltered, as the most significant bits.

The next problem is clock skew and protection against replaying ids for a time in the past. The UUID-1 spec dictates that the clock id be incremented in such an event, but this behavior is implementation-specific and we aren't aware of any Erlang implementations that met our safety goals. Flake durably writes a timestamp to a dets table periodically while running. Following a restart, flakepack will refuse to startup if the timestamp written there is from the future. Furthermore, flakepack will refuse to generate ids if it detects that the system clock is running backwards.

**When are FlakePack ids _not_ appropriate?**

Flake ids are predictable by design. Don't use use flakepack to generate ids that you'd rather be unpredictable. Don't use flakepack to generate passwords, security tokens, or anything else you wouldn't want someone to be able to guess.

Flake ids expose the identity of the machine which generated the id (by way of its MAC address) and the time at which it did so. This could be a problem for some security-sensitive applications.

Don't do modulo 2 arithmetic on flakepack ids with the expectation of random distribution. The least significant 16-bits are usually going to be 0 on a machine that is generating an average of one or fewer ids per millisecond.

