---
title: "My experience learning Rust (in late 2024/early 2025)"
tags:
  - rust
---

Learning Rust was on my radar for quite some time, and I finally got to spend
the good part of the last 3 months learning it. I am overall very pleased with
the learning experience and using the language. This article relates the steps I
took to learn it and will hopefully spark your interest in the language !

> _Disclaimer_: this is not intended to be a step by step guide or the optimal
> way to learn the language. It mainly focuses on my subjective experience and
> will not try to go deep into some concepts of the language.

For context I am a software engineer with 8 years of experience, spent writing
backend services for the energy and electricity industry, mostly in Python and
Node/Typescript (with some odd C++). I lean toward domain and test driven
development (DDD and TDD), having used hexagonal architecture in most of my
recent projects.

## Why Rust ?

I was initially interested in Rust for the usual "memory safety without
sacrificing performance" and the general hype over it. But recently I also felt
frustrated when using Python and Node's type systems (or lack thereof).
Especially after picking up on DDD and trying to model business entities and
invariants in the type system, it always felt like you are not reaping all the
benefits you should as they are either clunky to use, or easy to circumvent.
Rust's promise of a strong and built-in type system was what ultimately prompted
me to finally dedicate time learning it.

The following points were also strong arguments toward learning Rust:

- batteries included and the good developer experience (DX) out of the box (as
  with Deno or Go),
- something "new" that could potentially challenge my current conception of
  designing and writing software,
- a language that could also target embedded development (MicroPython and C/C++
  being the common alternatives)

## My previous attempts

