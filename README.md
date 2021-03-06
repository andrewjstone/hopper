# hopper - an unbounded mpsc with bounded memory

[![Build Status](https://travis-ci.org/postmates/hopper.svg?branch=master)](https://travis-ci.org/postmates/hopper)

Hopper provides a version of the rust
standard [mpsc](https://doc.rust-lang.org/std/sync/mpsc/) that is unbounded but
consumes a bounded amount of memory. This is done by paging elements to disk at
need. The ambition here is to support mpsc style communication without
allocating unbounded amounts of memory or dropping inputs on the floor.

## Quickstart 

Include the hopper library in your Cargo.toml

`hopper = "0.1"` 

and use it in much the same way you'd use stdlib's mpsc:

```
extern crate tempdir;
extern crate hopper;

let dir = tempdir::TempDir::new("hopper").unwrap();
let (mut snd, mut rcv) = hopper::channel("example", dir.path()).unwrap();

snd.send(9);
assert_eq!(Some(9), rcv.next());
```

The primary difference here is that you must provide a name for the channel and
a directory where hopper can page items to disk. 

## Wait, page items to disk?

Imagine that hopper's internal structure is laid out like a contiguous array:

```
[-------------------|~~~~~~~~~~~~~~~~~~. . .~~~~~~~~~~~~~~~~~~~~~~~~]
0                   1024
```

Between the indicies of 0 and 1024 hopper stores items in-memory until they are
retrieved. Above index 1024 items are paged out to disk. This fixes the memory
burden of the system at the expense of disk IO.
    
Hopper is intended to be used in situtations where your system
cannot [load-shed](http://ferd.ca/queues-don-t-fix-overload.html) inputs and
_must_ eventually process them. While hopper does page to disk it will not
preserve writes across restarts, much in the same way as stdlib mpsc.
    
## Inside Baseball

Hopper's channel looks very much like a named pipe in Unix. You supply a
name to either `channel_2` or `channel_with_max_bytes_3` and you push bytes
in and out. The disk paging adds a complication. In private, the name
supplied to the above two functions is used to create a directory under
`data_dir`. This directory gets filled up with monotonically increasing
files in situations where the disk paging is in use. We'll treat this
exclusively from here on.
    
The on-disk structure look like so:
    
```text
data-dir/
   sink-name0/
      0
      1
   sink-name1/
      0
```

You'll notice exports of Sender and Receiver in this module's
namespace. These are the structures that back the send and receive side of
the named channel. The Senders--there may be multiples of them--are
responsible for _creating_ "queue files". In the above,
`data-dir/sink-name*/*` are queue files. These files are treated as
append-only logs by the Senders. The Receivers trawl through these logs to
read the data serialized there.
    
### Won't this fill up my disk?

Nope! Each Sender has a notion of the maximum bytes it may read--which you
can set explicitly when creating a channel with
`channel_with_max_bytes`--and once the Sender has gone over that limit it'll
attempt to mark the queue file as read-only and create a new file. The
Receiver is programmed to read its current queue file until it reaches EOF
and finds the file is read-only, at which point it deletes the file--it is
the only reader--and moves on to the next.
    
### What kind of filesystem options will I need?

Hopper is intended to work on any wacky old filesystem with any options,
even at high concurrency. As common filesystems do not support interleaving
[small atomic
    writes](https://stackoverflow.com/questions/32851672/is-overwriting-a-small-file-atomic-on-ext4)
hopper limits itself to one exclusive Sender or one exclusive Receiver at a
time. This potentially limits the concurrency of mpsc but maintains data
integrity. We are open to improvements in this area.

## What kind of performance does hopper have? 

Hopper ships with a small-ish benchmark suite. For Postmates' workload, writes
of 10 MB/s are not uncommon and we've pushed that to 80 MB/s on occassion. So!
not super pokey but the actual upper-bounds have not been probed and can
probably be expanded. See end note in "What kind of filesystem options will I
need?" as an example of where hopper can be made better in concurrent workloads.
