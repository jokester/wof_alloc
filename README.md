Wheel-of-Fortune Memory Allocator
=================================

This has turned into a very interesting excercise in algorithms and data
structures. It was extracted from Wireshark's new 'wmem' framework.

History
-------

Version 1 of this allocator was embedded in the original emem framework. It
didn't have to handle realloc or free, so it was very simple: it just grabbed
a block from the OS and served allocations sequentially out of that until it
ran out, then allocated a new block. The old block was never revisited, so
it generally had a bit of wasted space at the end, but the waste was
small enough that it was simply ignored. This allocator provided very fast
constant-time allocation for any request that didn't require a new block from
the OS, and that cost could be amortized away.

Version 2 of this allocator was prompted by the need to support realloc and
free in wmem. The original version simply didn't save enough metadata to do
this, so I added a layer on top to make it possible. The primary principle
was the same (allocate sequentially out of big blocks) with a bit of extra
magic. Allocations were still fast constant-time, and frees were as well.
Large parts of that design are still present in this one, but for more
details see older versions of this file from git or svn.

Version 3 of this allocator was written to address some issues that
eventually showed up with version 2 under real-world usage. Specifically,
version 2 dealt very poorly with memory fragmentation, almost never reusing
freed blocks and choosing to just keep allocating from the master block
instead. This led to particularly poor behaviour under the tick-tock loads
(alloc/free/alloc/free or alloc/alloc/free/alloc/alloc/free/ or ...) that
showed up in a couple of different protocol dissectors (TCP, Kafka).

Blocks and Chunks
-----------------

As in previous versions, allocations typically happen sequentially out of
large OS-level blocks. Each block has a short embedded header used to
maintain a doubly-linked list of all blocks (used or not) currently owned by
the allocator. Each block is divided into chunks, which represent allocations
and free sections (a block is initialized with one large, free, chunk). Each
chunk is prefixed with a wmem_block_chunk_t structure, which is a short
metadata header (8 bytes, regardless of 32 or 64-bit architecture unless
alignment requires it to be padded) that contains the length of the chunk,
the length of the previous chunk, a flag marking the chunk as free or used,
and a flag marking the last chunk in a block. This serves to implement an
inline sequential doubly-linked list of all the chunks in each block. A block
with three chunks might look something like this:

         0                    _________________________
         ^      ___________  /        ______________   \       __________
||---||--|-----/-----------||--------/--------------||--\-----/----------||
||hdr|| prv | len |  body  || prv | len |   body    || prv | len | body  ||
||---||--------------------||--/--------------------||-------------------||
       \______________________/


When allocating, a free chunk is found (more on that later) and split into
two chunks: the first of the requested size and the second containing any
remaining free. The first is marked used and returned to the caller.

When freeing, the chunk in question is marked as free. Its neighbouring
chunks are then checked; if either of them are free, the consecutive free
chunks are merged into a single larger free chunk. Induction can show that
applying this operation consistently prevents us ever having consecutive
free chunks.

Free chunks (because they are not being used for anything else) each store an
additional pair of pointers (see the wmem_block_free_t structure) that form
the backbone of the data structures used to track free chunks.

Master and Recycler
-------------------

The extra pair of pointers in free chunks are used to build two doubly-linked
lists: the master and the recycler. The recycler is circular, the master is
a stack.

The master stack is only populated by chunks from new OS-level blocks,
so every chunk in this list is guaranteed to be able to serve any allocation
request (the allocator will not serve requests larger than its block size).
The chunk at the head of the master list shrinks as it serves requests. When
it is too small to serve the current request, it is popped and inserted into
the recycler. If the master list is empty, a new OS-level block is allocated,
and its chunk is pushed onto the master stack.

The recycler is populated by 'leftovers' from the master, as well as any
chunks that were returned to the allocator via a call to free(). Although the
recycler is circular, we will refer to the element referenced from the
allocator as the 'head' of the list for convenience. The primary operation on
the recycler is called cycling it. In this operation, the head is compared
with its clockwise neighbour. If the neighbour is as large or larger, it
becomes the head (the list rotates counter-clockwise). If the neighbour is
smaller, then it is removed from its location and inserted as the counter-
clockwise neighbour of the head (the list still rotates counter-clockwise,
but the head element is held fixed while the rest of the list spins). This
operation has the following properties:
 - fast constant time
 - once the largest chunk is at the head, it remains at the head
 - more iterations increases the probability that the largest chunk will be
   the head (for a list with n items, n iterations guarantees that the
   largest chunk will be the head).

Allocating
----------

When an allocation request is received, the allocator first attempts to
satisfy it with the chunk at the head of the recycler. If that does not
succeed, the request is satisfied by the master list instead. Regardless of
which chunk satisfied the request, the recycler is always cycled.
