---
name: codemind
description: |
  Delegate code changes to the Codemind AI agent via build_feature --
  the default first move for any task that touches source code
  (features, bug fixes, refactors, tests, single files). Also covers
  self-onboarding to Codemind's remote MCP server from zero credentials
  (create_free_account) when acting as or advising an agent/integration
  with no prior Codemind relationship. Invoke BEFORE writing any code.
---

# Codemind

Codemind is a remote MCP server that plans a component breakdown,
generates code against acceptance criteria, verifies every result
automatically before handing it back, and can review code standalone.
Your role — whatever agent or environment you're running in — is to
prepare the spec, call `build_feature`, and report the result, not to
implement code inline unless a build genuinely fails per the retry
guidance below.

**Source of truth**: every request/response shape here comes from
Codemind's real MCP tool schemas at `https://api.codemindhq.dev/mcp`.
If a tool's actual behavior ever contradicts something written here,
trust the live tool schema (`tools/list`) and the `get_usage_guide`
tool over this file, and treat the mismatch as this file being stale.

## Getting access

If `mcp__codemind__*` tools are already available in this environment
with no auth errors, skip this section — credentials are already
configured. Otherwise (a fresh environment, or every tool call fails
with an auth error), self-provision:

Call `create_free_account` (no arguments, no `Authorization` header).
The response is plain labeled text, not JSON:

```
Free account created.
tenantId: <uuid>
apiKey: <JWT>

Set this as your CODEMIND_API_KEY (or the Authorization: Bearer header
in .mcp.json) — usable immediately, no client restart required for
this same connection.
This is an anonymous free-tier account (plan=free, same limits as any
other free tenant) with no identity attached yet.

To attach an identity and unlock the dashboard, have your human visit:
<claim URL>
```

The last two lines may or may not appear — the claim URL only shows up
when identity provisioning succeeded, which never blocks account
creation. There's no OAuth prompt to wait for; the claim URL, when
present, is the only way to attach a human identity today.

Parse `tenantId:`/`apiKey:` out of the text (don't `JSON.parse` it).
Every other tool call needs `Authorization: Bearer <apiKey>`.

**Persist the key. Don't call `create_free_account` again** unless you
have to — it's limited per IP per rolling 24 hours (the limit is a
server setting and can change; treat an `IP_RATE_LIMIT_EXCEEDED` reply
as final for that day), keys
don't renew (90-day expiry, a new call means a brand-new unrelated
account), and if a human already completed identity sign-in on the
account via its claim URL, calling it again **silently creates a
disconnected new account** rather than refreshing the old one — ask
your human to sign in again instead. Only re-call freely when the
failing key was never claimed by a human. If a stored key fails with
error code `TOKEN_EXPIRED`, that's the one auth failure you can safely
tell apart from the rest; everything else (revoked, wrong key,
malformed, tenant suspended) collapses to a generic `UNAUTHORIZED`.

**Never print, log, or otherwise expose your `apiKey`.**

### Onboarding prompt for a human to paste

When a human asks how to set Codemind up in their own coding agent, give
them one of these two prompts verbatim. Both are copies of
`docs/onboarding-prompt.md` in the Codemind repository, which is the
source; a test there fails if a copy differs. The new-account prompt has
passed a full test (fresh account, saved config, a real build) on Claude
Code, Cursor, Codex and OpenCode. Cline is not supported yet.

No Codemind account yet:

