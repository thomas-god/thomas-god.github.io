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

The htmx doc explicitly [advises](https://htmx.org/docs/#download-a-copy) to
vendor the htmx source code directly within your project. This plays nicely when
using a non JS-based backend as you don't have to add another package manager
and/or build step.

If using tailwindcss/daisyui to style your application, you can use the same
strategy with the
[tailwind CLI](https://tailwindcss.com/docs/installation/tailwind-cli) (and
[daysui](https://daisyui.com/docs/install/standalone/)) to generate an
`output.css` file that contains only the class you're actually using. Though
unlike for the htmx source code, I chose not to commit this file but instead
generate it during the build process. The tailwind cli also provides hot reload
for you styles when developing.

### Content negotiation to serve json alongside htmx

We can use standard
[content negotiation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation)
to choose what representation to send back to the client (htmx or JSON) based on
the `Accept` header of the request.

```rust
pub async fn create_todo(
    State(state): State<ApiState>,
    headers: HeaderMap,
    ContentNegotiator(payload): ContentNegotiator<CreateTodoRequest>,
) -> impl IntoResponse {
    let mut state = state.write().await;
    let todo = state.todos.add_todo(&content);

    // Use the request's Accept header to determine response format
    match headers.get("Accept").and_then(|h| h.to_str().ok()) {
        Some("application/json") => Json(todo).into_response(),
        _ => todo_view(todo).into_response(), // Matching htmx template
    }
}
```

We can use the same strategy to extract data from different representation in
the request's body. For instance for a `POST /todo` route, we can either receive
the new todo content as JSON, or as a xxx-form-urlencoded. We can write an axum
extractor that abstract away this handling, and that allows having a single
handler for both.

```rust
pub struct ContentNegotiator<T>(pub T);

impl<S, T> FromRequest<S> for ContentNegotiator<T>
where
    S: Send + Sync,
    T: for<'de> Deserialize<'de> + Send,
{
    type Rejection = StatusCode;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let content_type = req
            .headers()
            .get(header::CONTENT_TYPE)
            .and_then(|v| v.to_str().ok())
            .unwrap_or("");

        // Check content type to determine how to parse
        if content_type.starts_with("application/json") {
            // Parse as JSON
            let Json(payload) = Json::<T>::from_request(req, state)
                .await
                .map_err(|_| StatusCode::BAD_REQUEST)?;

            return Ok(ContentNegotiator(payload));
        }

        // Default to form data
        let Form(payload) = Form::<T>::from_request(req, state)
            .await
            .map_err(|_| StatusCode::BAD_REQUEST)?;

        Ok(ContentNegotiator(payload))
    }
}
```

An alternative to content negotiation would be to maintain one route per
representation (`api/json/todo` and `/todo` for instance).

### WebSocket + htmx

### Testing strategy

### Changes on how to think when building a client/web page/application

## Some final thoughts
