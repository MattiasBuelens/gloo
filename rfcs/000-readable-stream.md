# Readable Stream Interoperability

## Summary

This RFC introduces an API to interact with a web [ReadableStream][spec-readable-stream], and to allow bi-directional
conversions between a web ReadableStream and a Rust [Stream][docs-futures-stream].

This API can be extended in the future to also support a web [WritableStream][spec-writable-stream], and conversions
with it and a Rust [Sink][docs-futures-sink]. However, since browser support for writable streams is quite limited,
this is left for a future RFC.

## Motivation

Currently, it is already possible to convert between a JavaScript `Promise` and a Rust `Future`
using [wasm-bindgen-futures][docs-wasm-bindgen-futures].
This enables Rust code to call promise-based web APIs such as [fetch][mdn-fetch],
or to expose a promise-based web API to JavaScript.
However, Rust code cannot yet access stream-based web APIs such as [Response.body][mdn-body]
or [Compression Streams][compression-streams].

With this interoperability API, it would become possible to:
* process the body of an HTTP response while it is being received (using `Response.body`)
* compress the body of an HTTP request while it is being uploaded (using `Request.body` and `CompressionStream`)

## The API

### Low-level API

First, we provide a low-level API to interact directly with a `ReadableStream` or `WritableStream`.
We can use `wasm-bindgen` to generate the glue code for all classes and methods defined in [the specification][spec-streams].

