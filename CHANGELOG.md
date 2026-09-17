# Changelog

All notable changes to backoff-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `bodecide` — the load-bearing interface. A decision is a pure
  function of four arguments the caller supplies: the attempt count,
  the elapsed time it measured, its own verdict on the error, and a
  uniform random number. Nothing sleeps and nothing is read from the
  world, so a failure log replays into exactly the delays that
  happened, and the party that owns the timer — an async runtime, a
  thread, a hardware timer, a test that advances a number — owns the
  pause.
- `bopolicy` — the policy as a value, in four shapes, with jitter as
  part of the value rather than an extra. `check` finds every fault
  that a line running once can produce, because a policy that is wrong
  this way retries instantly forever or gives up before it waits, and
  both look like a network problem from the outside.
- `bosched` — the policy written out: the delays in order, their
  total, the attempt bound and the worst case. What a policy DOES
  cannot be read off its constructor, and a reviewer should not have to
  do the arithmetic.
- `bofault` — six faults, each naming the field and the number written
  in it.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  backoff-nv.<module>.<fn>`.
- **Whether an error is worth retrying is the caller's question.**
  `bodecide.next` takes a `Bool`. A 503 is retryable and a 400 is not,
  and that is a fact about a protocol this package does not know.
- **No retry loop, and no decorator.** A loop needs to sleep and to
  call the caller's work; both are effects. The loop is five lines in
  the caller, around `bodecide.next`.
- **No circuit breaker.** A breaker is a shared state machine across
  calls, with its own half-open probe. It is a neighbouring package,
  not a policy field.
