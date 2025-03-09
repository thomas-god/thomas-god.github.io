---
title: "Using HTMX in a Rust application"
tags:
  - rust
  - htmx
---

- Why trying HTMX over traditional frontend frameworks, or even vanilla JS
- Goals of this PoC: how to integrate the view layer with the rest of a rust
  application, testing strategies, working with websockets, using
  tailwindcss/daisyui, etc.
- Some difficulties and pain points
  - choosing a templating crate
  - no (node) build toolchain: vendoring htmx.min.js and generating css with
    tailwind cli
  - serving htmx content alongside a json api
  - unit testing htmx templates, e2e testing with fantoccini
- Some final thoughts

## Why i wanted to test an htmx-base application

Last fullstack application i've built: felt that using a dedicated frontend
framework introduce some impedance mismatch. Despite the frontend being coupled
to the backend, you have to serialize and deserialize data between the two, and
thus maintain an api contract between the two. Even if with tools like zod it's
not that difficult to have type-safe parsing, this duplication feels overhead.

This also means you also have to maintain two separate subprojects, with
dedicated build tool chain, deployment considerations, etc.

For this specific application, all user interactions client-side are persisted
on the backend, there is very little client-only interactions beside some basic
show/hide elements or navigation. Also there is no intent to have a public
facing API beside the frontend application.

## Goals of this proof of concept/toy project

- how to integrate an htxm templating/view layer in a rust application without
  introducing coupling with the business logic/layer
- how to expose a JSON-based API alongside the htmx-based application, without
  duplicating code
- using tailwindcss/daisyui in the generated html
- doing websocket with htmx
- what testing strategies to use
- overall dx compared to popular js-based framework (hot reload in particular)

Basic todos application and chat (for the websocket part).

## Lessons learned

- [Available templating crates](#available-templating-crates)
- [Vendoring htmx.min.js and tailwind css](#vendoring-htmxminjs-and-tailwind-css)
- [Content negotiation to serve json alongside htmx](#content-negotiation-to-serve-json-alongside-htmx)
- [WebSocket + htmx](#websocket--htmx)
- [Testing strategy](#testing-strategy)
- [Changes on how to think when building a client/web page/application](#changes-on-how-to-think-when-building-a-clientweb-pageapplication)

### Available templating crates

### Vendoring htmx.min.js and tailwind css

### Content negotiation to serve json alongside htmx

### WebSocket + htmx

### Testing strategy

### Changes on how to think when building a client/web page/application

## Some final thoughts