For the record I had already tried to learn Rust 2-3 times before (at the
beginning of 2024 and 3 years ago), mostly by reading the
[Rust book](https://doc.rust-lang.org/book/) and doing the
[rustlings exercises](https://github.com/rust-lang/rustlings).

But each time I failed to stick with the language past those resources, both
because I did not have a project to practice it, and also because I think that
at that time I did not fully grasp the benefits the language provided over
regular Python/Node, especially regarding the type system. So it was hard to
justify spending more time on it for benefits that did not seem obvious.

Nonetheless, this was not wasted time as I am sure I kept some concepts in the
back of my head so that coming back to it this time was not a complete new
experience.

## Learning it for real this time

As this time it was clearer for me why I wanted to learn Rust, I made sure to
have some concrete projects I could apply it after having read the
[Rust book](https://doc.rust-lang.org/book/) again.

> _Where I was_: mostly theoretical knowledge from reading documentation.

### Embedded development using Rust

I started building a simple weather station for my home with a micro-controller
in each room that would relay temperature and humidity measures back to a
Raspberry Pi in my local network. It seemed like a small enough scope to start
with immediate benefits, but in hindsight it was probably a mistake because:

- you are really trying to learn 2 things at the same time (Rust and embedded
  development) which dilutes the effort and time you put in,
- the complexity of the project and its modelling potential are small and do not
  allow you to really explore what you can do with Rust's type system (and it is
  not performance sensitive enough to justify using Rust over MicroPython),
- embedded development iteration speed is slower compared to traditional
  software development, and your learning speed slows accordingly,
- you are targeting `no_std`, which can be argued is both a good and a bad thing
  I guess.

> _Where I was_: able to write a program using basic syntax, control flow and
> primitives like `Option/Result`, started to grasp the ownership system.

### Codecrafters: _build your own redis_

In the meantime I discovered [Codecrafters](https://app.codecrafters.io/catalog)
while watching a
[stream from Jon Gjenset](https://www.youtube.com/playlist?list=PLqbS7AVVErFhAhQ5s9SWcvxHh4GwsIk_d).
The pitch of Codecrafters is to learn by building replicas of common tools you
might use (git, Redis, an HTTP server, etc.) in the language of your choice.
Each challenge guides you step by step through the key concepts of the tool you
are trying to build. To go to the next step you have to pass integration tests
that run against your code at each commit, so it has a very nice TDD feel to it.
I started the
[_build your own redis_](https://app.codecrafters.io/courses/redis/overview)
challenge (just because it was the free challenge of the month).

Overall the challenge was complex enough that it made sense to spend time
utilizing the design capabilities of Rust's type system. The length of challenge
meant that you had several opportunities to refactor your code and challenge
your approach. Especially I found the
[actor pattern](https://doc.rust-lang.org/book/ch16-02-message-passing.html)
very idiomatic and easy to use in Rust thanks to built-in primitives like
[channel](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html). Though by
the end the "toy project" effect was starting to hinder the motivation to work
on it.

> _Where I was_: able to write more complex programs leveraging enums and
> synchronization primitives, how to test Rust code.

### A personal project: Parcelec

At this stage I was confident enough that I could build something using what I
had learnt. I also had not done any async programming in Rust yet and had just
finished reading the [tokio](https://tokio.rs/tokio/tutorial) documentation.

I had an old project called Parcelec I started a few years ago but lacked the
time to finish it. It is an educational game (in the browser) to introduce
people to electricity balancing and electricity markets. Each player owns a
bunch of different power plants (gas, nuclear, renewable, etc.) and has to
dispatch them to match the consumption of its consumers. The game has a
multiplayer dimension as there is a market in which players can buy and sell
energy to help manage their balance. The game is designed to be open enough to
encourage players to ask themselves the same questions an actual utility would.

I had a clear enough vision of the features I wanted for the MVP/reboot so that
it was manageable in a few weeks of work. Also the nature of the game fitted
well an asynchronous model as players are connected to the same real-time market
and can dispatch their power plants at the same time. There was also a lot of
non-trivial business logic and invariant to enforce using the type system.

While on the Redis project I tried to limit the use external libraries to really
grasp the language, here I had the opportunity to leverage common crates like
`tokio`, `axum`, `serde`, `thiserror`, `derive_more`, and more.

The experience was an overall success as I managed to finish the MVP in 4 weeks,
and quickly felt productive using Rust.

> _Where I was_: able to build an async Rust application, using traits and trait
> objects to reduce coupling, using the newtype pattern to model your business
> domain, error handling in Rust.

## Some closing thought

I am overall very pleased with the time I invested in learning Rust. The
language actually lived to its hype regarding most of what I expected from it.

### Some pain points I encountered

I clearly had to unlearn things coming from dynamically typed languages that do
dynamic dispatch by default. The main one is trying to overuse trait objects and
`dyn` to mimic what I would have done in those languages, while most of the time
you _know_ the concrete variants you expect and thus a Rust enum and static
dispatch can do the job without introducing excess complexity.

Crates documentation felt a bit austere at first, but once you learn how to use
it the fact that it is standardized across the entire Rust ecosystem feels very
ergonomic, both to _read_ and to _write_.

### My opinion on Rust's learning curve

Rust has a reputation of being hard to learn, but after spending a few hundred
hours using it I would say this difficulty is true but mostly comes from the
time it takes, rather than any intrinsic difficulty. Sure there is some advanced
concepts that will require coming back a few times to fully grasp them, but the
resources are usually very good at explaining the design motivations of a given
feature and the corresponding nuances. In my opinion this makes the learning
experience _predictable_: yes the learning curve is long, but it is mostly
regular, thus it is easy to stay motivated, provided you have concrete
project(s) to practice it.

> _What I have not touched yet_: writing your own macros, advanced use of
> lifetimes, using unsafe code, using a lot of smart pointers.

### Repositories

- my basic embedded application: https://github.com/thomas-god/envirometer
- _build your own redis_: https://github.com/thomas-god/codecrafters-redis-rust
- parcelec: https://github.com/thomas-god/parcelec

### Resources I used and find useful

- the go-to resources to start learning rust: the
  [Rust book](https://doc.rust-lang.org/book/) and the
  [rustlings exercises](https://github.com/rust-lang/rustlings).
- _Rust is beyond OOP_, a nice series of articles on how Rust compared to OOP as
  done in other languages and the misconceptions you might have as a result:
  https://www.thecodedmessage.com/tags/beyond-oop/.
- _How to code it_: the
  [newtype pattern](https://www.howtocodeit.com/articles/ultimate-guide-rust-newtypes),
  [hexagonal architecture in Rust](https://www.howtocodeit.com/articles/master-hexagonal-architecture-rust),
  and
  [error handling](https://www.howtocodeit.com/articles/the-definitive-guide-to-rust-error-handling)
  are comprehensive articles on key topics to go to the next steps in Rust.
- A good article about the type state pattern :
  https://cliffle.com/blog/rust-typestate/.
- _Rust for rustaceans_ by Jon Gjengset (https://rust-for-rustaceans.com/) and
  most of his [streams](https://www.youtube.com/@jonhoo) are a good entrypoint
  to "advanced" Rust.
- If you want to learn by doing:
  [Codecrafters](https://app.codecrafters.io/catalog) and
  [fly.io distributed challenges](https://fly.io/dist-sys/).