<!-- onboarding-prompt:new-account:start -->
```text
Set up Codemind for this project. Codemind is a remote MCP server that writes and verifies code from a story.

Setup creates an account and stores its key. Do not create the account, read the key or write it anywhere yourself: some permission modes (Claude Code's auto mode, now its default) block an agent from storing a credential. I will run one setup script myself.

1. Write this script, exactly as given, to ./codemind-setup.sh in this project. It contains no secret.

#!/bin/sh
# Creates a Codemind free account and wires it into the current project.
# Usage: sh codemind-setup.sh [claude|cursor|codex|opencode]   (default: claude)
# Claude Code, Cursor and OpenCode get a config file in this project; Codex
# reads only its own config.toml in your home directory, so the entry goes there.
# Run it yourself (in Claude Code: `! sh codemind-setup.sh`). It stores the key
# in your shell profile and never prints it. The agent never sees the key.
set -eu

host="${1:-claude}"
case "$host" in
  claude)   file="./.mcp.json";         has='"codemind"' ;;
  cursor)   file="./.cursor/mcp.json";  has='"codemind"' ;;
  codex)    file="${CODEX_HOME:-$HOME/.codex}/config.toml"; has='mcp_servers.codemind' ;;
  opencode) file="./opencode.json";     has='"codemind"' ;;
  *) echo "Unknown agent '$host'. Use one of: claude, cursor, codex, opencode." >&2; exit 2 ;;
esac

if [ -n "${CODEMIND_API_KEY:-}" ]; then
  echo "account:    CODEMIND_API_KEY is already set; using it, no new account created"
else
  case "${SHELL:-}" in
    */zsh)  profile="$HOME/.zshrc" ;;
    */bash) if [ "$(uname)" = Darwin ]; then profile="$HOME/.bash_profile"; else profile="$HOME/.bashrc"; fi ;;
    *)      profile="$HOME/.profile" ;;
  esac
  # Check the profile is writable before creating an account, so a key is never created and then lost.
  if ! : >> "$profile"; then
    echo "Cannot write to $profile, so no account was created. Fix that and run this again." >&2
    exit 1
  fi

  reply=$(curl -fsS -X POST https://api.codemindhq.dev/mcp \
    -H 'content-type: application/json' \
    -H 'accept: application/json, text/event-stream' \
    -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"create_free_account","arguments":{}}}')

  key=$(printf '%s' "$reply" | grep -o 'apiKey: [A-Za-z0-9._-]*' | head -1 | cut -d' ' -f2)
  tenant=$(printf '%s' "$reply" | grep -o 'tenantId: [A-Za-z0-9-]*' | head -1 | cut -d' ' -f2)
  claim=$(printf '%s' "$reply" | grep -o 'https://[^" \\]*claim[^" \\]*' | head -1)
  if [ -z "$key" ]; then
    echo "No apiKey in the reply (rate limited or an error). Reply text:" >&2
    printf '%s\n' "$reply" | sed -E 's/apiKey: [A-Za-z0-9._-]+/apiKey: [redacted]/' | cut -c1-400 >&2
    exit 1
  fi

  printf "\nexport CODEMIND_API_KEY='%s'\n" "$key" >> "$profile"

  echo "tenantId:   $tenant"
  echo "key stored: $profile (CODEMIND_API_KEY, not printed)"
  echo "claim link: ${claim:-<none in reply>}"
fi

# The server entry for each agent. None of them contains the key: each names
# the CODEMIND_API_KEY variable in that agent's own syntax.
entry() {
  case "$host" in
    claude) cat <<'EOF'
{
  "mcpServers": {
    "codemind": {
      "type": "http",
      "url": "https://api.codemindhq.dev/mcp",
      "headers": { "Authorization": "Bearer ${CODEMIND_API_KEY}" }
    }
  }
}
EOF
    ;;
    cursor) cat <<'EOF'
{
  "mcpServers": {
    "codemind": {
      "url": "https://api.codemindhq.dev/mcp",
      "headers": { "Authorization": "Bearer ${env:CODEMIND_API_KEY}" }
    }
  }
}
EOF
    ;;
    codex) cat <<'EOF'
[mcp_servers.codemind]
url = "https://api.codemindhq.dev/mcp"
bearer_token_env_var = "CODEMIND_API_KEY"
EOF
    ;;
    opencode) cat <<'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "codemind": {
      "type": "remote",
      "url": "https://api.codemindhq.dev/mcp",
      "oauth": false,
      "headers": { "Authorization": "Bearer {env:CODEMIND_API_KEY}" },
      "enabled": true
    }
  }
}
EOF
    ;;
  esac
}

if [ -f "$file" ] && grep -qF "$has" "$file"; then
  echo "config:     $file already has a codemind entry; left as it is"
elif [ ! -f "$file" ]; then
  mkdir -p "$(dirname "$file")"
  entry > "$file"
  echo "wrote:      $file (no secret in it)"
elif [ "$host" = codex ]; then
  { printf '\n'; entry; } >> "$file"
  echo "added:      codemind entry appended to $file (no secret in it)"
else
  echo "config:     $file already exists and was NOT changed."
  echo "            Ask your agent to merge this codemind entry into it (no secret in it):"
  entry
fi

case "$host" in
  cursor) echo "Next: open a new terminal, run 'cursor agent mcp enable codemind', then restart Cursor from that terminal." ;;
  codex)  echo "Next: restart Codex from a new terminal, then run 'codex mcp list' to see the codemind entry." ;;
  *)      echo "Next: restart your agent from a new terminal; approve the project-scoped 'codemind' server if asked." ;;
esac

2. Tell me to run it myself, with the name of the agent you are (claude, cursor, codex or opencode), and wait until I say it is done: in Claude Code, type ! sh codemind-setup.sh claude ; in another agent, run sh codemind-setup.sh <name> in a terminal in this project. Do not run it yourself, and do not run any other command that creates an account or prints a key. The script puts CODEMIND_API_KEY in my shell profile and writes the server entry for that agent, with no secret in it (Claude Code ./.mcp.json, Cursor ./.cursor/mcp.json, OpenCode ./opencode.json, Codex config.toml in my home directory). If it says a config file already exists and was not changed, merge the entry it printed into that file for me.

3. Tell me to restart this agent from a new terminal in this project, so that it sees CODEMIND_API_KEY and the new tools, and to follow the "Next:" line the script printed (Cursor needs one approval command). If the agent asks me to approve the project-scoped "codemind" server, tell me to approve it.

4. Remind me to open the claim link the script printed. Until I do, the account is anonymous: it has no owner, I cannot see it in the dashboard, and it expires if unused.

5. Do not call create_free_account. If a Codemind call later fails with an auth error, call get_usage_guide with topic "auth" and follow it.

After the restart, confirm by calling the Codemind tool list_builds yourself (not curl with the key); an empty list means it works.
```
<!-- onboarding-prompt:new-account:end -->

