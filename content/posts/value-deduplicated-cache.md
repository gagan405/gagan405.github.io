+++
title = "A value deduplicated cache library for rust"
date = "2026-08-13"

[taxonomies]
tags=["cache", "rust"]
+++

Back in 2021 or so, while investigating repeated OOM errors in our service, I discovered a peculiar data distribution in a particular cached data. It had all well distributed keys, but all (or most) pointing to the same data. Each data row being already pretty big, the cache was essentially keeping the same data repeated for every key.

I solved that with the obvious double map: `K → hash(V) → V`.

More on that here: https://cafeaffe.substack.com/p/a-value-de-duplicated-cache-1

After years now, I thought of having it as a generic LRU cache library for rust, with some acceptable level of customization and metrics to actually show if the dudup cache is a good fit or not.

Here is the first version of it published today: https://crates.io/crates/dedup-cache

A generic, concurrent, value deduplicated cache. Fully pluggable on top of any I/O or computations.

The architecture and the usage are well documented in the repo: https://github.com/gagan405/dedup-cache

If you happen to use it, I'd appreciate any feedback. 