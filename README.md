# backoff-nv

When a request to a service fails, a client that retries immediately
turns one failure into many. **Backoff** is the practice of waiting
longer before each retry, and **jitter** is the practice of spreading
those waits randomly so that clients which failed together do not
return together. This package holds the policy that decides both, as a
value. The reference implementations are the Python libraries
[backoff](https://github.com/litl/backoff) and
[tenacity](https://tenacity.readthedocs.io/), and the AWS Architecture
Blog's
[Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/),
which is where the jitter forms below were measured.

## What a retry policy is

A retry policy answers two questions: how long to wait before the next
attempt, and when to stop trying. The wait is a **delay**, computed
from the number of attempts already made.

| Shape | Delay for attempt *n* | What it is for |
| --- | --- | --- |
| Constant | The base delay | A dependency whose recovery time is known |
| Linear | Base plus a step per attempt | A slow, predictable queue |
| Exponential | Base multiplied by a factor per attempt | Almost every network client |
| Decorrelated | Between the base and three times the *last* delay | Many clients that must spread out fast |

The delay for attempt *n* before any jitter is the **computed delay**.
Exponential backoff alone still has every client retrying at the same
moment, because every client computes the same delay. Jitter is what
breaks that up, and it is a property of the policy rather than an
extra.

| Jitter | The wait |
| --- | --- |
| None | Exactly the computed delay |
| Full | Uniformly between zero and the computed delay |
| Equal | Half the computed delay, plus uniformly up to the other half |
| Decorrelated | Uniformly between the base delay and the computed delay |

Stopping has three reasons. The **attempt limit** counts tries. The
**budget** limits the total time the sequence may span, which is what a
caller with a deadline actually has. And the error itself may not be
worth retrying — a malformed request will fail the same way forever.
The first two are the policy's; the third is the caller's, and this
package takes the caller's answer as an argument.

**Nothing here sleeps.** Every function answers a number of
milliseconds. The caller does the waiting, with whatever it already has
— an async runtime, a thread, a hardware timer, or a test that advances
a counter.

## Install

```
novo pkg add backoff-nv
```

## Example

```novo
use bopolicy
use bodecide
use bosched

fn main() [io]
    // The policy, built once. Full jitter comes with `exponential`.
    let policy = bopolicy.with_budget(
                     bopolicy.with_max_attempts(
                         bopolicy.exponential(100, 2.0), 5), 10000)

    // Check it where it is built: every fault it can have is decided here.
    match bopolicy.check(policy)
        Some(f) => println("unusable policy: ${f.message()}")
        None    => println(bosched.describe(policy))

    // After an attempt failed. `true` is the caller's verdict on the
    // error, `0.42` a uniform random number it drew, and the state
    // carries what it measured.
    let state = bodecide.record(bodecide.start(), 100, 130)
    match bodecide.next(policy, state, true, 0.42)
        BoRetryAfter(ms) => println("wait ${ms} ms, then try again")
        BoGiveUp(reason) => println("stop: ${bodecide.reason_name(reason)}")
```

## What the package contains

| Module | Contents |
| --- | --- |
| `bofault` | The six ways a policy can be unusable, each naming the field. |
| `bopolicy` | The policy value: the four shapes, the four jitters, the limits and the check. |
| `bodecide` | The decision after a failed attempt, the state a caller carries between attempts, and the delay arithmetic. |
| `bosched` | The whole sequence a policy implies: the delays, their total, the attempt bound and the worst case. |

## How to choose an entry point

**`bodecide.next` is the one call a retry loop makes.** It takes the
policy, the state, the caller's verdict on the error and a random
number, and answers either a delay or a reason to stop. The loop is:
make an attempt; when it fails, call `next`; wait the delay it
answers; call `bodecide.record` with that delay and the elapsed time;
try again.

**`bodecide.may_retry` asks only the policy's half** — the attempt
limit and the budget — without a random number or a verdict. Use it
before doing the work of classifying an error.

**`bosched.delays` is for a test and a start-up log.** It writes out
what the policy would do if every attempt failed.

**`bopolicy.check` runs once, at start-up.** Every fault it finds is
decided by a line that runs once.

## The rules a user needs

1. **This package never sleeps.** Every answer is a number of
   milliseconds, and the caller waits.
2. **The elapsed time is what the caller measured.**
   `bodecide.record` takes it. No function here reads a clock, so a
   replay of a failure log produces the delays that happened.
3. **The randomness comes from the caller.** `bodecide.next` takes a
   uniform number in `0.0 ..< 1.0`, and
   `bopolicy.uniforms_needed` says how many one decision consumes —
   one, or none for `BoNoJitter`.
4. **Whether an error is worth retrying is the caller's verdict.**
   `next` takes a `Bool`. A 503 is retryable, a 400 is not, and a parse
   failure never is.
5. **Use jitter.** A policy without it puts every client that failed
   together back on the wire together. `exponential` comes with full
   jitter, and `BoNoJitter` is for a test.
6. **Full jitter clears a herd fastest; equal jitter keeps a floor
   under every wait.** Take the second when the retry itself is
   expensive.
7. **A decorrelated policy needs a maximum delay**, because it grows
   from the last delay rather than from the attempt number and has no
   bound without one. `bopolicy.check` refuses one without. Its first
   delay is between the base and three times the base.
8. **Cap the delay.** Without `with_max_delay`, an exponential policy's
   tenth attempt waits minutes.
9. **A budget is not an attempt limit.** Five attempts with a ceiling
   of thirty seconds can span two minutes. `with_budget` is what a
   caller with a deadline sets, and `bosched.worst_case_millis` is what
   it checks.
10. **The attempt limit counts the first attempt.**
    `with_max_attempts(p, 3)` allows one attempt and two retries.
11. **A policy with no attempt limit and no budget retries forever.**
    That is a reasonable choice for a background worker and a mistake
    in a request path. `bosched.describe` ends with `unbounded` for it.
    A budget alone bounds the time and not the count, because a
    jittered delay can be zero, so `bosched.attempt_bound` answers `0`
    for any policy without an attempt limit.
12. **Check the policy at start-up.** A base delay of zero, a factor
    that does not grow, a ceiling below the floor: each of them looks
    like a network problem at run time, and like a named field in
    `bopolicy.check`.

## What is not included

- **Sleeping, and the retry loop.** Both need effects this package does
  not declare. The loop is five lines around `bodecide.next`.
- **A random number generator.** See rule 3. A caller that has one
  already should not get a second, and a test should be able to hand in
  the numbers it wants.
- **A clock.** See rule 2.
- **Deciding whether an error is retryable.** See rule 4.
- **A circuit breaker.** A breaker is shared state across calls, with a
  half-open probe of its own. It is a neighbouring package rather than
  a field on a policy.
- **A build for a microcontroller.** Every function is free of input
  and output, clocks and random numbers, but the policy, the state and
  the decision are heap values, and a device build admits none.
- **Rate limiting.** Backoff decides when to retry after a failure;
  a rate limiter decides whether an action may happen at all. See
  ratelimit-nv.

## Related packages

- [ratelimit-nv](https://novo-lang.org/packages/ratelimit-nv) answers
  the other question, and takes its instant from the caller the same
  way.
- [prometheus-nv](https://novo-lang.org/packages/prometheus-nv) is
  where a retry count and a give-up reason go.
  `bodecide.reason_name` is free of anything an error carried, so it is
  safe as a metric label.

## Tests

```bash
novo test tests/bodecide_tests.nv    # the decision, the state and the jitter
novo test tests/bopolicy_tests.nv    # the four shapes, the check and the schedule
novo test tests/formula_tests.nv     # the formulas over seeded uniforms
bash tests/coverage.sh               # line coverage over src/
```

The reference implementations are `backoff` and `tenacity`, whose
policies this package makes values rather than decorators, and the AWS
Architecture Blog's measurements of full, equal and decorrelated
jitter. The suite asserts that the same four arguments always produce
the same decision, that the delay grows with the attempt number and is
held by the ceiling, that the three reasons to stop are told apart,
that a decorrelated policy grows from the last delay, that full jitter
spans zero to the computed delay while equal jitter keeps a floor, and
that an unusable policy is refused where it is built.

`tests/formula_tests.nv` computes the expected delays from the
definitions rather than from the package: the shapes from the table
above, and full, equal and decorrelated jitter from the formulas the
AWS article states. The uniforms come from a linear congruential
generator with a fixed seed.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
