---
authors: [dmerej]
slug: introducing-rusync
date: 2018-05-08T17:59:06.225602+00:00
draft: false
title: "Introducing rusync"
tags: [rust]
---

Today I wrote my first "real" rust project.

It's a re-write of `rsync` in Rust called `rusync`.

Here's what its installation and usage look like:

```
$ cargo install rusync
$ rusync test/src test/dest
:: Syncing from test/src to test/dest …
-> foo/baz.txt
-> foo/bar.txt
 ✓ Synced 2 files (1 up to date)
```

You can find the sources [on github](https://github.com/dmerejkowsky/rusync).

What makes this project interesting, I think is how the ETA is computed. Let me explain.

# How the EAT is computed

My goal was to have an ETA that was both precise _and_ fast.

## First attempt

My first attempt was an algorithm like this:

* Walk through every folder and every file in the source folder
* Compute the size of each file and store it in RAM
* Start a counter of all the bytes that were transfer
* Start transferring files
* At the core of the implementation, every time a new chunk of bytes is written,
  add it to the counter and update the progress bar.
* If the file was skipped, add its size to the counter directly

This worked well and gave a very precise ETA, but there was a huge problem: if the source folder
is big, it can take quite a whole to compute the total size. Plus, once this is done,
we have to walk the source folder *again* to perform the transfer.

## Second attempt

After a while, I dediced I could manage to measure progress *and* perform the transfer in
*the same times* by using Rust's built-in concurrency features, namely, the `mpsc` (multiple producer,
single consumer) library.

Here's how it works:

When rusync starts, we build a "pipeline" of workers, using channels made of `Entry` and `ProgressMessage` types

The `Entry` is a plain old struct containing file metadata like size and mtime.
The `ProgressMessage` is an enum type:

```rust
pub enum ProgressMessage {
    // other variants emitted for brievity
    Todo {
        num_files: u64,
        total_size: usize,
    },
    Syncing {
        size: usize,
        done: usize,
    },
}
```
            |- SyncWorker  -|
WalkWorker -|               |- ProgressWorker
            |- StatsWorker -|

`Entry` is a plain old struct

* The WalkWorker sends `Entry` structs to both the SyncWorker and the StatsWorker
* The SyncWorker takes entries, perform the transferm and sends `ProgressMessage` struct to the ProgressWorker
* Similarly, the `StatsWorker` collects size information from the `Entry` it gets and emits its own `ProgressMessage`.

```rust
// In walk()
let mut todo = 0;
for fs_entry in fs_entries {
    let message = ProgressMessage::ToDo{ total_size };
    progress_sender.send(message);
}
```


* `DoneSyncing` is the message use to tell the whole pipeline to stop.
* `StartSync` allows the ProgressWorker to know about the name of the file being transferred. It is emitted by the SyncWorker everytime it opens a new file for reading.
* `Syncing` is emitted everytime a chunk is written to the dest file

```rust
// In sync_entry()
let message = ProgressMessage::StartSync{...};
progress_sender.send(message);

let num_read = src_file.read(...);
dest_file.write_all(...);
let progress = ProgressMessage::Syncing {
    size: src_size as usize,
    done: num_read,
};
progress_sender.send(progress);
```

All that's left to do is for the ProgressWorker to process ProgressMessage in a nice `match` clause:

```rust
for progress in self.input.iter() {
    match progress {
        ProgressMessage::Todo {
            num_files,
            total_size,
        } => {
            stats.num_files = num_files;
            stats.total_size = total_size;
        }
        ProgressMessage::Syncing { done, size, .. } => {
            file_done += done;
            total_done += done;
            ...
            // ETA is computed here!
        }
    }
}
```

And there you have it :) Each source file is only read exactly once. The total number of bytes to be transferred
is updated both when:
* new files are discovered in the source folder
* a new chunk of bytes has been written
* and even when the file is skipped (not show here)

# Feedback request

I wrote this because I wanted to give Rust a try.

If you're already are a Rust developer, I'd appreciate it if you could give me a honest review of the code I wrote.

See the [contact page]({{< ref "/pages/contact.md" >}}) for all the possible ways to reach me, and many thanks in advance!

The main feature is that I tried to have a reliable _and_ fast ETA displayed during the transfer

