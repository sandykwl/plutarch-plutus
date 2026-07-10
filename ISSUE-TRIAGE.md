# Plutarch — Easiest Upstream Issues (personal triage)

Source: `Plutonomicon/plutarch-plutus` open issues (31 non-PR at time of scan).
Ranked easiest → hardest for a community first contribution. Doc source lives in
`plutarch-docs/src/` (the `book/` folder is rendered mdbook HTML — do NOT edit it).

_Local tracking file — not committed to git._

| Rank | Issue | Type | Why easy | Start point |
|------|-------|------|----------|-------------|
| 1 | #934 Consider updating repo documentation | boilerplate | Add CODEOWNERS, CODE_OF_CONDUCT.md, issue/PR templates (IntersectMBO base templates). CONTRIBUTING.md already done on current branch. | Create `.github/` (missing) + root files |
| 2 | #803 Add more cases for `checkPLiftableLaws` | testing | Add one-liners `checkPLiftableLaws @PRational`, `@PEither`, `@PPair`, … following existing pattern. Compiler-guided. | `plutarch-testlib/test/Plutarch/Test/Suite/Plutarch/Maybe.hs:52` |
| 3 | #954 More tests for `PBuiltinValue` interface | testing | Follow-on from #947; extend coverage in `Plutarch.LedgerApi.V3.Value`. Mechanical. | `plutarch-ledger-api` value tests |
| 4 | #959 Document `PLiftable` | docs | Single typeclass; MLabs blog to paraphrase + deriving example. | `plutarch-docs/src/Typeclasses/` |
| 5 | #958 Document use of the fixed point | docs | One focused conceptual topic (fix-point combinators). | `plutarch-docs/src/Concepts/` |
| 6 | #956 Document `PValidateData` | docs | Single new typeclass replacing `PTryFrom`; explain why + deriving examples. | `plutarch-docs/src/` |
| 7 | #960 Document what `Term` 'actually is' | docs | One conceptual explainer (Term = code-gen monad); blog to reference. | `plutarch-docs/src/Concepts/WhatIsTheS.md` |
| 8 | #955 Replace documentation of `PlutusType` | docs | Single typeclass; MLabs article to paraphrase. Also touches PInner/subtyping. | `plutarch-docs/src/Typeclasses/PlutusType,PCon,PMatch.md` |
| 9 | #686 CoArbitrary/Function instances for orphanage | enhancement (low priority) | Small, mechanical instance-writing; needs light QuickCheck familiarity. | Orphan-instance module |
| 10 | #971 Pretty-printing of `Case`/`Constr` is misleading | bug | Well-scoped, exact line pointers; fix grouping/separators. | `Plutarch/Pretty.hs:321-327` |

## Excluded as NOT low-hanging (despite tempting labels)
- #892, #313 — whole-guide rewrites / restructures (meta-issues)
- #957 — spans unit + golden + property testing
- #1003 — subjective cabal reorg, can balloon
- #962 — remove deprecations, broad
- #937, #984–#1002, #1009, #1011, #1015 — prettyprinter/optimizer enhancements, substantial design work

## Best first picks
- #934 — finish what the current branch began (easiest overall)
- #803 — cleanest standalone code contribution
