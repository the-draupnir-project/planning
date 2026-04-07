# TDD Report

## Context

We need to get better with tests in the Draupnir project. Tests have been
neglected, it is unfortunate. The reason this has happened is because we haven't
found the right formula that works for us yet. All of our tests contain ugly
imperative brittle code that is the exact opposite of what we strive for in our
implementation source code. Tests lack declarative structure, and it is unclear
why they exist, why they are important, and what behaviour they are meant to be
testing.

It's unclear if bad abstraction design in general leads to bad tests, and
whether changing our testing process would force cleaner abstractions.

## Unknowns

- Can we express the core invariants of a feature in a way that is:
  - concise
  - declarative
  - stable over time

- Does writing invariants + interface _before_ implementation improve:
  - refactor safety
  - design clarity
  - contributor understanding

- What is the cost overhead (time, cognitive load) of this style, and is it
  acceptable for “hardened” features?

- Can we prevent tests from ossifying accidental behaviour (implementation
  quirks) rather than intended invariants?

- Does this approach reduce bitrot on infrequently used paths (logging, edge
  features) enough to justify itself?

## Experiment

### Hypothesis

By designing from the outside first specifying invariants, then the public
interface, and only then the implementation. We will:

- keep tests concise and declarative,
- prevent tests from pinning internal implementation details,
- and end up with higher-quality abstractions and more stable behaviour over
  time.

In this opinionated version of TDD:

1. **Invariants**: We write down the behavioural laws of the
   feature/abstraction. Each invariant must include a short explanation of _why_
   it exists, to make future negotiation (change/removal) possible.

2. **Interface**: We design the public surface that makes those invariants
   usable. This is the types/signatures/config shape.

3. **Implementation**: We write the code that satisfies the interface while
   making all invariants pass.

### Evaluation plan

- For this experiment, a feature/abstraction is only considered “hardened” if:
  - its core invariants can be stated in a short, declarative form, and
  - those invariants are enforced by tests.

- If we cannot express the invariants declaratively, we treat this as a design
  smell:
  - either the abstraction is not yet well-formed and should remain “internal”,
  - or we need to refactor the concept until the invariants become clear.

- We will judge the experiment successful if, for the chosen pilot feature:
  - refactors feel safer and faster,
  - tests remain small and readable (no giant imperative scaffolding),
  - and contributors can understand the behaviour by reading the invariants +
    interface without diving into implementation.

### Exploration points

- We will design and use a utility to support work on projections in the policy
  list subscription preview work
  https://github.com/Gnuxie/matrix-protection-suite/pull/101.

## Outcomes

We developed a `SemanticType` abstraction that could document the behaviour of
an abstraction:

> A semantic type is used to describe the expected behavior of a type beyond its
> shape. At the moment invariants can be specified through the `Law` method.
> This adds structure to important behavioural contracts that would otherwise be
> placed loosely in unit tests with no organisation.

> Another important feature of the invariant law is that they describe why they
> are included in the sematic type which helps negotiating their removal if the
> underlying abstraction changes.

> Critically the laws as opposed to tests are also defined against the interface
> and not any single concrete implementation. But all implementations can test
> their implementation against the semantic type. This brings the benefits of
> test driven development directly to the stage where the interface (and thus
> abstraction) is defined (or changed). Before any concrete implementation is
> even considered.

- This worked but required a large amount of boiler plate to implement the
  checks. Which might not be appropriate for all cases. There is value in
  specifying the `what` and `why` of behaviour without having to provide a check
  to verify it.

- It's clear that wider use of the `SemanticType` abstraction would lead to
  repetition in the checks. This is already visible in
  https://github.com/Gnuxie/matrix-protection-suite/pull/101.

- It's too early to really evaluate it but it at least "felt" cleaner, more
  structured, and valuable than unit tests. Even if that's because it forces a
  TDD approach to designing, and then implementing the abstraction.

- There's a gap for how to apply `SemanticType` to existing abstractions in
  order to break them or modify their behaviour without completely rewriting
  them. This is important because obviously almost all of the abstractions in
  our code base haven't been designed this way.
