---
name: tidy
description: Apply Kent Beck's "Tidy First?" methodology before implementing features or fixes. Use when: (1) about to change code and the structure feels messy, (2) asked to refactor or clean up code, (3) a PR mixes structural and behavioral changes, (4) code is hard to read or understand. Guides developer through separating structural tidyings from behavioral changes, and sequencing them correctly.
---

# Tidy First?

Based on Kent Beck's *Tidy First?* — separate structural changes from behavioral changes and sequence them deliberately.

## Core Rule

**Never mix tidying (structural) and feature work (behavioral) in the same commit.**

Ask: *Should I tidy first, after, or not at all?*

- **Tidy first** — if the change will be clearer or safer after tidying
- **Tidy after** — if the current structure is good enough to ship, clean up post-merge
- **Don't tidy** — if the code is being deleted soon, or the benefit doesn't justify the cost

## Workflow

1. Read the code you're about to change
2. Identify structural friction (hard to read? wrong shape for the change?)
3. Apply only the tidyings needed to make the behavioral change easier
4. Commit tidyings separately
5. Then make the behavioral change in a separate commit

## The 15 Tidyings

See [references/tidyings.md](references/tidyings.md) for the full list with examples.

Quick reference by category:

**Clarity**
- Guard Clauses — early return for preconditions
- Explaining Variables — name a sub-expression
- Explaining Constants — name a magic value
- Explaining Comments — add *why*, not *what*
- Delete Redundant Comments — remove comments that restate code

**Structure**
- Extract Helper — name a chunk of logic
- Chunk Statements — blank lines between logical steps
- Cohesion Order — move related code together
- Reading Order — arrange code in the order it's read
- One Pile — consolidate before splitting

**Cleanup**
- Dead Code — delete unused code
- Normalize Symmetries — make similar patterns identical
- Move Declaration And Initialization Together — co-locate `let x` and `x = ...`
- Explicit Parameters — remove implicit/hidden state passing
- New Interface, Old Implementation — wrap old API behind new signature

## Managing Tidyings

- Keep each tidy small — reviewable in minutes, reversible easily
- One tidy per commit, descriptive message: `tidy: extract helper validateInput`
- If a tidy reveals a bug, **stop** — file the bug, don't fix it mid-tidy
- Tidying is not perfectionism. Stop when the code is good enough for the change at hand

## Theory: Why Tidy?

Tidying creates **optionality** — future ability to change the code quickly. Like a financial option, it has value before it's exercised.

- **Time value of money**: ship features fast to earn sooner
- **Optionality**: clean structure = cheap future changes = that has economic value
- **Coupling & Cohesion**: reduce what must change together; increase what belongs together
