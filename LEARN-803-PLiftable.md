# Learner guide — Issue #803: "Add more cases for `checkPLiftableLaws`"

_Audience: you know Haskell (intermediate) but have never seen Plutarch. This walks you
from zero Plutarch knowledge to a concrete plan for the fix. Local note file — not committed._

Issue: <https://github.com/Plutonomicon/plutarch-plutus/issues/803>

> Currently only a few types, `PMaybe` and `PMaybeData`, are tested. We need to add tests to
> ensure `PLiftable` behaves correctly for other types — i.e. `PRational`, `PEither`, `PPair`, … — as well.

---

## 1. The 5-minute Plutarch primer (only what you need for #803)

Plutarch is an **eDSL embedded in Haskell** that compiles down to **UPLC** (Untyped Plutus
Core), the low-level language that runs on the Cardano blockchain. You write Haskell that
*builds a Plutarch program*, then compile/evaluate that program.

Four ideas are enough for this issue:

1. **The `P` prefix and the weird kind.** Plutarch's own types are prefixed with `P`:
   `PInteger`, `PBool`, `PMaybe`, `PEither`, `PPair`, `PRational`. They do **not** have kind
   `Type` like normal Haskell types. They have kind `S -> Type`.

2. **What `S` and `s` are.** `S` is a phantom "scope" tag, exactly like the `s` in the `ST`
   monad (`ST s a`). You'll see `s :: S` threaded everywhere. You almost never construct it
   yourself — just carry it along. So `PInteger :: S -> Type`, and a *value* of that type is a
   `Term s PInteger`.

3. **`Term s a` is a program fragment.** A `Term s a` is a piece of Plutarch code (an AST)
   that, when compiled and run, produces a value of Plutarch-type `a`. Think "quoted code",
   not "a plain value". `PInteger` is the *on-chain* integer type; it is **not** Haskell's
   `Integer`.

4. **The Haskell ⇄ Plutarch bridge.** Because `PInteger ≠ Integer`, you need conversions:
   - `pconstant :: AsHaskell a -> Term s a` — lift a Haskell value *into* a Plutarch term.
   - `plift :: Term s a -> AsHaskell a` — evaluate a closed Plutarch term *back* to Haskell.

   The typeclass that makes this bridge possible is **`PLiftable`**, and that is exactly what
   issue #803 is about testing.

That's it. You don't need to understand compilation, Scott encoding, or Data encoding to do
this issue — just the bridge.

---

## 2. What `PLiftable` actually is

Defined in `Plutarch/Internal/Lift.hs` (around line 164):

```haskell
class PlutusType a => PLiftable (a :: S -> Type) where
  type AsHaskell a  :: Type   -- the Haskell type that mirrors Plutarch type `a`
  type PlutusRepr a :: Type   -- an intermediate "Plutus universe" representation

  haskToRepr :: AsHaskell a  -> PlutusRepr a
  reprToHask :: PlutusRepr a -> Either LiftError (AsHaskell a)
  reprToPlut :: PlutusRepr a -> PLifted s a          -- into a Plutarch term
  plutToRepr :: (forall s. PLifted s a) -> Either LiftError (PlutusRepr a)
```

- **`AsHaskell a`** — the everyday Haskell type paired with the Plutarch type. Examples:
  | Plutarch type `a`      | `AsHaskell a`                          |
  |------------------------|----------------------------------------|
  | `PInteger`             | `Integer`                              |
  | `PMaybe PInteger`      | `Maybe Integer`                        |
  | `PEither a b`          | `Either (AsHaskell a) (AsHaskell b)`   |
  | `PRational`            | `PlutusTx.Rational`                    |
  | `PBuiltinPair a b`     | `(AsHaskell a, AsHaskell b)`           |

- **`PlutusRepr a`** — an intermediate form. It exists because Plutus's builtin containers
  (`PBuiltinList`, `PBuiltinPair`) can only hold a fixed set of "universe" types, so a value
  sometimes has to be converted in two hops (Haskell → repr → Plutarch) rather than one. For
  this issue you can treat it as plumbing.

### The laws (this is what we're testing)

From the class's doc comment:

1. `reprToHask . haskToRepr  ==  Right`  (Haskell → repr → Haskell round-trips)
2. `plutToRepr . reprToPlut  ==  Right`  (repr → Plutarch → repr round-trips)

