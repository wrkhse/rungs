# rungs

rungs decides which steps of a plan run next. Each step waits on the outcomes of
earlier steps, and the program reports back how each one ended. rungs has no
I/O, no dependencies and no allocation (`no_std`).

## Setup

Pin the release tag in `Cargo.toml`. `v0.1.0` needs Rust 1.96 or later.

```toml
[dependencies]
rungs = { git = "https://github.com/wrkhse/rungs", tag = "v0.1.0" }
```

## Example

A program checks something with a fast judge, and asks a surer judge only when
the fast one is unsure or fails. A caller could model a cheap model followed by
GLiDE this way. Each judge has two outcomes: 0 is sure, 1 is unsure.

```rust
use rungs::{Need, Outcome, Plan, State, Step};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let unsure_or_failed = Need::outcome(0, [Outcome(1), Outcome::FAILED]);

    let plan = Plan::new([
        Step::new(2),
        Step::new(2).after([unsure_or_failed]),
    ])?;

    let mut run = plan.start();
    assert_eq!(run.advance(), [0]);

    run.settle(0, Outcome(1))?;
    assert_eq!(run.advance(), [1]);

    run.settle(1, Outcome(0))?;
    assert_eq!(run.advance(), []);
    assert_eq!(run.state(), State::Over { unhandled: None });

    let resumed = plan.resume(run.progress())?;
    assert_eq!(resumed.state(), State::Over { unhandled: None });

    Ok(())
}
```

The fast judge settles unsure, so `advance` starts the surer one. Had it
settled sure, `advance` would skip step 1 and the run would be over.

## Using it

A plan holds up to 16 steps. rungs knows steps only by position and outcomes
only by number. Names, models, prompts, saving and time stay with the program.

- **Wait on earlier steps.** `Step::after` takes needs that must all hold, and
  `Step::after_any` needs of which one must hold. `Need::outcome` waits for
  given outcomes of a step; `Need::settled` waits for it to finish with any
  outcome or be skipped.
- **Run the plan.** Call `advance` for the steps to start, then `settle` each
  with its outcome. `state` says whether the run goes on.
- **Retry a step.** `Step::attempts` sets how many times a step may run.
  `fail` returns `Failed::Retry` while attempts remain, then settles the step
  `Outcome::FAILED` and returns `Failed::Final`.
- **Restart or skip.** `restart` sends a step and every step that depends on it
  back to pending. `skip` passes over a pending step for a reason the program
  knows, such as a deadline.
- **Save and resume.** `progress` returns plain `Copy` data to store.
  `Plan::resume` refuses a progress the plan could not have reached.

[atelier](https://github.com/wrkhse/atelier) depends on `v0.1.0`.

## More

- [Contributing](CONTRIBUTING.md): the checks CI runs and what the Kani proofs
  cover.
- [The plan, rungs#1](https://github.com/wrkhse/rungs/issues/1).
- [The API](src/lib.rs), documented in the source; `cargo doc --open` renders
  it.

## License

MIT