Already has an account and an API key (for example from the dashboard):

<!-- onboarding-prompt:existing-key:start -->
```text
Set up Codemind for this project. Codemind is a remote MCP server that writes and verifies code from a story. I already have a Codemind account and an API key for it.

1. Do not call create_free_account: that would create a second, separate account. Do not ask me to paste the key into this chat.

2. I will store the key myself as the environment variable CODEMIND_API_KEY in the file my shell reads for every new interactive terminal (check $SHELL: ~/.zshrc for zsh, ~/.bashrc for bash, ~/.bash_profile for bash on macOS), so that this agent's own process has it when it starts. A project .env file is not enough: agents do not read it when they connect to an MCP server. Tell me the exact line to add, with a placeholder for the key, and wait until I say it is set. Do not print the key, and do not write it into the MCP config itself.

3. Add Codemind to this agent's MCP configuration as a remote HTTP server named "codemind":
   URL https://api.codemindhq.dev/mcp, header "Authorization: Bearer ${CODEMIND_API_KEY}".
   Use this agent's own config file and format. If the format cannot read an environment variable, tell me and stop; do not paste the key in.

4. Tell me to restart this agent from a new terminal, so that it sees CODEMIND_API_KEY and the new tools, and what to do after the restart. If the agent asks me to approve the project-scoped "codemind" server, tell me to approve it.

5. If a Codemind call later fails with an auth error, call get_usage_guide with topic "auth" and follow it.

When that is done, confirm by calling the Codemind tool list_builds itself (not curl with the key); a list, empty or not, means it works.
```
<!-- onboarding-prompt:existing-key:end -->

**Every tool returns formatted text, not a JSON object** — read values
out of the text (the one exception is the webhook JSON payload in
[reference/tools.md](reference/tools.md), a real HTTP POST body).

## What goes through Codemind

Every source file change in this environment — new features, endpoints,
components, middleware; bug fixes (framed as "rewrite X to fix Y" with
the fix in `acceptanceCriteria`); refactors; tests; single-file edits
(Codemind always returns full file content for you to write yourself).

**Exceptions — do inline, not via Codemind:**
- Config files: `wrangler.toml`, `package.json`, `tsconfig.json`, `fly.toml`
- Hand-written SQL migrations
- Documentation: `CHANGELOG.md`, `README.md`, `ARCHITECTURE.md`, log files
- Secrets/infra ops: `wrangler secret put`, DNS, dashboard changes
- Lock files, generated files
- Timing/closure functions (debounce, throttle, retry-with-backoff, memoize-with-TTL) — automated verification reliably can't validate these; see [reference/spec-writing.md](reference/spec-writing.md)

