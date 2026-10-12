# System Design — LLD / Machine Coding

Hand-written low-level design exercises, one subproject per problem. These are plain
Java: no Spring, no web server, no database. Each problem is self-contained, keeps its
own package namespace, and can be built and run on its own.

| Problem | What it exercises | Run it |
|---|---|---|
| [`splitwise/`](splitwise) | Splitting expenses across a group, with pluggable split rules (equal, exact, percent) and a running balance sheet | `./gradlew :splitwise:test` |
| [`tictactoe/`](tictactoe) | A board game engine separated from its rule engine, with a simple AI opponent | `./gradlew :tictactoe:run --console=plain` |
| [`elevator/`](elevator) | A LOOK-style elevator scheduler and a pluggable multi-car dispatcher | `./gradlew :elevator:run` |

Requires a JDK 21 toolchain. Gradle comes with the wrapper, so `./gradlew` is enough.

```bash
./gradlew build        # compile everything and run all tests
./gradlew :elevator:run
```

## splitwise

Models users, groups and expenses, and keeps a balance sheet of who owes whom. The
interesting part is the split rule: `EqualSplit`, `ExactSplit` and `PercentSplit` all
implement a common `Split` interface, so adding a new way to divide an expense means
adding one class rather than editing the service. `SplitCalculatorTest` covers the
arithmetic for each rule.

## tictactoe

Separates three things that are easy to tangle together: the `Board` holds state, the
`RuleEngine` decides whether the game is over and who won, and the `GameEngine` applies
moves. `AIEngine` picks a move for the opponent. Because the rule engine is an interface,
the same game engine would work for a different board size or win condition.

`Main.java` is an interactive console runner — it reads a row and a column each turn.
It still carries an `@SpringBootApplication` annotation inherited from the repository
this code came from, which the game never uses. Deleting that annotation and its import
removes the last Spring dependency from this repository; see `tictactoe/build.gradle`.

## elevator

The largest of the three, and the one with real scheduling logic.

A single car is driven by `ElevatorController`, which holds two priority queues — a
min-heap of stops above the car and a max-heap of stops below it — and advances one
floor per call to `step()`. This implements a LOOK-style sweep: the car keeps moving in
one direction and serves every eligible stop on the way, then reverses when nothing is
left ahead. That keeps travel efficient without letting anyone wait forever.

Two details in there are worth reading closely, because they are the parts that are easy
to get wrong:

- **Requests are filed by the direction the car must travel to reach the floor**, not by
  the direction the rider pressed. Those two often disagree, and using the rider's button
  to pick the queue puts unreachable floors at the head of a heap, where they block every
  stop behind them.
- **A hall call whose rider wants the opposite direction moves between queues mid-journey.**
  It sits in one queue to pull the car towards it, then transfers to the other on arrival,
  so the doors open only on a sweep that is heading the rider's way.

For multiple cars, `ExternalButtonDispatcher` decides which car answers a hall call, and
delegates that decision to a swappable `ElevatorSelectionStrategy`. Three strategies are
included — nearest car, direction-aware, and least loaded — and the same button press
routes to a different car under each one. `DemoElevatorDesign` runs five scenarios that
demonstrate all of the above.

### Known gaps

Deliberately not implemented, and worth knowing before reading the code as a reference:

- Nothing is thread-safe. Button presses from multiple threads corrupt the queues, and
  the routing decision reads the car's floor and then inserts, so the car can move in
  between and strand a request behind itself.
- The dispatchers are static singletons, which means one building per process and no
  test isolation.
- There is no door state, no capacity limit, no notion of real time, and no way to take
  a car out of service.

### Write-ups

[`elevator/docs/elevator-lld-article.html`](elevator/docs/elevator-lld-article.html) is a
walkthrough that builds this design from the most naive version — one class, one queue of
plain integers — and lets each class earn its place by fixing a bug that the previous
version actually had. The diagrams it uses are in `elevator/docs/images/`.

[`elevator/docs/singleton-dispatcher-note.html`](elevator/docs/singleton-dispatcher-note.html)
is a design note on the static singleton dispatcher listed under Known gaps above: seven
concrete failures it causes, each reproduced against this code, and what constructor
injection replaces them with.
