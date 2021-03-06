.. github display
   GitHub is NOT the preferred viewer for this file. Please visit
   https://flux-framework.rtfd.io/projects/flux-rfc/en/latest/spec_10.html

10/Content Storage Service
==========================

This specification describes the Flux content storage service
and the messages used to access it.

-  Name: github.com/flux-framework/rfc/spec_10.rst

-  Editor: Jim Garlick <garlick@llnl.gov>

-  State: raw


Language
--------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to
be interpreted as described in `RFC 2119 <http://tools.ietf.org/html/rfc2119>`__.


Related Standards
-----------------

-  :doc:`3/CMB1 - Flux Comms Message Broker Protocol <spec_3>`


Goals
-----

The Flux content storage service is available for general purpose
data storage within a Flux instance. The goals of the content storage
service are:

-  Provide storage for opaque binary blobs.

-  Stored content remains available for the lifetime of the Flux instance.

-  Stored content is immutable.

-  Stored content is available from any broker rank within an instance.

-  Stored content is addressable by its message digest, computed using a
   cryptographic hash.

-  The cryptographic hash algorithm is configurable per instance.

-  Content may be shared between instances

This kind of store has interesting and well-understood properties, as
explored in Venti, Git, and Camlistore (see References below).


Implementation
--------------

The content service SHALL be implemented as a distributed cache with a
presence on each broker rank. Each rank MAY cache content according
to an implementation-defined heuristic.

Ranks > 0 SHALL make transitive load and store requests to their parent on
the tree based overlay network to fill invalid cache entries, and flush
dirty cache entries.

Rank 0 SHALL retain all content previously stored by the instance.

Rank 0 MAY extend its cache with an OPTIONAL backing store, the details
of which are beyond the scope of this RFC.

Rank 0 MAY, as a last resort, attempt to satisfy load requests by making
a transitive request to the enclosing instance, if any.


Content
~~~~~~~

Content SHALL consist of from zero to 1,048,576 bytes of data.
Content SHALL NOT be interpreted by the content service.


Blobref
~~~~~~~

Each unique, stored blob of content SHALL be addressable by its blobref.
A blobref SHALL consist of a string formed by the concatenation of:

-  the name of hash algorithm used to store the content

-  a hyphen

-  a message digest represented as a lower-case hex string

Example:

::

   sha1-f1d2d2f924e986ac86fdf7b36c94bcdf32beec15

Note: "blobref" was shamelessly borrowed from Camlistore
(see References below).


Store
~~~~~

A store request SHALL be encoded as a CMB1 request message with the blob
as raw payload (blob length > 0), or no payload (blob length = 0).

A store response SHALL be encoded as a CMB1 response message with
NULL-terminated blobref string as raw payload, or an error response.

A request to store content that exceeds the maximum size SHALL
receive error number 27, "File too large", in response.

After the successful store response is received, the blob SHALL be
accessible from any rank in the instance.


Load
~~~~

A load request SHALL be encoded as a CMB1 request message with
NULL-terminated blobref string as raw payload.

A load response SHALL be encoded as a CMB1 response message with blob
as raw payload (blob length > 0), no payload (blob length = 0),
or an error response.

A request to load unknown content SHALL receive error number 2,
"No such file or directory", in response.


Flush
~~~~~

A flush request SHALL cause the local rank content service to finish
storing any dirty cache entries. A flush response SHALL NOT be sent
until there are no dirty cache entries.

On rank 0, "dirty" SHALL be defined as "not stored on a backing store".
On rank > 0, "dirty" SHALL be defined as "not stored on rank 0".

A flush request SHALL receive error number 38, "Function not implemented",
on rank 0 if a backing store is not configured.


Dropcache
~~~~~~~~~

A dropcache request SHALL cause the local content service to drop all
non-essential entries from its cache.


Foreign Content
~~~~~~~~~~~~~~~

If a load request cannot be satisfied by the instance’s content service,
a load request MAY be sent to the enclosing instance, if applicable.

The enclosing instance MAY have configured a different hash algorithm.
The content service, therefore, SHALL NOT require that a blobref specified
in a load request match the configured hash.


Garbage Collection
~~~~~~~~~~~~~~~~~~

References to content are unconstrained from the perspective of the
content service, therefore content MUST persist for the lifetime of
the instance.

During instance shutdown, some content MAY be preserved by storing it
in the enclosing instance when the instance is *reaped*. All other
content SHALL be destroyed when the instance terminates.


Message Definitions
~~~~~~~~~~~~~~~~~~~

Content service messages SHALL follow the CMB1 rules described
in RFC 3 for requests and responses, and are described in detail by
the following ABNF grammar:

::

   CONTENT         = C:store-req     S:store-rep
                   / C:load-req      S:load-rep
                   / C:flush-req     S:flush-rep
                   / C:dropcache-req S:dropcache-rep

   ; Multi-part ZeroMQ messages
   C:store-req     = [routing] "content.store" [blob] PROTO
   S:store-rep     = [routing] "content.store" blobref PROTO

   ; Multi-part ZeroMQ messages
   C:load-req      = [routing] "content.load" blobref PROTO
   S:load-rep      = [routing] "content.load" [blob] PROTO

   ; Multi-part ZeroMQ messages
   C:flush-req     = [routing] "content.flush" PROTO
   S:flush-rep     = [routing] "content.flush" PROTO

   ; Multi-part ZeroMQ messages
   C:dropcache-req = [routing] "content.dropcache" PROTO
   S:dropcache-rep = [routing] "content.dropcache" PROTO

   blobref         = hash-name "-" digest %x00
   hash-name   = 1*(ALPHA / DIGIT)
   digest      = 1*(HEXDIG)

   blob            = 0*(OCTET)

   ; PROTO and [routing] are as defined in RFC 3.


References
----------

-  `Camlistore is your personal storage system for life <https://camlistore.org/>`__.

-  `Venti: a new approach to archival storage <http://doc.cat-v.org/plan_9/4th_edition/papers/venti/>`__, Bell Labs, Quinlan and Dorward.

-  `git reference manual <http://git-scm.com/doc>`__
