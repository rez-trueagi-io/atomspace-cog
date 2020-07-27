
AtomSpace CogStorage Client
===========================
The code in this git repo allows an AtomSpace to communicate with
other AtomSpaces by having them all connect to a common CogServer.
The CogServer itself also provides an AtomSpace, which all clients
interact with, in common.  In ASCII-art:
```
 +-------------+
 |  CogServer  |
 |    with     |  <-----internet------> Remote AtomSpace A
 |  AtomSpace  |  <---+
 +-------------+      |
                      +-- internet ---> Remote AtomSpace B

```

Here, AtomSpace A can load/store Atoms (and Values) to the CogServer,
as can AtomSpace B, and so these two can share AtomSpace contents
however desired.

This provides a simple, unsophisticated backend for AtomSpace storage
via the CogServer. At this time, it is ... not optimized for speed,
and is super-simplistic.  It is meant as a proof-of-concept for
a scalable distributed network of AtomSpaces; one possible design being
explored is the
[AtomSpace OpenDHT backend](https://github.com/opencog/atomspace-dht).

Example Usage
-------------
Well, see the examples directory for details. But, in brief:

* Start the CogServer at "example.com":
```
$ guile
scheme@(guile-user)> (use-modules (opencog))
scheme@(guile-user)> (use-modules (opencog cogserver))
scheme@(guile-user)> (start-cogserver)
$1 = "Started CogServer"
scheme@(guile-user)> Listening on port 17001
```
Then create some atoms (if desired)

* On the client machine:
```
$ guile
scheme@(guile-user)> (use-modules (opencog))
scheme@(guile-user)> (use-modules (opencog persist))
scheme@(guile-user)> (use-modules (opencog persist-cog))
scheme@(guile-user)> (cogserver-open "cog://example.com/")
scheme@(guile-user)> (load-atomspace)
```

That's it! You've copied the entire AtomSpace from the server to
the client!  Of course, copying everything is generally a bad idea
(well, for example, its slow, when the atomspace is large). More
granular load and store is possible; see the
[examples directory](examples) for details.

Status
------
This is Version 0.6. All eighteen (nine+nine) unit tests consistently
pass(***).  Performance looks good. Four of the unit tests take about
a minute to run; this is intentional, they are pounding the server with
large datasets.

Note: (***) There is a bug in the cogutils logger shutdown sequence,
which sometimes causes it to crash after a unit test has finished,
and after `main()` has exited(!). This bug will show up here as a test
failure, even though the test itself passed. The bug is reported in the
[cogserver repo, issue #34](https://github.com/opencog/cogserver/issues/34),

Design
------
There are actually two implementations in this repo. One that is
"simple", and one that is multi-threaded and concurrent (and so
should have a higher throughput). Both "do the same thing",
functionally, but differ in network usage, concurrecncy, etc.

=== The Simple Backend
This can be found in the [opencog/persist/cog-simple](opencog/persist/cog-simple)
directory.  The grand-total size of this implementation is less than 500
lines of code. Seriously! This is really a very simple system!  Take a
look at [CogSimpleStorage.h](opencog/persist/cog-simple/CogSimpleStorage.h)
first, and then take a look at
[CogSimpleIO.cc](opencog/persist/cog-simple/CogSimpleIO.cc)
which does all of the data transfer to/from the cogserver. Finally,
[CogSimpleStorage.cc](opencog/persist/cog-simple/CogSimpleStorage.cc)
provides init and socket I/O.

This backend can be accessed via:
```
scheme@(guile-user)> (use-modules (opencog persist-cog-simple))
scheme@(guile-user)> (cog-simple-open "cog://example.com/")
```

=== The Production Backend
This backend opens four sockets to the cogserver, and handles requests
asynchronously. Be sure to pepper your code with `(barrier)` to flush
the network buffers, else you might get unexpected behavior.
