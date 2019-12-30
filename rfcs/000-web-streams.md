# Web Streams Interoperability

## Summary

This is an API to convert between [web streams][spec-streams]
and their Rust counterparts from [the `futures` crate][docs-futures].
More specifically, it allows bi-directional conversions between:
* A web [`ReadableStream`][spec-readable-stream] and a Rust [`Stream`][docs-futures-stream]
* A web [`WritableStream`][spec-writable-stream] and a Rust [`Sink`][docs-futures-sink]

## Motivation

Currently, it is already possible to convert between a JavaScript `Promise` and a Rust `Future`
using [`wasm-bindgen-futures`][docs-wasm-bindgen-futures].
This enables Rust code to call promise-based web APIs such as [`fetch()`][mdn-fetch],
or to expose a promise-based web API to JavaScript.
However, Rust code cannot yet access stream-based web APIs such as [`Response.body`][mdn-body]
or [the upcoming `CompressionStream` and `DecompressionStream` APIs][compression-streams].

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
[The exploratory `wasm-streams` project](https://github.com/MattiasBuelens/wasm-streams/blob/fe0ded7e2e9442464aa999513ba688c31470de7c/src/readable/sys.rs)
has more complete definitions for both `ReadableStream` and `WritableStream`, which can serve as a starting point.

Note that, once the streams specification [switches to WebIDL][gh-streams-963], these low-level bindings could be
auto-generated as part of the `web-sys` crate.

### Mid-level API

The mid-level API wraps the low-level API, but makes it a bit more friendly to use.

First, low-level methods that return a `js_sys::Promise` are instead provided as `async` methods with a proper
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

The mid-level API could also add a generic type parameter for the type of chunks produced by a `ReadableStream`
or consumed by a `WritableStream`. Since most readable streams always produce the same type of chunks, and
most writable streams always consume the same type of chunks, it could be nice if this type was associated with the stream.
This way, developers don't have to manually convert from/to a `JsValue` every time they read from or write to a stream.
```rust
struct JsReadableStream<T> { /* ... */ }
struct JsReadableStreamDefaultReader<'a, T> { /* ... */ }

impl<'a, T: From<JsValue>> JsReadableStreamDefaultReader<'a, T> {
    pub async fn read(&mut self) -> Result<Option<T>, JsValue> { /* ... */ }
}
```

### Converting to Rust Stream and Sink

To convert a web `ReadableStream` into a Rust `Stream`, we add an `into_stream()` method on `JsReadableStreamDefaultReader`:
```rust
impl<'a, T: From<JsValue>> JsReadableStreamDefaultReader<'a, T> {
    pub fn into_stream(self) -> IntoStream<'a, T> {
        IntoStream::new(self)
    }
}
```
The `IntoStream<'a, T>` struct implements the `Stream<Item = Result<T, JsValue>>` trait, by repeatedly calling
`JsReadableStreamDefaultReader::read()` and awaiting its result until the stream errors or closes.
A possible implementation is available as part of
[the exploratory `wasm-streams` project](https://github.com/MattiasBuelens/wasm-streams/blob/fe0ded7e2e9442464aa999513ba688c31470de7c/src/readable/into_stream.rs).

Similarly, to convert a web `WritableStream` into a Rust `Sink`, we add an `into_sink()` method on `JsWritableStreamDefaultWriter`,
which returns a struct that implements `Sink<T, Error = JsValue>`.
An example implementation is available [here](https://github.com/MattiasBuelens/wasm-streams/blob/fe0ded7e2e9442464aa999513ba688c31470de7c/src/writable/into_sink.rs).

Alternatively, instead of putting `into_stream()` on `JsReadableStreamDefaultReader`, we could put it on `JsReadableStream` instead.
This would be nicer to use, but is more difficult to implement.
The `IntoStream` struct must take ownership of the entire `JsReadableStream`, so the lifetime `'a` in
`JsReadableStreamDefaultReader<'a, T>` remains valid. However, this requires a self-referential struct.
We could bypass this problem by using the low-level API directly, but that would lead to code duplication.

### Converting from Rust Stream and Sink

To convert a Rust `Stream` into a web `ReadableStream`, we add a `From<Stream>` implementation:
```rust
impl<T, S> From<S> for ReadableStream<T> 
where T: Into<JsValue>,
    S: Stream<Item = Result<T, JsValue>> { /* ... */ }
```

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

Since async trait methods are not yet supported in Rust, we will need to create a dedicated overload for
constructing a `ReadableStream` from our specific underlying source implementation. 
```rust
mod sys {
    #[wasm_bindgen]
    extern "C" {
        #[derive(Clone, Debug)]
        pub type ReadableStream;

        #[wasm_bindgen(constructor)]
        pub fn new_with_source(source: IntoUnderlyingSource) -> ReadableStream;
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
[mdn-fetch]: https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch
[mdn-body]: https://developer.mozilla.org/en-US/docs/Web/API/Body/body
[compression-streams]: https://wicg.github.io/compression/
[gh-streams-963]: https://github.com/whatwg/streams/issues/963
