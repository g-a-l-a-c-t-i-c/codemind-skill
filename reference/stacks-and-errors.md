# Stacks and error codes

## stackType values

`stackType` is a free-text field. The API refuses a known list of
stacks and treats every other value as JavaScript/TypeScript, so a
typo or an unlisted stack (for example `java`) is not refused: it is
built as JS/TS. Stick to the values below.

**Supported today: JavaScript and TypeScript only.**

| stackType | Runtime | QA method |
|---|---|---|
| `worker` | TypeScript / Cloudflare Workers | Real test execution |
| `node`, `react` and variants | JavaScript / TypeScript | Real test execution |

**Refused with `UNSUPPORTED_STACK_TYPE`:** `python`, `go`, `swift`
(`swift-ios`), `kotlin` (`kotlin-android`) and `rust`, plus `infra`,
`terraform`, `docs`, `markdown`, `sql`, `bash` and `shell`. The reply
names the supported stacks. A stack is opened once its results have
been measured.

**Retired — do not use:** `pages`, `expo`, `infra`, `design`.

## errorCode reference

Terse lookup table — see `SKILL.md`'s "If a build doesn't succeed"
section for the full guidance, including `ORACLE_INVALID`'s two-step
fix.

| errorCode | What it means | What to try |
|---|---|---|
| `CAPACITY_EXHAUSTED` | No generation capacity available right now | Wait, then `retry_build` |
| `UPSTREAM_UNAVAILABLE` | A dependency was briefly unreachable | `retry_build` |
| `INTERNAL_ERROR` | Unexpected server-side failure | `retry_build` once |
| `BUILD_FAILED_QA` | Generated code failed automated testing | Add concrete examples to `acceptanceCriteria`, resubmit fresh |
| `BUILD_FAILED_REVIEW` | Code passed testing but not an automated review pass | Same fix as `BUILD_FAILED_QA` |
| `BUILD_FAILED_GENERATION` | Code generation didn't produce valid output | Narrower/more concrete spec, resubmit fresh |
| `BUILD_FAILED_LIMITS` | Request too large for one build | Split the story |
| `INVALID_INPUT` | Request malformed, or asked only for a test file | Fix the request — don't retry unchanged |
| `ORACLE_INVALID` | Couldn't validate the result against your criteria | Add concrete examples, or supply real content via `existingFiles` for anything referenced by name |

`error` is always a human-readable string with no model names, account
IDs, or provider details. Branch your logic on `errorCode`, never on
this string.
