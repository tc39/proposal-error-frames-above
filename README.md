# ECMAScript Proposal: Error option `framesAbove`

## Status

Champion: Ruben Bridgewater

Author: Ruben Bridgewater <ruben@bridgewater.de>

TODO: Define a stack frame is implementation defined.

Stage: 0

## Overview

This proposal introduces a new `Error` options property, `framesAbove`, that allows authors to exclude specific leading stack frames when an error's stack trace is captured.

- `framesAbove` receives a method (i.e., a callable function). If the provided value is neither a method nor `undefined`, the constructor throws a `TypeError`.
- When the stack trace for the error is collected, all frames above the topmost call to the provided function — including that call — are omitted from the resulting stack trace.
- Frames omitted due to `framesAbove` do not count towards any defined stack trace limit (e.g., `Error.stackTraceLimit` in V8).

This capability mirrors long-standing host-specific facilities (for example, the `constructorOpt` parameter in Node.js/V8's `Error.captureStackTrace`) while providing a standardized, cross-platform option available on all ECMAScript `Error` subclass constructors.
Next to the easier to use API and better stack limit handling, it is also using less CPU cycles due to only evaluating the stack frames once instead of twice.

## Proposed API

```js
new Error(message, { framesAbove: someMethod })
new TypeError(message, { framesAbove: someMethod })
new RangeError(message, { framesAbove: someMethod })
new AggregateError(iterable, message, { framesAbove: someMethod })
// ... applies to all built-in Error constructors that accept an options bag
```

### Validation

- If `options` is provided and has a `framesAbove` property whose value is not `undefined` and not callable, throw a `TypeError`.
- If `framesAbove` is `undefined` or absent, the default behavior is unchanged.

## Motivation

Applications often create small wrapper utilities around user code. Those wrappers show up at the top of stack traces and provide little value to the developer debugging the failure. Authors currently rely on host-specific APIs or post-processing to remove these frames.

`framesAbove` standardizes this behavior in the language:

- Improves signal-to-noise in error stacks.
- Provides deterministic behavior across engines.
- Avoids reliance on non-standard APIs or brittle regex post-processing.

## Semantics

- Let F be the value of `options.framesAbove`.
  - If F is `undefined`, do nothing special.
  - Else if F is not callable, throw a `TypeError`.
  - Else, record F with the newly created error instance for use during stack collection.
- When collecting the stack for the error:
  - Find the topmost occurrence of a frame whose associated function is F, scanning from the most-recent frame down the stack.
  - Omit all frames strictly above that occurrence, as well as the occurrence itself.
  - Apply any stack trace limit to the remaining frames only. Frames omitted due to `framesAbove` do not count towards the limit.
- If F does not appear in the captured stack, the stack is unchanged (aside from any unrelated platform behavior and stack limits).
- If F appears multiple times (e.g., recursion), the topmost occurrence is used.

Note: The exact mapping from a frame to an associated function is host-defined today. This proposal uses the same association that engines already use for stack trace frames.

## Examples

### Basic wrapper removal

```js
function wrapper(fn, ...args) {
  return fn(...args);
}

function doWork() {
  throw new Error('boom', { framesAbove: wrapper });
}

// Without framesAbove, the stack might start with:
// Error: boom
//   at doWork (...)
//   at wrapper (...)
//   at main (...)
//
// With framesAbove: wrapper, the stack starts at the first frame below wrapper:
// Error: boom
//   at doWork (...)
//   at main (...)
```

### Multiple occurrences (topmost wins)

```js
function trampoline(fn) { return fn(); }
function inner() { throw new Error('x', { framesAbove: trampoline }); }
function mid() { return trampoline(inner); }
function outer() { return trampoline(mid); }
outer();
// All frames above the topmost trampoline call (including that call) are omitted.
```

### Interaction with stack trace limits

```js
Error.stackTraceLimit = 2;
function helper() { throw new Error('x', { framesAbove: helper }); }
helper();
// The omitted helper frame does not consume the limit.
// The remaining 2 frames are taken from below helper.
```

## Detailed semantics

- `framesAbove` is an option recognized by all standard `Error` subclass constructors that accept an options bag.
- The value associated with `framesAbove`:
  - Must be a callable function, or `undefined`.
  - Is consulted only for the error on which it is provided; it has no global effect.
  - Does not change the message, name, or cause semantics.
- Omitted frames:
  - Are not present in the produced stack string.
  - Do not count towards any stack length limit that the engine applies.
  - Are conceptually removed prior to applying any limit or formatting.

## Relationship to existing host behavior

Many hosts already provide capabilities that remove frames (for example, V8/Node's `Error.captureStackTrace(target, constructorOpt)`). `framesAbove` provides standardized language-level behavior with similar intent:

- Works across environments that do not expose host-specific APIs.
- Provides consistent semantics for "topmost call" and "inclusive omission".

Hosts may continue to provide additional facilities; those remain orthogonal to `framesAbove`.

## Specification text (outline)

This section sketches normative changes; precise wording and integration points are provided in the accompanying spec text.

1. In each `Error` (and subclass) constructor that accepts an options argument:
   - Let `options` be the second (or third, for `AggregateError`) argument.
   - If `options` is not `undefined`:
     - Let `framesAbove` be ? Get(options, "framesAbove").
     - If `framesAbove` is not `undefined` and IsCallable(`framesAbove`) is `false`, throw a `TypeError`.
2. During error instance initialization, record `framesAbove` (which may be `undefined`) in the error's internal error data slot for later use during stack capture.
3. When capturing the stack for an error instance E:
   - Let `S` be the sequence of frames that would be captured for E absent this option.
   - If E.[[ErrorData]].[[FramesAboveFunction]] is a function F:
     - Let `k` be the smallest index in `S` (counting from the most-recent frame at index 0) such that `S[k]`'s associated function is F.
     - If such `k` exists, replace `S` with `S` sliced from index `k + 1` to the end.
   - Apply any stack length limit to `S`.
   - Produce the stack string from `S` as usual.

See the full spec draft:

- [spec.emu](./spec.emu)

## Non-goals and edge cases

- This proposal does not standardize stack string formatting, source map application, or async stack joining.
- If the provided function never appears in the captured stack, nothing is removed.
- If the provided value is callable but exotic (e.g., a Proxy-wrapped function), engines use their existing notion of frame-to-function association.
- The option is per-error and does not affect other errors.

## Alternatives considered

- Continued reliance on host-specific APIs (`Error.captureStackTrace`) or post-processing. Rejected for lack of portability to other JS engines, difficulty of API use, and CPU overhead related.
- Accepting a frame index or sentinel instead of a function. Rejected due to brittleness and lack of composability across call paths.

## TC39 stages and champions

- Ready for Stage 1

## Conclusion

`framesAbove` provides a simple, explicit way to remove unhelpful wrapper frames from error stacks, improving developer ergonomics while aligning with existing host practices. By standardizing this behavior, the ecosystem gains a portable, deterministic tool for producing higher-signal stack traces.
