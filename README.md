# scope
A small library with context helpers for golang.

Consists of 2 main components:

- Closer
- Group


## Closer

Closer tries to loosely fill in gap between new context-aware world and old world of context-less network
primitives with blocking operations, whose unblocking and cancellation logic is entirely different.
Cancelling such blocking operations is often achieved by `Close()`-ing corresponding primitives in another goroutine.
ContextCloser runs separate goroutine, which closes all `io.Closer`-s passed to Closer constructor func, when
its bind `context`'s `Done()` channel is closed. This separate goroutine is finished either after context is expired
and `io.Closer`-s are closed or when `Cancel()` or `Close()` method of Closer is called, if it happens earlier.
Closer must be constructed with `scope.Closer` or `scope.CloserWithErrorGroup` func.

To finish Closer's goroutine before context is expired call either `Close()` or `Cancel()`, but not both!
`Close()` closes all the `io.Closer`s passed to ContextCloser constructor func regardless of context and finishes goroutine.
`Cancel()` just finishes goroutine and doesn't close anything.
Both `Close()` and `Cancel()` close inner channel without any check, so calling them both or calling any of them more
than once leads to panic! Nevertheless, it is perfectly safe to call either of them once regardless of whether context was
already expired or not.

It is convenient to construct Closer and call either `Close()` or `Cancel()` at once in defer statement.
If `io.Closer` (e.g. a `net.Conn`) should be bound not only to `context`, but also to current func scope, then use `Close()`:

```Go
defer scope.Closer(ctx, conn).Close()
```

If `io.Closer` (e.g. a `net.Conn`) should be bound to context only while current goroutine is in the current func (rarer
case, that might break structured concurrency principle), then use `Cancel()`:

```Go
defer scope.Closer(ctx, conn).Cancel()
```


## Group

Group is basically a convenient wrapper for `errgroup.Group` and optional Closer-s.
Group must be constructed with `scope.Group` func. Any number of Closer-s can be added to it using
`Add[After]Closer[Cancelling]()` functions.

Another way to look at Group is as at `context.Context` with its `errgroup.Group`, where new child goroutines could be
run in [structured](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/) way.