Together these imply the headline property:

```
plift . pconstant  ==  id
```

i.e. "lift a Haskell value into Plutarch, evaluate it back, and you get the same value."
If a `PLiftable` instance is buggy, one of these fails.

---

## 3. What `checkPLiftableLaws` does

It's a ready-made property-test bundle. In `plutarch-testlib/Plutarch/Test/Laws.hs` (~line 298):

```haskell
checkPLiftableLaws ::
  forall (a :: S -> Type).
  ( Arbitrary (AsHaskell a)
  , Pretty    (AsHaskell a)
  , Eq        (AsHaskell a)
  , Show      (AsHaskell a)
  , PLiftable a
  ) =>
  [TestTree]
checkPLiftableLaws =
  [ testProperty "plutToRepr . reprToPlut = Right"  ...
  , testProperty "reprToHask . haskToRepr = Right"  ...
  , testProperty "plift . pconstant = id"           ...
  ]
```

Key takeaways:

- You call it **at the type level**: `checkPLiftableLaws @(PEither PInteger PInteger)`.
- It returns `[TestTree]` (a list of `tasty` property tests), which you drop into a `testGroup`.
- QuickCheck generates random `AsHaskell a` values via `Arbitrary`, so **`AsHaskell a` must
  have `Arbitrary`, `Pretty`, `Eq`, and `Show` instances.** ← *this is where the real work is.*

### How it's used today

`plutarch-testlib/test/Plutarch/Test/Suite/Plutarch/Maybe.hs`:

```haskell
, testGroup (instanceOfType @(S -> Type) @(PMaybe PInteger) "PLiftable") $
    checkPLiftableLaws @(PMaybe PInteger)
, testGroup (instanceOfType @(S -> Type) @(PMaybeData PInteger) "PLiftable") $
    checkPLiftableLaws @(PMaybeData PInteger)
```

So each new case is essentially **one `testGroup` line** — *if* the instance constraints are
already satisfied. When they aren't, you supply them (see §5).

---

## 4. What's already covered vs. what #803 wants

