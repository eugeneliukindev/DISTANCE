# DISTANCE

Code design rules for any language.

## Exceptions

A rule gives way in two cases only. In both, name the rule in your reply.

1. The codebase already does it another way in several places: follow
   the codebase and leave existing code as it is.
2. The rule makes the code more complex in this place: skip it here.

A1 and T1 have no exceptions: new code passes errors up and validates
foreign input at the boundary, whatever the codebase does.

## D - public API

- D1. Extract a function, class or module when its public API is simpler
  than the code it hides. Length alone is no reason to split.
- D2. Shape the contract so the caller has nothing to handle separately:
  if the goal of the call is already reached, that is success.
- D3. A caller depends only on the methods it uses.

## I - dependencies

- I1. A business rule and a call to an external system (database,
  network, files, clock) live in different code. Only an adapter knows
  the external system: its API is named in business terms, it arrives
  through the constructor and is created at the entry point. Code that
  only moves data stays without an adapter.
- I2. Dependencies between packages run one way.

## S - single owner

- S1. Every decision (a rule, a format, a threshold) is written in one
  place. Similar code that changes for different reasons stays separate.
- S2. State with an invariant is changed only by its owner's method,
  and the owner checks the transition.
- S3. Everything a result depends on arrives as a parameter or through
  the constructor. Data that has no reason to change is immutable.

## T - boundary

- T1. Parse a foreign format (a request, a table row, an API response)
  once at the boundary into your own type that guarantees validity.
  Inside, trust the types.
- T2. Translate an external system's error the caller can react to into
  your own error, at the place where you call that system.

## A - errors

- A1. Catch only the error you expect and know what to do with. Pass
  every other error up.
- A2. Let a bug, and a failure nobody can react to, fall through to the
  single top-level handler.

## N - needed now

- N1. Write only what the task requires. Think ahead only about the
  stored data format and the external API.
- N2. Introduce an interface when there are already two implementations;
  a test fake counts. While there is one, depend on the class.

## C - contract

- C1. Above a public method, write what the name and types do not show:
  special cases, units, errors.
- C2. Every implementation and every subclass fulfills the whole
  contract. If it cannot, change the contract or hold the object in a
  field instead of inheriting.
- C3. A test calls the public API and checks the result. Your own fast
  code is real in a test; an external system is replaced by a fake.

## E - evident

- E1. A name states the meaning in the domain. A significant number or
  string gets a name.
- E2. Do the same thing the way the codebase already does it.
- E3. A comment explains a reason the code cannot show.

## Before you reply

Go through the code you changed and name every rule the change breaks,
with the exception that allows it.
