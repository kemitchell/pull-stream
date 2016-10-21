# pull-stream

Minimal, pipeable, pull streams

In [classic Node.js streams](https://github.com/nodejs/node-v0.x-archive/blob/v0.8/doc/api/stream.markdown),
streams _push_ data to the next stream in pipelines.
In [new Node.js streams](https://github.com/nodejs/node-v0.x-archive/blob/v0.10/doc/api/stream.markdown),
destination streams _pull_ data out of source streams.
`pull-stream` is a minimal take on streams that pull data.
`pull-stream`s work great for "object" streams as well as streams of raw text or binary data.

[![build status](https://secure.travis-ci.org/pull-stream/pull-stream.png)](https://travis-ci.org/pull-stream/pull-stream)

## Quick Example

Stat some files:

```js
var pull = require('pull-stream')
var fs = require('fs')

pull(
  pull.values(['file1', 'file2', 'file3']),
  pull.asyncMap(fs.stat),
  pull.collect(function (err, array) {
    console.log(array)
  })
)
```

Note that `pull(a, b, c)` is basically the same as `a.pipe(b).pipe(c)`.
It's a lot like [`pump`](https://www.npmjs.com/package/pump).

To grok how `pull-stream`s work, read through [pull-streams by example](https://github.com/dominictarr/pull-stream-examples)

## How do I do X with pull-streams?

There is a module for that!

Check the [pull-stream FAQ](https://github.com/pull-stream/pull-stream-faq)
and post an issue if you have a question that is not on that.

## Compatibly with Node Streams

pull-streams are not _directly_ compatible with node streams,
but you can convert a `pull-stream` into a Node stream with
[pull-stream-to-stream](https://github.com/pull-stream/pull-stream-to-stream)
and convert a Node stream a pull-stream with
[stream-to-pull-stream](https://github.com/pull-stream/stream-to-pull-stream).
These modules preserve correct back pressure.

### Readable & Reader vs. Readable & Writable

The fundamental Node streams are Readable streams and Writable streams.
The fundamental `pull-stream`s are Source streams and Sink streams.
Through streams are Sinks that return Sources.
They work a lot like Node Transform streams.

This package contains a number of useful Sources, Sinks, and Throughs.
For more information, see:
* [Sources](./docs/sources/index.md)
* [Throughs](./docs/throughs/index.md)
* [Sinks](./docs/sinks/index.md)

### Sources

As Source is just a `function source (end, cb)`.
A source may be called many times,
and will call `cb(null, data)` asynchronously, once for each call.

To signify an end state, a Source calls `cb(err)` or `cb(true)`.
A Source *must* ignore data when indicating a terminal state.

A Source *must not* be called until the previous call has called back,
unless the call is to abort the stream with `source(truthy, cb)`.

```js
// Create a Source of `n` random numbers.
function createRandomSource (count) {
  return function source (end, cb) {
    if (end) {
      return cb(end)
    } else if (count === 0) {
      return cb(true)
    } else {
      count--
      return cb(null, Math.random())
    }
  }
}
```

### Sinks

A Sink is just a `function sink (source)`.  A Sink calls a Source
until either the Sink decides to stop reading or the Source ends with
`cb(err)` or `cb(true)`.

All [Throughs](./docs/throughs/index.md)
and [Sinks](./docs/sinks/index.md)
are Sinks.

```js
//read source and log it.
function createConsoleLogSink () {
  return function consoleLogSink (source) {
    source(null, function next (end, data) {
      if(end === true) return
      if(end) throw end

      console.log(data)
      source(null, next)
    })
  }
}
```

Since these are just functions, you can pass them to each other!

```js
var randomSource = createRandomSource(100)
var logSink = createConsoleLogSink()

logSink(randomSource) // "pipe" the streams.
```

But, it's easier to read if you use's pull-stream's `pull` method:

```js
var pull = require('pull-stream')

pull(createRandomSource(100), createConsoleLogSink())
```

### Throughs

Throughs read data on one end and write data on the other.
In other words, Throughs are Sinks that returns Sources,
or `function sink (source)` that return `function source (end, cb)`.

```js
function createMappingThrough (map) {
  return function sink (source) {
    // Return a Source!
    return function source (end, cb) {
      source(end, function (end, data) {
        cb(end, data !== null ? map(data) : null)
      })
    }
  }
}

pull(
  source,
  createMappingThrough(JSON.parse),
  sink
)
```

### Piping

To process data, a `pull-stream` pipeline must go from a Source to a
Sink. Between the Source and Sink, data can flow through any number
of Throughs.

Data will not flow through a `pull-stream` pipeline until the whole
pipeline is connected.

```js
pull(source, firstThrough, secondThrough, sink)
```

Sometimes, it's simplest to describe a stream in terms of other
streams. If call `pull` with only Through arguments, `pull` return will
return a new Through.

```js
// Combine two Throughs into a new Through that processes
// newline-delimited JSON.
var parseNDJSON = pull(
  buffersToLinesThrough,
  parseJSONThrough
)

// Uses the new Through to process data.
pull(source, parseNDJSON, sink)
```

`pull` decides what kind of `pull-stream` its arguments are by
argument count, or arity. Sources have two arguments, `end` and `cb`,
and Sinks and Throughs have one argument, `source`.

## Duplexes

Duplex streams are used to for communication, where messages pass
both ways.  `pull-stream` Duplexes are objects with Source and Sink
properties like `{source: source, sink: sink}`.

Pipe Duplexes like this:

``` js
var a = duplex()
var b = duplex()

// Pull from `a` to `b`, and from `b` to `a`.
pull(a, b, a)

// or

pull(a.source, b.sink)
pull(b.source, a.sink)

// or

b.sink(a.source); a.sink(b.source)
```

## Design Goals & Rationale

There is a deeper,
[platonic abstraction](http://en.wikipedia.org/wiki/Platonic_idealism),
where a stream is just an array in time, instead of in space.
All the various streaming "abstractions",
[classic Node streams](https://github.com/joyent/node/blob/v0.8.16/doc/api/stream.markdown),
[new Node streams](https://github.com/joyent/node/blob/v0.10/doc/api/stream.markdown), and
[reducers](https://github.com/Gozala/reducers),
are just crude implementations of this abstract idea.

`pull-streams` try to realize all the best features of these
implementations as simply as possible.

### Type Agnostic

A stream abstraction should be able to handle both streams of text and streams
of objects.

### A pipeline is also a stream.

Something like this should work: `a.pipe(x.pipe(y).pipe(z)).pipe(b)`
This makes it possible to write a custom stream simply by
combining a few available streams.

### Propagate end and error conditions.

If a stream ends in an unexpected way (error),
then other streams in the pipeline should be notified.
(This is a problem in node streams - when an error occurs,
the stream is disconnected, and the user must handle that specially.)

Also, the stream should be able to be ended from either end.

### Make backpressure and laziness transparent.

Very simple transform streams must be able to transfer back pressure
instantly.

This is a problem in Node streams. Pause is only transferred on write, so
on a long chain (`a.pipe(b).pipe(c)`), if `c` pauses, `b` will have to write to it
to pause, and then `a` will have to write to `b` to pause.
If `b` only transforms `a`'s output, then `a` will have to write to `b` twice to
find out that `c` is paused.

[reducers](https://github.com/Gozala/reducers) reducers has an interesting method,
where synchronous transformations propagate back pressure instantly!

This means you can have two "smart" streams doing I/O at the ends, and lots of dumb
streams in the middle, and back pressure will work perfectly, as if the dumb streams
are not there.

This makes laziness work right.

### Handle ends, errors, and aborts throughout.

In pull streams, any part of the stream (source, sink, or through)
may terminate the stream. (This is the case with Node streams too,
but it's not handled well.)

#### source: end, error

A source may end (`cb(true)` after read) or error (`cb(error)` after read)
After ending, the source *must* never `cb(null, data)`

#### sink: abort

Sinks do not normally end the stream, but if they decide they do
not need any more data they may "abort" the source by calling `read(true, cb)`.
An abort (`read(true, cb)`) may be called before a preceding read call
has called back.

### How to handle ends, aborts, and errors in Throughs.

Rules for implementing `read` in a through stream:

1. Sink wants to stop. sink aborts the through

   just forward the exact read() call to your source,
   any future read calls should cb(true).

2. We want to stop. (abort from the middle of the stream)

   abort your source, and then cb(true) to tell the sink we have ended.
   If the source errored during abort, end the sink by cb read with `cb(err)`.
   (this will be an ordinary end/error for the sink)

3. Source wants to stop. (`read(null, cb) -> cb(err||true)`)

   forward that exact callback towards the sink chain,
   we must respond to any future read calls with `cb(err||true)`.

   In none of the above cases data is flowing!

4. If data is flowing (normal operation:   `read(null, cb) -> cb(null, data)`

   forward data downstream (towards the Sink)
   do none of the above!

There either is data flowing (4) OR you have the error/abort cases (1-3), never both.

## 1:1 read-callback ratio

A pull stream source (and thus transform) returns *exactly one value* per read.

This differs from Node streams, which can use `this.push(value)` and an internal
buffer to create transforms that write many values from a single read value.

pull-streams don't come with their own buffering mechanism, but [there are ways
to get around this](https://github.com/dominictarr/pull-stream-examples/blob/master/buffering.js).

## Minimal bundle

If you need only the `pull` function from this package you can reduce the size
of the imported code (for instance to reduce a Browserify bundle) by requiring
it directly:

```js
var pull = require('pull-stream/pull')

pull(random(), logger())
```

## Further Examples

- [dominictarr/pull-stream-examples](https://github.com/dominictarr/pull-stream-examples)
- [./docs/examples](./docs/examples.md)

Explore this repo further for more information about
[sources](./docs/sources/index.md),
[throughs](./docs/throughs/index.md),
[sinks](./docs/sinks/index.md), and
[glossary](./docs/glossary.md).

## License

MIT
