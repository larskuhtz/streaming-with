streaming-with
==============

[![Hackage](https://img.shields.io/hackage/v/streaming-with.svg)](https://hackage.haskell.org/package/streaming-with) [![Build Status](https://travis-ci.org/haskell-streaming/streaming-with.svg)](https://travis-ci.org/haskell-streaming/streaming-with)

> with/bracket-style idioms for use with [streaming]

This library provides resource management for the [streaming]
ecosystem of libraries using bracketed continuations.

[streaming]: http://hackage.haskell.org/package/streaming

Currently, these only contain file-handling utilities; if you can
think of any more functions that fit in here please let me know!

There are two ways of using this library:

1. Explicitly pass around the continuations using `Streaming.With`.

2. If you have a lot of nested continuations, you may prefer using
   `Streaming.With.Lifted` with either [`ContT`] or [managed]; these
   will allow you to pass around the parameters to the continuations.

[`ContT`]: http://hackage.haskell.org/packages/archive/transformers/latest/doc/html/Control-Monad-Trans-Cont.html#v:ContT
[managed]: http://hackage.haskell.org/package/managed

Motivation
----------

The streaming library has some usages of `MonadResource` from the
[resourcet] package to try and perform resource allocation; in
comparison to `conduit` however, streaming doesn't support prompt
finalisation.  Furthermore, because in may ways `Conduit`s can be
considered to be a datatype representing monadic functions between two
types whereas a single `Stream` is more akin to a producing-only
`Conduit` or `Pipe`, attempts at having something like `readFile` that
returns a `Stream` does not always lead to the resource being closed
at the correct point.  The consensus in the issue for [promptness for
streaming] was that perhaps relying upon `MonadResource` was not the
correct approach.

[resourcet]: http://hackage.haskell.org/package/resourcet
[promptness for streaming]: https://github.com/michaelt/streaming/issues/23


The [Bracket pattern] (also known as the _with..._ idiom, or a subset
of continuation passing style) allows for a convenient way to obtain
and then guarantee the release of (possibly scarce) resources.  This
can be further enhanced when dealing with nested usages of these with
the use of either the [`ContT`] managed transformer or the [managed]
library (where `Managed` -- or at least its safe variant -- is
isomorphic to `ContT () IO`).

[Bracket pattern]: https://wiki.haskell.org/Bracket_pattern

The biggest downside of using `bracket` from the standard `base`
library is that the types are not very convenient in the world of
monad transformers: it is limited to `IO` computations only, which
means for our use of streaming that we could only use `Stream f IO r`
without any state, logging, etc.  However, the [exceptions] library
contains a more general purpose variant that provides us with extra
flexibility; it doesn't even need to be in `IO` if you have a
completely pure `Stream`! (A variant is also available in
[lifted-base], but it is less used and streaming already depends upon
exceptions).

[exceptions]: http://hackage.haskell.org/package/exceptions
[lifted-base]: http://hackage.haskell.org/package/lifted-base

Disadvantages
-------------

Whilst the bracket pattern is powerful, it does have some downsides of
which you should be aware (specifically compared to [resourcet] which
is the main alternative).

First of all, independent of its usage with streaming, is that whilst
`bracket` predates [resourcet], the latter is [more powerful].
Whether this extra power is of use to you is up to you, but it does
mean that you in effect have lots of nested resource management rather
than just one overall resource control.

[more powerful]: http://www.yesodweb.com/blog/2013/03/resourcet-overview

The obvious disadvantage of using the bracket pattern is that it does
not fit in as nicely in the function composition style that usage of
streaming enables compared to other stream processing libraries.  This
can be mitigated somewhat with using the lifted variants in this
package which allows you to operate monadically (which still isn't as
nice but may be preferable to lots of explicitly nested continuations).

Furthermore, without prompt finalisation the same "long running
computation" [issue] is relevant.  For example, consider something
that looks like this:

```haskell
withBinaryFile "myFile" ReadMode $
  doSomethingWithEachLine
  . B.lines
  . B.hGetContents
```

Ideally, after the last chunk from the file is read, the file handle
would be closed.  However, that is not the case: it's not until the
_entire computation_ is complete that the handle is closed.  Note,
however, that the same limitation is present when using
`MonadResource`: this is a limitation of the `Stream` type, not on how
we choose to allocate and manage resources.

[issue]: http://www.yesodweb.com/blog/2013/10/core-flaw-pipes-conduit
