---
title: "My experience learning Rust (and trying embedded development too)"
tags:
  - rust
---

## Why Rust ?

Learning Rust was on my radar for quite some time, and I finally got to spend
the good part of the last 3 months learning it. For context I'm a software
engineer with 8 years of experience, spent writing backend services for the
energy and electricity industry, mostly in Python and Node/Typescript (with some
odd C++). I lean toward domain and test driven development (DDD and TDD), having
used hexagonal architecture in most of my recent project.

I was initially interested in Rust for the usual "memory safety with
performances" and the general hype over it, but recently I also felt frustrated
when using Python and Node's (lack thereof) type systems. Especially after
picking up on DDD and trying to model business entities and invariants in the
type system, it always felt like your not reaping all the benefits you should
from spending time doing it, as they're either clunky to use, or easy to
circumvent. Thus Rust's promise of a strong and built-in type system was what
prompted me to finally spend some time learning it.

As a bonus the following points were also in favor of learning Rust (together
with alternatives I considered):

- battery included, i.e. a good developer experience (DX) out of the box (as
  with Deno of Go),
- something "new" that could potentially challenge my current conception of
  designing and writing software,
- a language that could target embedded development (MicroPython and C/C++ being
  the common alternatives)

## My previous attempts

For the record I had already tried to learn Rust 2-3 times before (3 years ago
and at the beginning of 2024), mostly by reading the
[Rust book](https://doc.rust-lang.org/book/) and doing the
[rustlings exercises](https://github.com/rust-lang/rustlings).

But each time I failed to stick with the language past those resources, both
because I did not have a project to practice the it, and also I think that at
that time I did not fully grasp the benefits the language provided over regular
Python/Node, especially regarding the type system.

Nonetheless, this was not wasted time as I am sure I kept some concepts in the
back of my mind so that coming back to it was not a complete new experience.

## Learning it for real this time

As it was clearer for me this time why I wanted to learn Rust, I made sure to
have some concrete projects I could apply it after having read the
[Rust book](https://doc.rust-lang.org/book/) again.

### Embedded development using Rust

I wanted to build a simple weather station for my home with a micro-controller
in each room that would send temperature and humidity measures back to a
Raspberry Pi in my local network. This seemed like a small enough scope to
start, but in hindsight it was probably a mistake because:

- you're really trying to learn 2 things at the same time (rust and embedded
  development) which tends to dilute the effort you put in,
- the complexity of the project and its modelling potential are small and do not
  allow you to really explore what you can do with Rust's type system,
- embedded development is slower, so does your iteration and learning speed,
- you are targeting `no_std`, which both a good and a bad thing when learning I
  guess.

_Where I was_: basic syntax and control flows of the language, the ownership
system, and primitives like `Option/Result`.

### Codecrafters: _build your own redis_

In the meantime I discovered [codecrafters](https://app.codecrafters.io/catalog)
while watching a
[stream from Jon Gjenset](https://www.youtube.com/playlist?list=PLqbS7AVVErFhAhQ5s9SWcvxHh4GwsIk_d).
The concept is to build replicas of common tools you might use (git, redis, an
HTTP server) in the language of your choice. Each challenge guides you steps by
steps through the key concepts of the tools you are trying to build. To go to
the next step you have to pass integration tests that run against your code, so
it has a very nice TDD feel to it. I started the _build your own redis_
challenge.

Overall the challenge was complex enough that it made sense to spend some time
utilizing the design capabilities of Rust's type system. The challenge was long
enough that you had several opportunities to refactor your code and question
your initial approach. Especially I found the
[actor pattern](https://doc.rust-lang.org/book/ch16-02-message-passing.htmlhttps://doc.rust-lang.org/book/ch16-02-message-passing.html)
very idiomatic and easy in Rust thanks to built-in primitives like
[channel](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html). Though by
the end, the "toy project" effect was starting to hinder the motivation on
working on it.

_Where I was_: enums, synchronization primitives, testing in Rust

### A personal project: parcelec

At this stage I was confident enough that I could build something using what I
ahd learnt. I also had not done any async programming in Rust yet and had just
finished reading the [tokio](https://tokio.rs/tokio/tutorial) documentation.

I had an old project called Parcelec I started a few years ago but had not the
time finish. It is an educational game to teach people electricity balancing and
electricity markets. Each player has a bunch of different power plant (gas,
nuclear, renewable, etc.) and has to dispatch them to match the consumption of
it consumers. The game has a multiplayer dimension as there is a market in which
players can participate to buy and sell energy to help manage their balance. The
game is open enough to encourage players to ask themselves the same question an
actual utility would do.

I had a clear enough vision of the features I wanted for the MVP/reboot so that
it was manageable in a few weeks of work. Also the nature of the game fitted
well an asynchronous model as players are connected to the same market and can
dispatch their power plants at the same time. Also there was a lot of non
trivial business logic and invariant to enforce using the type system.

While on the redis project I tried to limit the use external libraries to really
grasp the language, here I had the opportunity to leverage crates like `tokio`,
`axum`, `serde`, `thiserror`, `derive_more`, and more.

_Where I was_: async Rust, trait and trait object, newtype pattern, error
handling in Rust

## Some closing thought

I am overall very pleased with the time I invested in learning Rust. The
language actually lived to its hype regarding most of what I wanted from.

What i love about the language:

- the type system (ouf, what i came for) and the powerful syntax to support it
  (match, let/if let, the ? operator)
- the overall DX: cargo, compiler error messages, clippy
- the way rust "does OOP" (no inheritence and emphasis on composition with its
  trait system)
- the overall ecosystem for embedded development (HAL, embassy)
- the quality of the resources

What i disliked during my learning or about the language:

-

### Repositories

- my basic embedded application: https://github.com/thomas-god/envirometer
- _build your own redis_: https://github.com/thomas-god/codecrafters-redis-rust
- parcelec: https://github.com/thomas-god/parcelec

### Resources I used and find useful

-
