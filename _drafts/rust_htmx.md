---
title: "Trying HTMX in a Rust application"
tags:
  - rust
  - htmx
---

## Why I wanted to test building an htmx-base application

The last fullstack application I built used a [Svelte](https://svelte.dev/)
client (SvelteKit with the
[static adapter](https://svelte.dev/docs/kit/adapter-static) to be precise)
exchanging data with a JSON-based API written in Rust. They live in the same
repository and can be deployed simultaneously. Both were tightly coupled: there
was no intent to expose and maintain a public API from the Rust application
beside the Svelte client.

Still, you kinda have to maintain an implicit private API when serializing and
deserializing data between the two. You can easily have type-safety on both
sides (with [serde](https://serde.rs/) in Rust and [zod](https://zod.dev/) in
Svelte) but this leads to **code duplication** and feels like an unnecessary
impedance mismatch. When your application grows you have to invest in some
costly and brittle end-to-end testing if you want to catch discrepancy between
both sides of the API.

To circumvent this, one can use a fullstack framework like
[SvelteKit](https://svelte.dev/docs/kit/introduction), but it forces you to use
javascript on your backend which you might not want and/or can.

In this context, [htmx](https://htmx.org/) appealed to me as it's a
language-agnostic way of building an web application on top of HTML: all you
have to serve to your clients are plain HTML files with some custom htmx
attributes. Instead of serializing data for you client, you can instead send
HTML fragment that can replace part of a page to give the same interactivity you
would have using a javascript framework.

The implicit private API disappears as you are directly building your htmx
fragment from your business types, and so the need for end-to-end testing to
catch API discrepancies (more on that later).

<!-- In this application, most user interactions where persisted on the backend and
client-only interactions were limited to some basic show/hide elements of
navigation. -->

## Goals of this proof of concept/toy project

In no particular order, these were the points I wanted to explore and test with
this proof of concept:

- how to integrate an htxm templating/view layer in a rust application without
  introducing coupling with the business logic/layer,
- how to expose a JSON-based API alongside the htmx-based application, without
  duplicating code,
- how to integrate [taillwindcss](https://tailwindcss.com/) and
  [daisyui](https://daisyui.com/),
- how to do websocket with htmx,
- what testing strategies to use
- overall developer experience (DX) compared to popular js-based framework (hot
  reload in particular)

The proof of concept consists in a basic CRUD todo application and a live chat
to test the WebSocket part. You can find the repository with the final code
[here](https://github.com/thomas-god/poc-rust-htmx).

## Some technical decisions

- [Available templating crates](#available-templating-crates)
- [Vendoring htmx.min.js and tailwind css](#vendoring-htmxminjs-and-tailwind-css)
- [Serving a JSON API alongside htmx](#serving-a-json-api-alongside-htmx)
- [WebSocket + htmx](#websocket--htmx)
- [Testing strategy](#testing-strategy)

### Templating crate

While you could generate your htmx fragments by hand using standard string
formatting, it is
[advised to use a templating engine](https://htmx.org/essays/web-security-basics-with-htmx/#always-use-an-auto-escaping-template-engine)
to prevent common XSS attacks.

From the available [options](https://www.arewewebyet.org/topics/templating/) I
chose [maud](https://maud.lambda.xyz/) as it seems to have the best ergonomics
regarding type safety. You can easily define component-like fragments for
reusability.

```rust
pub fn todos_list_view(todos: &[Todo]) -> Markup {
    html! {
        ul.list.bg-base-100.rounded-box.shadow-md.m-6 id="todos-list" {
            @for todo in todos {
                (todo_view(todo)) /// Separate a single todo view from the list of all todos
            }
        }
    }
}

pub fn todo_view(todo: &Todo) -> Markup {
    /// Fragment for a single todo
}
```

### Vendoring htmx.min.js and Tailwind CSS

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

### Serving a JSON API alongside htmx

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

The [Web Socket extension](https://htmx.org/extensions/ws/) is pretty
straightforward to use and what I though would be one of the hardest point of
the PoC was eventless.

### Testing strategy

Testing was probably one of the less ergonomic part of using htmx. Since your
application only generates HTML fragments, writing tests for those fragments is
limited to basic behavior like "is there the expected number of elements in this
list ?", "does this button point to the correct action/URL ?", etc.

```rust
#[test]
fn test_view_done_todo() {
    let todo = Todo {
        content: "done todo".to_owned(),
        done: true,
        id: 0,
    };
    let fragment = Html::parse_fragment(&todo_view(&todo).into_string());
    let selector = Selector::parse("span").unwrap();

    let span = fragment
        .select(&selector)
        .find(|el| el.inner_html() == "done todo")
        .expect("span should be present");

    // Basic assertion on CSS class
    assert!(span.value().attr("class").unwrap().contains("line-through"));
}
```

If you want to test more complex behavior you pretty fast have to write
end-to-end tests, even for behavior scoped to a single component. To write those
tests I used the [fantoccini](https://github.com/jonhoo/fantoccini) crate that
allows you programmatically interact and run assertions with a headless web
browser.

Testing was the biggest letdown of using htmx, especially compared to the
excellent options available in the javascript world like
[testing-library](https://testing-library.com/).

## Some final thoughts

### A word on locality
