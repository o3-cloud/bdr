---
id: bdr-ears
layer: practices
type: spec
extends: [bdr]
---

# EARS Acceptance Criteria for BDRs

## Purpose

`bdr` requires acceptance criteria that a black-box observer can confirm, but it leaves their grammar open. Free-text criteria drift: they bundle two behaviours with "and", hide the precondition, blur normal and fault paths, and use words like "should" that do not say who must do what. EARS (Easy Approach to Requirements Syntax) limits each requirement to one of six sentence patterns built from the same parts: optional feature, optional state, optional trigger, one system name, `shall`, and one response. This spec makes every BDR acceptance criterion an EARS requirement and maps those parts onto the BDR's Given / When / Then scenarios. A reader, a test author or a loop can then turn each criterion into exactly one pass/fail test.

## References

- **spec** `bdr`: the format, lifecycle and black-box rules this spec extends
- **external** `https://alistairmavin.com/ears/`: EARS patterns, by the method's authors

## Patterns

| Pattern | Keyword | Template | When to use it |
|---|---|---|---|
| Ubiquitous | none | `The <system> shall <response>.` | Always true, with no state or trigger |
| Event-driven | `When` | `When <trigger>, the <system> shall <response>.` | A discrete request or event sets it off |
| State-driven | `While` | `While <state>, the <system> shall <response>.` | True for as long as a state holds |
| Unwanted behaviour | `If … then` | `If <condition>, then the <system> shall <response>.` | Invalid input, faults, conflicts, prohibitions |
| Optional feature | `Where` | `Where <feature>, the <system> shall <response>.` | Only applies when a feature or configuration is present |
| Complex | combined | `Where …, while …, when/if …, [then] the <system> shall <response>.` | Needs more than one clause |

## Rules

1. Write every item in a BDR's `## Acceptance Criteria` section as one EARS requirement, in the form `- AC-n: <requirement>`.
2. Use exactly one of the six patterns above for each criterion.
3. Use the system name declared in the BDR index (`docs/bdr/README.md`) in every criterion, verbatim. In Connect Four Arena, the system name is `the Arena`. Name the interface (REST API, `connect4` CLI, MCP server, browser UI) in the trigger, not as a different system name.
4. Write exactly one `shall` per criterion, followed by one response. If the response needs "and" to join two independently checkable results, split it into two criteria.
5. Reserve `shall` for the system's obligation. Do not use `should`, `may`, `will`, `must` or `can` in a criterion.
6. Capitalise the leading keyword (`Where`, `While`, `When`, `If`) and keep `shall` lower-case.
7. Combine clauses in this fixed order: `Where`, then `While`, then `When` or `If … then`.
8. Write every state, trigger and condition as something an external client can set up or observe: a request sent through a named interface, a game status returned by a state read, an elapsed time, a configuration flag. Do not write vague conditions such as "normally", "as appropriate" or "if needed".
9. Write every response as externally observable output: an HTTP status, an `error.code`, a field value in a response, a CLI exit code or output line, an MCP tool result, or rendered page content. Do not name internal types, functions, tables or frameworks (`bdr` rule 8).
10. Use the `If … then` pattern for every rejection, error, conflict, timeout and fault path, even when the trigger is a request, so reviewers can find all fault handling by its keyword.
11. State a prohibition as an `If … then` requirement with a positive, observable response (for example, "returns the same `version` as before the request") instead of "shall not".
12. Give every quantitative response a number and a unit (for example, "within 2 seconds", "at most 3 interactions"); subjective qualifiers stay banned (`bdr` rule 11).
13. Use the `Where` pattern only for behaviour that depends on a feature or configuration that a deployment can switch on or off, such as authentication or rate limits. Do not use it for game modes that every deployment supports; use `While` with the game's mode instead.
14. Keep a BDR to three to five criteria after splitting (`bdr` rule 12). If splitting by rule 4 gives more than five, split the behaviour into a second BDR.
15. Write the `## Behavior` sentence as a ubiquitous or event-driven EARS requirement, using the same system name.
16. Map every Verification scenario onto the criterion it proves: the `While` and `Where` clauses become **Given** steps, the `When` or `If` clause becomes the **When** step, and the response becomes the **Then** step.
17. Name the criterion each scenario verifies in its heading, in the form `### Scenario n: <title> (AC-n)`. Give every criterion at least one scenario.
18. Apply this spec to new BDRs and to `Proposed` BDRs when they are next edited. Do not rewrite an `Accepted` BDR to adopt it; `bdr` rule 14 forbids that. Convert an Accepted BDR only by superseding it with a new BDR.

## Example

BDR-006 (Reject invalid moves) in free text:

> - AC-1: A column below 0 or above 6 is rejected with error code `INVALID_COLUMN`.
> - AC-4: After any rejected move, `board`, `currentTurn` and `version` are the same as before the request.

The same criteria in EARS:

- AC-1: If a player submits a move with a column below 0 or above 6, then the Arena shall reject it with `error.code` `INVALID_COLUMN`.
- AC-2: If a player submits a move whose column is not an integer, then the Arena shall reject it with `error.code` `INVALID_COLUMN`.
- AC-3: If a player submits a move into a column that holds six discs, then the Arena shall reject it with `error.code` `COLUMN_FULL`.
- AC-4: If the Arena rejects a move, then the Arena shall return the same `board`, `currentTurn` and `version` on the next state read as before the request.

AC-4 checks three fields, but it states one observable result (the game state is unchanged), so it stays one criterion.

The matching scenario:

```markdown
### Scenario 1: Out-of-range column (AC-1, AC-4)

- **Given** an in-progress game `G` at `version` 0 with red to move
- **When** red submits `{"column": 7, "expectedVersion": 0}`
- **Then** the response has `error.code` `INVALID_COLUMN`
- **And** `GET /api/v1/games/G` shows `version` 0, `currentTurn` `red` and an empty board
```

More patterns, from other Arena BDRs:

- Ubiquitous: The Arena shall publish an OpenAPI document that lists every error code in SPEC.md §16.
- Event-driven: When a player creates a game through the REST API, the Arena shall return a game with status `waiting` and `version` 0.
- State-driven: While a game has status `in_progress`, the Arena shall report the colour to move in `currentTurn`.
- Unwanted behaviour: If a player submits a move when it is not their turn, then the Arena shall reject it with HTTP 409 and `error.code` `NOT_YOUR_TURN`.
- Optional feature: Where authentication is enabled, if a request to submit a move carries no credential, then the Arena shall reject it with HTTP 401 and `error.code` `UNAUTHENTICATED`.
- Complex: While a spectator has a game open in the browser UI, when either player makes a move through any interface, the Arena shall show that move on the spectator's board within 2 seconds.

## Checking

A criterion line conforms when it matches this pattern (Python `re` syntax):

```
^- AC-\d+: (Where [^,]+, )?([Ww]hile [^,]+, )?(([Ww]hen [^,]+, )|([Ii]f .+?, then ))?[Tt]he Arena shall (?!.*\bshall\b)[^;]+\.$
```

The pattern catches a missing or second `shall`, `should` in place of `shall`, wrong keyword order and a missing system name. Every example in this spec matches it. It does not judge whether a condition is observable or whether a response is one result; reviewers check those against rules 4, 8 and 9.