## Writing a spec that succeeds

Before calling `build_feature`, read
[reference/spec-writing.md](reference/spec-writing.md) — it covers the
four checks that predict most rejections (one concern per call, no
timing/closure functions, no implementation details in the title, UI
edits need `existingFiles`), how to frame bug fixes and refactors, and
what `acceptanceCriteria` needs to include. Skipping this is the single
biggest cause of an avoidable failure.

**Modifying an existing file — always pass `existingFiles`.** If the
story changes a file that already exists, read it first and pass its
current content:

```
build_feature({
  storyTitle: "signup: set merchant_category from entity type",
  acceptanceCriteria: "Modify createMerchant in src/signup.ts to also persist merchant_category … keep all existing fields.",
  stackType: "worker",
  existingFiles: [{ path: "src/signup.ts", content: "<full current file content>" }],
  projectId: "my-repo-slug",
})
```

Without `existingFiles`, Codemind has no view of the file and returns
a from-scratch stub instead of a real edit. Full parameter reference —
`role`, the sibling-import requirement, `projectId`, `skipTestsFor`,
private-registry limits: [reference/patch-mode.md](reference/patch-mode.md).

## Generating code

```
build_feature({
  storyTitle: "<title>",
  acceptanceCriteria: "<full spec>",
  stackType: "<worker|node|react|...>",
  existingFiles: [...],   // when modifying existing code
  projectId: "<stable slug for this codebase>",
})
```

`stackType` is free text. JavaScript and TypeScript are the only
supported stacks today (`worker`, `node`, `react` and variants).
Python, Go, Swift, Kotlin and Rust are refused with
`UNSUPPORTED_STACK_TYPE` and a message naming the supported stacks.
Full detail:
[reference/stacks-and-errors.md](reference/stacks-and-errors.md).

The response is one of three things:
- **`{buildId}`** — accepted. Immediately call `stream_build(buildId)`
  — do not implement anything while waiting.
- **A clarifying-question text response** — your `acceptanceCriteria`
  was too vague to act on. Resubmit with more specifics. If rejected
  twice in a row, stop and ask your user for the missing detail rather
  than guessing a third time.
- **An `isError: true` throttle rejection**, text formatted as
  `[<CODE>] <message>`: `CONCURRENT_LIMIT_EXCEEDED` (too many of your
  builds already queued/running — clears when one finishes, no fixed
  reset), `RATE_LIMIT_EXCEEDED` (too many builds submitted in the last
  rolling hour — message ends with `Resets at <ISO timestamp>.` when
  computable; wait rather than poll), or `PLAN_LIMIT_EXCEEDED` (a
  monthly cap was reached — not transient). The same three apply to
  `retry_build`. All three scale with plan tier, but there's no
  self-serve upgrade path today — don't suggest "upgrade your plan" as
  an actionable step; tell your user the limit was hit and, for
  `PLAN_LIMIT_EXCEEDED` specifically, that raising it requires contact
  outside this API (email support@codemindhq.dev). This does NOT
  apply to `CAPACITY_EXHAUSTED` below — that one is global generation
  capacity, unrelated to plan tier, and a plan change would not help
  it at all.

**If `build_feature` itself errors** (a genuine tool-level error — bad
args, auth failure, not a build failure): surface it verbatim to the
user, don't implement inline.

## Watching progress and getting the result

`stream_build` narrates progress and resolves when the build finishes.
Connect immediately after `build_feature` returns — reconnecting late
can mean missing progress content, though a "still running" response
isn't a failure; call `stream_build(buildId)` again and it replays
from the start.

On success, `stream_build`'s own terminal response already contains
full file content — no separate call needed on this path. (If you
used a webhook instead of streaming, or you're revisiting a build from
earlier, call `get_build_files {buildId}` to fetch the files instead.)
Then:
1. Write each file to disk at its exact returned path
2. Run `/ship` (or commit + PR manually) to land them
3. Report: files generated, story title

Can't hold a streaming connection open? Pass `webhookUrl` +
`webhookSecret` to `build_feature` instead — delivery/signature
contract in [reference/tools.md](reference/tools.md).

