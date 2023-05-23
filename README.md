# history-throttled

This is a drop-in replacement for
[`history.replaceState`](https://developer.mozilla.org/en-US/docs/Web/API/History/replaceState)
and
[`history.pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState),
with appropriate throttling applied to avoid browser errors.

## What's the problem?

If you call `history.replaceState` too often, you may get one of the following errors:

* Safari: "SecurityError: Attempt to use history.replaceState() more than 100
  times per 30 seconds"
* Chrome: "Throttling navigation to prevent the browser from hanging. See
  https://crbug.com/1038223. Command line switch
  --disable-ipc-flooding-protection can be used to bypass the protection"
* Firefox: "Too many calls to Location or History APIs within a short timeframe."

You could catch and ignore these errors, but once browsers hit the rate limit,
they disable all calls to `replaceState` for a while.

## Features

* Tiny: 0.4 KB min-gzipped with no dependencies
* Smart: prioritizes `pushState` over `replaceState` when rate limited
* Browser-aware: applies different throttling to Safari (310 ms) than other
  browsers (52 ms)
* Compatible: works in any modern browser, and can be imported from Node

## Installation

```
npm install --save history-throttled
```

## Usage

Replace all your calls to `history.pushState` and `history.replaceState` and all
assignments to `location.hash` as follows:

```js
import { pushState, replaceState } from "history-throttled";

pushState("", "", "/foo"); // instead of history.pushState("", "", "/foo")
replaceState("", "", "/bar"); // instead of history.replaceState("", "", "/bar")
replaceState("", "", "#baz"); // instead of location.hash = "baz"
```

Even if you only care about `replaceState` throttling, you should still replace
all calls to `history.pushState` with the throttled version:

* Browsers put both functions on the same timer, so `history.pushState` can fail
  if you call `replaceState` a lot.
* The throttled `pushState` version prevents delayed `replacedState` calls from
  being executed out-of-order after `pushState`, which would result in a wrong
  URL state.

## Behavior in detail

When you call `replaceState` or `pushState` more often than every 310
milliseconds (or 52 milliseconds on non-Safari browsers), calls will be
automatically throttled.

`pushState` calls will get priority over `replaceState` calls. Say you're making
the following calls in quick succession:

```js
pushState("", "", "/a");
pushState("", "", "/b");
pushState("", "", "/c");
replaceState("", "", "/c/1");
replaceState("", "", "/c/2");
```

This will result in the following behavior:

Immediately:

```js
// We're not yet rate limited when the first pushState call occurs, so
// it will be executed synchronously.
history.pushState("", "", "/a");
```

After 310 milliseconds:

```js
// We run the most recent pushState call, because it takes priority over
// the subsequent replaceState calls. Note that the intermediate pushState
// call to "/b" is dropped, because it exceeds the allowed rate.
history.pushState("", "", "/c");
```

After 620 milliseconds:

```js
// Finally, the most recent of any remaining replaceState calls is flushed.
history.replaceState("", "", "/c/2");
```

## Node compatibility

The package can safely be imported from Node, for example for server-side
rendering, as long you don't call `pushState` or `replaceState`.

## Testing

To disable all throttling and synchronously pass all calls through to the
`history` object, run the following before any calls to `pushState` or
`replaceState`:

```js
import { setDelay } from "history-throttled";

setDelay(0);
```

## About

Copyright 2023 Jo Liss, licensed under the [Apache License, Version
2.0](https://www.apache.org/licenses/LICENSE-2.0).

Written for use in the [calcu.net](https://www.calcu.net/) calculator.