As an example, the low-level bindings for `ReadableStream` could look something like this:
```rust
mod sys {
    #[wasm_bindgen]
    extern "C" {
        #[derive(Clone, Debug)]
        pub type ReadableStream;

        #[wasm_bindgen(constructor)]
        pub fn new() -> ReadableStream;

        #[wasm_bindgen(method, getter, js_name = locked)]
        pub fn is_locked(this: &ReadableStream) -> bool;

        #[wasm_bindgen(method, catch, js_name = getReader)]
        pub fn get_reader(this: &ReadableStream) -> Result<ReadableStreamDefaultReader, JsValue>;
    }

    #[wasm_bindgen]
    extern "C" {
        #[derive(Clone, Debug)]
        pub type ReadableStreamDefaultReader;

        #[wasm_bindgen(method, getter, js_name = closed)]
        pub fn closed(this: &ReadableStreamDefaultReader) -> Promise;
    
        #[wasm_bindgen(method, js_name = cancel)]
        pub fn cancel(this: &ReadableStreamDefaultReader) -> Promise;
    
        #[wasm_bindgen(method, js_name = cancel)]
        pub fn cancel_with_reason(this: &ReadableStreamDefaultReader, reason: &JsValue) -> Promise;
    
        #[wasm_bindgen(method, js_name = read)]
        pub fn read(this: &ReadableStreamDefaultReader) -> Promise;
    
        #[wasm_bindgen(method, catch, js_name = releaseLock)]
        pub fn release_lock(this: &ReadableStreamDefaultReader) -> Result<(), JsValue>;
    }
}
```
[The exploratory wasm-streams project](https://github.com/MattiasBuelens/wasm-streams/blob/7a34a038f17f010903327126df1062902b7a19c2/src/readable/sys.rs)
has more complete definitions, which can serve as a starting point.

Note that, once the streams specification [switches to WebIDL][gh-streams-963], these low-level bindings could be
auto-generated as part of [the web-sys crate][docs-web-sys].

### Mid-level API

#### Overview

The mid-level API wraps the low-level API, but makes it a bit more friendly to use.
It also provides utilities to convert between Rust and JavaScript streams.

```rust
impl JsReadableStream {
    pub fn from_raw(raw: sys::ReadableStream) -> Self { /* ... */ }

    pub fn as_raw(&self) -> &sys::ReadableStream { /* ... */ }

    pub fn into_raw(self) -> sys::ReadableStream { /* ... */ }

    pub fn is_locked(&self) -> bool { /* ... */ }

    pub async fn cancel(&mut self) -> Result<(), JsValue> { /* ... */ }

    pub async fn cancel_with_reason(&mut self, reason: &JsValue) -> Result<(), JsValue> { /* ... */ }

    pub fn get_reader(&mut self) -> Result<JsReadableStreamDefaultReader<'_>, JsValue> { /* ... */ }
}

// Convert from Rust Stream to JavaScript ReadableStream
impl From<Box<dyn Stream<Item = Result<JsValue, JsValue>>>> for JsReadableStream { /* ... */ }

impl<'stream> JsReadableStreamDefaultReader<'stream> {
    pub async fn closed(&self) -> Result<(), JsValue> { /* ... */ }

    pub async fn cancel(&mut self) -> Result<(), JsValue> { /* ... */ }

    pub async fn cancel_with_reason(&mut self, reason: &JsValue) -> Result<(), JsValue> { /* ... */ }

    pub async fn read(&mut self) -> Result<Option<JsValue>, JsValue> { /* ... */ }

    pub fn release_lock(&mut self) -> Result<(), JsValue> { /* ... */ }

    // Convert from JavaScript ReadableStream to Rust Stream
    pub fn into_stream(self) -> IntoStream<'stream> { /* ... */ }
}
```

#### Futures instead of raw promises

Low-level methods that return a `js_sys::Promise` are instead provided as `async` methods with a proper
return type, using `JsFuture` from `wasm-bindgen-futures`.
For example:
```rust
impl JsReadableStream {
    pub async fn cancel_with_reason(&mut self, reason: &JsValue) -> Result<(), JsValue> {
        let js_value = JsFuture::from(self.as_raw().cancel_with_reason(reason)).await?;
        debug_assert!(js_value.is_undefined());
        Ok(())
    }
}
```

#### Mutable borrow to lock the stream

Second, acquiring a reader or writer ["locks" the stream][spec-lock-stream], which prevents it from being used
by another reader/writer or by another `cancel()`/`abort()` call.
In Rust, we can make this behavior more explicit by keeping a mutable borrow on the reader or writer:
```rust
impl JsReadableStream {
    pub fn get_reader(&mut self) -> JsReadableStreamDefaultReader<'_> { /* ... */ }
}

struct JsReadableStreamDefaultReader<'a> {
    raw: sys::ReadableStreamDefaultReader,
    _parent: PhantomData<&'a mut JsReadableStream>,
}
```
This way, we can verify *at compile-time* that at most one reader or writer is active at any given time.

#### Generic chunk type

The mid-level API could also add a generic type parameter for the type of chunks produced by a `ReadableStream`.
Since most readable streams always produce the same type of chunks, and, it could be nice if this type were associated 
with the stream. This way, developers don't have to manually convert from/to a `JsValue` every time they read from the stream.
```rust
struct JsReadableStream<T> { /* ... */ }
struct JsReadableStreamDefaultReader<'a, T> { /* ... */ }

impl<'a, T: From<JsValue>> JsReadableStreamDefaultReader<'a, T> {
    pub async fn read(&mut self) -> Result<Option<T>, JsValue> { /* ... */ }
}
```
However, this has not yet been successfully implemented in the wasm-streams project.

Alternatively, users could first convert their Rust `Stream<Item = T>` into a `Stream<Item = Result<JsValue, JsValue>>`
using [StreamExt.map][docs-streamext-map], and then convert that stream into a JavaScript `ReadableStream` (see below).

### Converting to Rust Stream

To convert a web `ReadableStream` into a Rust `Stream`, we add an `into_stream()` method on `JsReadableStreamDefaultReader`:
```rust
impl<'a> JsReadableStreamDefaultReader<'a> {
    pub fn into_stream(self) -> IntoStream<'a> {
        IntoStream::new(self)
    }
}
```
The `IntoStream<'a>` struct implements the `Stream<Item = Result<JsValue, JsValue>>` trait, by repeatedly calling
`JsReadableStreamDefaultReader::read()` and awaiting its result until the stream errors or closes.
A possible implementation is available as part of
[the exploratory `wasm-streams` project](https://github.com/MattiasBuelens/wasm-streams/blob/7a34a038f17f010903327126df1062902b7a19c2/src/readable/into_stream.rs).

Alternatively, instead of putting `into_stream()` on `JsReadableStreamDefaultReader`, we could put it on 
`JsReadableStream` instead. This would be nicer to use, but is more difficult to implement.
The `IntoStream` struct must take ownership of the entire `JsReadableStream`, so the lifetime `'a` in
`JsReadableStreamDefaultReader<'a>` remains valid. However, this requires a self-referential struct.
We could bypass this problem by using the low-level API directly, but that would lead to code duplication.

### Converting from Rust Stream

To convert a Rust `Stream` into a web `ReadableStream`, we add a `From<Stream>` implementation. More specifically:
```rust
impl From<Box<dyn Stream<Item = Result<JsValue, JsValue>>>> for JsReadableStream { /* ... */ }
```
The stream needs to be boxed, since we need to be able to read items from it whenever JavaScript requests it.

For the actual implementation of this method, we first need to create a `IntoUnderlyingSource` struct which implements
[the underlying source API][spec-underlying-source].
```rust
struct IntoUnderlyingSource {
    stream: Stream<Item = Result<JsValue, JsValue>>
}

impl IntoUnderlyingSource {
    pub async fn pull(controller: sys::ReadableStreamDefaultController) -> Result<(), JsValue> { /* ... */ }
    pub async fn cancel(reason: JsValue) -> Result<(), JsValue> { /* ... */ }
}
```

We will need to create a dedicated (private) overload for constructing a `ReadableStream` from our specific underlying 
source implementation. In the future, once Rust supports async trait methods, we could create an `UnderlyingSource`
trait, accept a `Box<dyn UnderlyingSource>` (or similar) and make this public.
```rust
mod sys {
    #[wasm_bindgen]
    extern "C" {
        #[derive(Clone, Debug)]
        pub type ReadableStream;

        #[wasm_bindgen(constructor)]
        pub(crate) fn new_with_source(source: IntoUnderlyingSource) -> ReadableStream;
    }
}
```

[spec-streams]: https://streams.spec.whatwg.org/
[spec-readable-stream]: https://streams.spec.whatwg.org/#rs-class
[spec-writable-stream]: https://streams.spec.whatwg.org/#ws-class
[spec-underlying-source]: https://streams.spec.whatwg.org/#underlying-source-api
[spec-lock-stream]: https://streams.spec.whatwg.org/#lock
[docs-futures]: https://docs.rs/futures/0.3.1/futures/index.html
[docs-futures-stream]: https://docs.rs/futures/0.3.1/futures/stream/trait.Stream.html 
[docs-futures-sink]: https://docs.rs/futures/0.3.1/futures/sink/trait.Sink.html 
[docs-wasm-bindgen-futures]: https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/
[docs-web-sys]: https://docs.rs/web-sys/0.3.39/web_sys/
[docs-streamext-map]: https://docs.rs/futures/0.3.5/futures/stream/trait.StreamExt.html#method.map
[mdn-fetch]: https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch
[mdn-body]: https://developer.mozilla.org/en-US/docs/Web/API/Body/body
[compression-streams]: https://wicg.github.io/compression/
[gh-streams-963]: https://github.com/whatwg/streams/issues/963