**Already exercised** (so you don't re-do these):
- `PMaybe`, `PMaybeData` — `.../Suite/Plutarch/Maybe.hs`
- `PFoo` (a `DeriveAsTag` enum) — `.../Suite/Plutarch/DeriveAsTag.hs`
- Ledger types via `checkLedgerProperties`, which *includes* `checkPLiftableLaws` internally —
  e.g. `PRawValue`, `PMap`, `PEitherData` (`Laws.hs`, `.../Suite/Plutarch/Either.hs`).

**Missing (the target list — add these):**
- `PRational`   → suite file `.../Suite/Plutarch/Rational.hs` (currently no liftable test)
- `PEither`     → `.../Suite/Plutarch/Either.hs` (only the *Data* variant `PEitherData` is tested)
- `PPair`       → `.../Suite/Plutarch/Pair.hs` (currently only golden tests, no laws)
- "…" — sensible extras with existing `PLiftable` instances, e.g. `PBuiltinPair`, and other
  small types you find lack coverage.

Confirm the exact gap yourself with a quick grep before starting:
```bash
grep -rn "checkPLiftableLaws\|checkLedgerProperties" plutarch-testlib/test
```

---

## 5. The fix, step by step

### The mechanical 90%

For each target type, add a group to its suite file. Pattern (copy from `Maybe.hs`):

```haskell
import Plutarch.Test.Laws (checkPLiftableLaws)
import Plutarch.Test.Utils (instanceOfType)

-- inside the suite's top-level `testGroup [...]`:
, testGroup (instanceOfType @(S -> Type) @(PEither PInteger PInteger) "PLiftable") $
    checkPLiftableLaws @(PEither PInteger PInteger)
```

Pick **monomorphic** element types (`PInteger`, `PBool`, `PByteString`, `PString`) so the
constraints resolve — e.g. `PEither PInteger PByteString`, `PPair PInteger PString`.

### The real 10% — satisfying the four instances on `AsHaskell a`

`checkPLiftableLaws @a` needs `Arbitrary`, `Pretty`, `Eq`, `Show` for `AsHaskell a`. `Eq`/`Show`
almost always exist. The two that bite are **`Arbitrary`** and **`Pretty`**:

- `AsHaskell (PEither PInteger PByteString) = Either Integer ByteString`. QuickCheck already has
  `Arbitrary (Either a b)`, but **`Prettyprinter` has no `Pretty (Either a b)`** → you must add one.
- `AsHaskell PRational = PlutusTx.Rational`. You likely need **both** an `Arbitrary` and a
  `Pretty` instance for it.

This missing-instance plumbing is exactly why the issue says other types "have been skipped" —
the one-liner is easy, the instances are the actual task.

**Where to put the instances** — follow the existing convention, don't invent a new one:
- The **`DeriveAsTag.hs`** suite shows the local pattern — define the needed instances right
  next to the test:
  ```haskell
  instance Pretty (PFoo s)    where pretty = pretty . show
  instance Arbitrary (PFoo s) where arbitrary = elements [A, B, C, D, E]
  ```
- Shared `Pretty` orphans already live in `plutarch-testlib/Plutarch/Test/QuickCheck.hs`
  (see the `deriving newtype instance Pretty ...` block ~line 184). If an instance is reusable
  (e.g. `Pretty (Either a b)`), add it there rather than duplicating per-suite.

> ⚠️ These are **orphan instances** (instance for a type + class you don't own). That's already
> accepted practice in this test library, but keep them in the test tree, never in the library
> proper.

### Watch out for
- **`PPair` may lack a `PLiftable` instance.** `Plutarch/Pair.hs` derives `PlutusType` via
  `DeriveAsSOPStruct` but I did not see a `PLiftable` instance. Check first:
  ```bash
  grep -n "PLiftable" Plutarch/Pair.hs
  ```
  If it's missing, either (a) add the appropriate `deriving ... PLiftable` line to the library
  (a slightly bigger change — mention it in the PR), or (b) start with `PBuiltinPair`, which
  *does* have a `PLiftable` instance (`Plutarch/Internal/Lift.hs:488`), and note `PPair` as
  follow-up. Prefer the smallest correct change.
- Compile errors here are your friend: a missing constraint is reported as
  `No instance for (Arbitrary ...)` / `(Pretty ...)` — that tells you exactly what to supply.

---

## 6. Build & run (from CONTRIBUTING.md)

```bash
nix develop          # first run downloads a lot; pins GHC 9.8.4 + native deps
cabal build all
cabal test plutarch-testlib
```

Iterate faster by filtering to your new group (tasty pattern match):

```bash
cabal test plutarch-testlib --test-options='--pattern "PLiftable"'
```

If you touch anything that emits golden output (you shouldn't for pure law tests) and a golden
diff appears, accept intentional changes with:

```bash
cabal test plutarch-testlib --test-options='--accept'
```

---

## 7. Suggested order of attack (smallest → biggest)

1. **`PEither PInteger PByteString`** in `Either.hs` — add the `testGroup` line; supply
   `Pretty (Either a b)` (Arbitrary already exists). Cleanest first win.
2. **`PRational`** in `Rational.hs` — add `Arbitrary` + `Pretty` for `PlutusTx.Rational`, then
   the `testGroup` line.
3. **`PBuiltinPair PInteger PByteString`** (or `PPair`, if/after it has `PLiftable`) in `Pair.hs`.
4. Grep for any other simple types with a `PLiftable` instance but no law coverage; add them.

Each type is an independent, compiler-guided change — perfect for small, reviewable commits.

## 8. Key files

| File | Why |
|------|-----|
| `Plutarch/Internal/Lift.hs` | `PLiftable` class, laws, `AsHaskell`/`PlutusRepr`, builtin instances |
| `plutarch-testlib/Plutarch/Test/Laws.hs` | `checkPLiftableLaws` definition (the 3 properties) |
| `.../test/.../Suite/Plutarch/Maybe.hs` | canonical usage to copy |
| `.../test/.../Suite/Plutarch/DeriveAsTag.hs` | how to supply `Arbitrary`/`Pretty` next to a test |
| `.../test/.../Suite/Plutarch/{Either,Pair,Rational}.hs` | the files you'll edit |
| `plutarch-testlib/Plutarch/Test/QuickCheck.hs` | home for shared `Pretty`/`Arbitrary` orphans |
| `plutarch-testlib/test/Test.hs` | where suites are registered (check your group runs) |
