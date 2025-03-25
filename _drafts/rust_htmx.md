---
title: "Trying HTMX in a Rust application"
tags:
  - rust
  - frontend
---

## Why I wanted to test building an htmx-based application

The last fullstack application I built uses a [Svelte](https://svelte.dev/)
client (SvelteKit with the
[static adapter](https://svelte.dev/docs/kit/adapter-static) to be precise) that
exchanges data with a JSON-based API written in Rust. They live in the same
repository and can be deployed simultaneously.

Both were tightly coupled: there was no intent to expose and maintain a public
API from the Rust application, but you still have to maintain an implicit
private API when serializing and deserializing data between the two. You can
easily have type-safety on both sides (with [serde](https://serde.rs/) in Rust
and [zod](https://zod.dev/) in Svelte/JavaScript) but this leads to **code
duplication** and feels like an unnecessary **impedance mismatch** (i.e. dealing
with different data models like representing Rust's enums in JavaScript).

When your application grows you have to invest in some costly and brittle
end-to-end testing if you want to catch discrepancies between both sides of this
private API. Another alternative would be to use a fullstack framework like
[SvelteKit](https://svelte.dev/docs/kit/introduction), but it forces you to use
JavaScript on your backend which you might not want or can.

In this context, [htmx](https://htmx.org/) appealed to me as it's a
language-agnostic way of building a web application on top of HTML: all you have
to serve to your clients are plain HTML pages with some custom htmx attributes
on your HTML elements. Instead of serializing data for your client, you can
instead send HTML fragments that can replace part of a page to give the same
interactivity without full page reloads you would have using a JavaScript
framework like React or Svelte.

The implicit private API disappears as you directly build your htmx pages and
fragments from within your backend code, with strong type-safety. This removes
the need for end-to-end testing solely to catch API discrepancies (but more on
end-to-end testing later).

As with all things, there are some tradeoffs as htmx cannot emulate all
client-side interactions you would easily do with a JavaScript frontend
framework, but it is commonly claimed that it can cover most websites
interactions.

## Goals of this proof of concept/toy project

These were the points I wanted to explore and test with this proof of concept:

- How to integrate an htmx templating/view layer in a rust application without
  introducing coupling with the business logic,
- How to expose a JSON-based API alongside the htmx-based application, without
  duplicating code,
- How to integrate [tailwindcss](https://tailwindcss.com/) and
  [daisyui](https://daisyui.com/),
- How to do websocket with htmx,
- What testing strategies to use
- Overall developer experience (DX) compared to popular JavaScript-based
  framework (hot reload in particular)

The proof of concept consists in a basic CRUD todo application and a live chat
to test the WebSocket part. You can find the repository with the final code
[here](https://github.com/thomas-god/poc-rust-htmx).

## Some technical decisions

This sections lists in no particular order the technical decisions I made and
their eventual tradeoffs.

- [Templating crates](#templating-crate)
- [Vendoring htmx.min.js and tailwind css](#vendoring-htmxminjs-and-tailwind-css)
- [Developer experience and hot-reloading](#developer-experience-and-hot-reloading)
- [Serving a JSON API alongside htmx](#serving-a-json-api-alongside-htmx)
- [WebSocket + htmx](#websocket--htmx)
- [Testing strategy](#testing-strategy)
- [A word on locality](#a-word-on-locality)

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

### Developer experience and hot-reloading

To have some form of hot-reloading I used two commands:

- the tailwind CLI:
  `npx @tailwindcss/cli -i ./input.css -o ./assets/style/output.css --watch` to
  watch for change in the CSS of the application,
- [bacon CLI](https://dystroy.org/bacon/) `bacon run-long` to recompile and run
  the application on code (and template) change. But this does not propagate the
  reload to connected clients that you have to reload yourself.

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
    let todo = state.todos.add_todo(payload);

    // Use the request's Accept header to determine response format
    match headers.get("Accept").and_then(|h| h.to_str().ok()) {
        Some("application/json") => Json(todo).into_response(),
        _ => todo_view(todo).into_response(), // Matching htmx template
    }
}
```

We can use the same strategy to extract data from different representation in
the request's body. For instance for a `POST /todo` route, we can either receive
the new todo content as `application/json`, or as a
`application/xxx-form-urlencoded` and write an axum extractor that abstract away
this handling in order to have a single handler for both.

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

I thought the Web Socket part would be one of the hardest point of the PoC but
it was actually pretty straightforward thanks to the
[Web Socket extension](https://htmx.org/extensions/ws/) since once the
connection is established it is just regular htmx syntax.

You can refer to the actual implementation
[here](https://github.com/thomas-god/poc-rust-htmx/blob/main/src/chat/handlers.rs)
for more details.

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

If you want to test more complex behavior you quickly have to write end-to-end
tests, even for behavior scoped to a single component. To write those tests I
used the [fantoccini](https://github.com/jonhoo/fantoccini) crate that allows
you programmatically interact and run assertions with a headless web browser.

Testing was the biggest letdown of using htmx, especially compared to the
excellent options available in the javascript world like
[testing-library](https://testing-library.com/). I don't think it is due to a
lack of tooling, but rather is inherent to how htmx works and the inability to
have a component in isolation (more on that on the following section on
locality).

### A word on locality

One recurring argument when reading about htmx is the concept of
[locality of behavior](https://htmx.org/essays/locality-of-behaviour/) that
roughly states that the closer a component's elements (view, styling and
behavior) are to each other, the easier it is to understand and maintain it.
Think HTML, CSS and JavaScript in a single file rather than in dedicated
separated files.

The canonical example, taken from the
[htmx docs](https://htmx.org/essays/locality-of-behaviour/), shows that the
following button is easy to understand as its behavior is colocated within the
`<button>` element: clicking the button will make a `GET` request to
``/clicked`.

```html
<button hx-get="/clicked">Click Me</button>
```

While I do agree that htmx favors locality at the component/fragment level, I
found that it is less true when you start assembling fragments into bigger
component or pages. Because of how htmx works, you often have to reference other
elements or fragments from a fragment.

In the following example we want to append the fragment returned by the call to
`POST /todo` to the element with the ID `todos-list` located elsewhere in the
page.

```rust
pub fn todo_form() -> Markup {
    html!(
        div {
            form
                hx-post="/todo"
                hx-target="#todos-list"
                hx-swap="beforeend" {
                /// Actual form content
            }
        }
    )
}
```

We could pass the values for `hx-target` and `hx-swap` to `todo_form()` to
improve the reusability of this particular component, but this does not remove
the necessary coupling between this component and its surroundings.

If this complexity is manageable for small components, I found that it scales
poorly and very fast. I think it is inherent to the way htmx is intended to
work, as there is no integration layer or component you would normally used to
localize this coupling between components (think molecules and organisms in
[atomic design](https://atomicdesign.bradfrost.com/chapter-2/)).

## Some final thoughts

htmx is a valuable tool to have in your tool belt. It can be enough for simple
applications (think a form and some content that updates) and can avoid the
unnecessary complexity that comes with using a JavaScript framework. That being
said, this PoC made me realize the ceiling of htmx is lower than I expected, at
least for the application I typically build. The poor testing experience and the
coupling between fragments/components are deal breakers for me, and I still
prefer the compromise of using a dedicated JavaScript-based frontend I outlined
in the introduction.