## If a build doesn't succeed

Branch on `errorCode`, never the human-readable `error` text — it can
change wording without notice.

| errorCode | What to try |
|---|---|
| `CAPACITY_EXHAUSTED` | Wait, then `retry_build` — clears on its own, not something to fix in your spec. |
| `UPSTREAM_UNAVAILABLE` | Retry with `retry_build` — a dependency was briefly unreachable. |
| `BUILD_FAILED_QA` | Add concrete examples/edge cases to `acceptanceCriteria` and resubmit fresh. If it's a timing/closure function, implement inline instead (see the exceptions list above). |
| `BUILD_FAILED_REVIEW` | Same fix as `BUILD_FAILED_QA` — the code passed testing but not an automated review pass; tighten the spec and resubmit fresh. |
| `BUILD_FAILED_GENERATION` | Resubmit fresh with a narrower or more concrete spec. |
| `BUILD_FAILED_LIMITS` | Split the story — it's too large for one build (see Check 1 in spec-writing.md). |
| `INVALID_INPUT` | Fix the request itself, don't retry unchanged — usually a story too large to split, or one asking only for a test file (provide an implementation; tests are generated automatically). |
| `ORACLE_INVALID` | See just below — almost always fixable. |
| `INTERNAL_ERROR` | Retry once with `retry_build`. |

**If you see `ORACLE_INVALID`**, it's almost always one of two fixable
things — try both before treating it as a dead end:
1. Add concrete literal input/output examples, exact function
   signatures, exact error messages to `acceptanceCriteria`.
2. Supply real content via `existingFiles` for anything your criteria
   references by name (a shared package, a sibling module) — this is
   the single most common cause, even with an otherwise concrete spec.
   The error message will often name the specific missing piece
   directly.

Retry once with a **fresh** `build_feature` call addressing whichever
applies (not `retry_build`, which resubmits the identical failing
spec). If two well-targeted attempts still fail and the story
genuinely references nothing external, implement inline — and check
whether it's worth flagging as a Codemind limitation in the target
repo's own bug-tracking convention.

**General rule**: retry at most once per failure type — a fresh,
improved `build_feature` call for anything needing a different spec
(`BUILD_FAILED_QA`/`BUILD_FAILED_REVIEW`/`BUILD_FAILED_GENERATION`/
`BUILD_FAILED_LIMITS`/`INVALID_INPUT`/`ORACLE_INVALID`), `retry_build`
for the same spec against a transient condition
(`CAPACITY_EXHAUSTED`/`UPSTREAM_UNAVAILABLE`/`INTERNAL_ERROR`). Two
consecutive failures on the same story → implement inline.

## After landing

If this repo tracks its own docs for changes like this — a changelog,
an API/interfaces doc, a components doc, a testing doc; naming and
presence vary by repo — update them to reflect what shipped. In an
environment that follows this convention specifically: `CHANGELOG.md`
(feature + PR number), `INTERFACES.md` (endpoint added/changed),
`COMPONENTS.md` (new reusable component), `TESTING.md` (test coverage
changed materially). If the repo has no such convention, this step is
a no-op.

## Beyond the golden path

- **Full tool catalog** (retry_build, cancel_build, continue_build,
  get_build, get_build_files, get_build_spec, list_builds,
  notify_files_written, test_component, review_code, build_batch/
  get_batch/stream_batch/build_from_spec, get_usage_guide, and their
  streaming/polling variants): [reference/tools.md](reference/tools.md)
- **Dispatching several independent stories, or a raw spec, at once**
  instead of calling `build_feature` N times: `build_batch`/
  `build_from_spec` in reference/tools.md's "Batch dispatch" section.
- **Working against an existing codebase in full** (patch-mode
  parameters, iterative builds): [reference/patch-mode.md](reference/patch-mode.md)
- **Stack support and error codes in full**: [reference/stacks-and-errors.md](reference/stacks-and-errors.md)
- **Verification scenarios this file was checked against**: [EVALS.md](EVALS.md)
- **Anything not covered here**: call `get_usage_guide {topic}` —
  live, server-maintained guidance. Topics: `overview`, `patch-mode`,
  `webhooks`, `error-handling`, `common-failures`, `auth`,
  `cloud-swarm`. It needs a valid `apiKey` like any other tool.
