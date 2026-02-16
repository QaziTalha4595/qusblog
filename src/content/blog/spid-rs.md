---
title: spid-rs. Why I Built Yet Another Web Crawler (and Why It’s in Rust)
author: Quazi Talha
pubDatetime: 2026-02-16T06:06:31Z
slug: spid-rs
featured: true
draft: false
tags:
  - Rust
  - Web
  - Development
description: High-performance, concurrent website crawler and mirroring tool written in Rust.
---

# Why I Built Yet Another Web Crawler (and Why It’s in Rust)

Last week, I needed to mirror a small documentation site for offline reading. I tried HTTrack, but it was a disaster. Not only did the output lack any proper formatting, but it also started downloading the entirety of GitHub just because the documentation had a single outbound link. Something that would easily be solved if the program told me before hand how many pages it was going to download.

So, I did what any developer with a free afternoon and a copy of the Rust book does: I wrote my own.

Enter **spid-rs**.

## The "Perfect" Crawler Problem

Most crawlers fall into two traps. Software like HTTrack is made for large scale downloading of websites, so of course it downloads everything greedily—flooding the server and your RAM—or they’re too timid like `wget`, which misses half the assets because they aren't "strictly" links.

I wanted something different. I wanted a tool that was:

1. **Predictable**: I want to know how many files I'm downloading _before_ I start filling my disk.
2. **Robust**: If a single image 404s, the whole process shouldn't roll over and die.
3. **Idiomatic**: It should leverage what makes modern systems programming great.

## The Architecture of Certainty

One of the first "opinionated" decisions I made was splitting the process into two distinct phases: **Mapping** and **Downloading**.

Most crawlers crawl and download simultaneously. It’s efficient, sure, but it’s a nightmare for UX. You have no idea if you’re 5% or 95% done. By separating discovery into a "Phase 1," `spid-rs` builds a complete manifest of the site. Only when we know exactly what we're dealing with do we start the "Phase 2" download.

```rust
// Phase 1: Mapping
state.discovered.insert(base_url.to_string());
map_frontier(base_url, state.clone(), pb_map.clone(), args.concurrency).await?;

// Phase 2: Downloading
let total = state.discovered.len() as u64;
let pb_down = mp.add(ProgressBar::new(total));
download_all(state, pb_down, args.concurrency).await?;
```

Is it a bit slower? Maybe. Does it provide a pixel-perfect progress bar and an actual ETA? Absolutely. As a developer, I’ll trade 5% performance for 100% visibility any day.

## Concurrency without Chaos

Rust’s `tokio` and `futures` ecosystem is powerful, but it’s easy to get wrong.

Early in the project, I used a manual `Semaphore` and `FuturesUnordered`. It worked, but it felt... manual. I ended up switching to `stream::iter(...).buffer_unordered(N)`. It’s a more elegant way to say "I want to do a lot of things at once, but please don't open 10,000 sockets simultaneously."

```rust
let mut stream = stream::iter(current_batch)
    .map(|(url, depth)| {
        let state = state.clone();
        let pb = pb.clone();
        async move { (depth, discover_links(url, depth, state, pb).await) }
    })
    .buffer_unordered(concurrency);
```

We also used `DashMap` and `DashSet` for shared state. If you’re coming from a world of Mutex-wrapped HashMaps, the "interior mutability" of DashMap feels like magic. It minimized lock contention and kept the code clean.

## The Boring Stuff That Actually Matters: URL Normalization

You’d think a URL is a URL. It’s not.
Between fragments (`#section`), varying ports (`:443`), and trailing slashes, a single page can look like ten different resources to a naive crawler.

I spent an embarrassing amount of time on `normalize_url`.

```rust
fn normalize_url(mut url: Url) -> Url {
    url.set_fragment(None);
    if let Some(port) = url.port() {
        if (url.scheme() == "http" && port == 80) || (url.scheme() == "https" && port == 443) {
            let _ = url.set_port(None);
        }
    }
    if url.path().is_empty() {
        url.set_path("/");
    }
    url
}
```

If you don't strip the fragment and handle query strings properly, you end up with a mess. For `spid-rs`, I decided to hash query strings into the filename. It keeps the files safe for the OS while ensuring `search?q=rust` and `search?q=java` don't overwrite each other.

```rust
if let Some(query) = url.query() {
    let mut hasher = Sha256::new();
    hasher.update(query);
    let hash = hex::encode(hasher.finalize());
    let mut name = path_buf.file_name().unwrap_or_default().to_os_string();
    name.push("-q-");
    name.push(&hash[..8]);
    path_buf.set_file_name(name);
}
```

## Lessons Learned

Building `spid-rs` reminded me that "simple" software is rarely simple. Handling the edge cases of the web—the redirects, the weird mime-types, the `mailto:` links that aren't actually files—is where the real work is.

If you’re looking to build something similar, my advice is this: **Focus on the data structure first.** Once I moved from a loose collection of strings to a centralized `SharedState` with precompiled selectors, the performance didn't just improve; the bugs started fixing themselves.

Rust isn't just about speed. It’s about building a mental model of your data that the compiler actually enforces.

You can check out the source code for [spid-rs here](https://github.com/QaziTalha4595/spid-rs). It’s opinionated, it’s concurrent, and it actually works.
