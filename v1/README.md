# Specification

> **DRAFT — written, not finalised.**

[`spec.md`](./spec.md) is the normative profile, written against what `cheqd/agent-toolkit`
actually implements and has tested end-to-end (TypeScript and Python, cross-language). It
proceeds on the assumption that the extension URI namespace question in the [repository
README](../README.md#open-questions) will resolve in cheqd's favour — if it does not, the
activation URI embedded in Agent Cards, the `A2A-Extensions` header and every conformance vector
changes. See `spec.md`'s own [§7](./spec.md#7-open-items--not-done) for that and every other open
item blocking this from being finalised or proposed anywhere, including an unresolved mismatch
between this repository's proposed trust-extension URI and the one `agent-toolkit` actually ships.
