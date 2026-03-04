# AI Agent Requests to Complete NanoClaw v3

### Precise order of AI Agent requests to fully implement the [NanoClaw v3 Specification](https://github.com/japer-technology/githubification/blob/main/.githubification/specification/nanoclaw-v3.md)

Each request below is a self-contained prompt for an AI coding agent (e.g., Claude Code, GitHub Copilot). Execute them in order. Each request builds on the output of the previous ones. The specification source is `japer-technology/githubification/.githubification/specification/nanoclaw-v3.md`.

v3 is the reconciled implementation spec that resolves contradictions between v1-style minimalism and v2-style redesign by choosing a **compatibility-first, incremental channel-addition architecture**. The core principle: **Preserve NanoClaw's proven internals; adapt only ingress/egress and lifecycle to GitHub's event model.**

---

## Phase 0 — Foundations

### Request 1: Create the `.github-nanoclaw/` directory structure and documentation

Create the self-contained `.github-nanoclaw/` folder structure for the NanoClaw v3 Githubification. This folder is the canonical deployable product boundary — it must contain all GitHub-mode runtime orchestration, state, templates, and documentation. This follows Non-Negotiable Invariant #1 ("Self-contained install unit") and #7 ("No host repo lock-in") from the v3 specification.

Create the following directory tree:

```
.github-nanoclaw/
├── README.md
├── AGENTS.md
├── github-nanoclaw-ENABLED.md
├── lifecycle/
├── install/
├── state/
│   ├── store/
│   ├── groups/
│   │   └── global/
│   │       └── CLAUDE.md
│   ├── data/
│   │   ├── sessions/
│   │   └── ipc/
│   ├── issues/
│   └── readable/
│       ├── issues/
│       ├── sessions-index.json
│       └── latest-run.json
└── tests/
```

For `README.md`: Document that this folder is the NanoClaw v3 Githubification runtime — the self-contained install unit. Explain the GitHub primitive mapping from Section 4.1 of the spec:

| GitHub Primitive | NanoClaw v3 Mapping |
|---|---|
| **Actions** | Ephemeral compute per issue event |
| **Git** | Durable state + memory + session artifacts |
| **Issues** | Conversation UI (one issue = one thread/group) |
| **Secrets** | Credential source for Claude and GitHub API |

Describe the one-shot execution pipeline from Section 4.2: Issue open/comment event → Auth + guard checks → Build NanoClaw message context for issue N → One-shot agent execution (containerized) → Post response comment → Persist + export state → Commit/push with retry. Describe the group identity mapping from Section 4.3 where `chat_jid` format is `gh:<owner>/<repo>#<issue_number>` and group folder is `github-issue-<issue_number>`. Include a "Key Design Decisions" section documenting: (1) Strategy 5 (Channel Addition) + one-shot GitHub execution mode chosen because NanoClaw already has channel abstraction and Actions are ephemeral, (2) Container isolation kept as a first-class invariant in v3.0, (3) SQLite as canonical persistence with mandatory human-readable companion exports for auditability, (4) Fail-closed controls maintained (auth + sentinel + event validation + bot-loop prevention), (5) Scheduler deferred to dedicated later phase (Phase 3), (6) Shipped as self-contained `.github-nanoclaw/` unit following Githubification composition principles.

For `AGENTS.md`: Document the agent instructions for GitHub-mode execution. Specify that the agent runs in one-shot mode per issue event, that each issue is an isolated conversation thread with no session/memory leakage across issues, that state persists via git commits, and that the agent must never write secrets into committed state. Reference the v3 spec Section 3 (Non-Negotiable Invariants).

For `github-nanoclaw-ENABLED.md`: Create the sentinel file that acts as the fail-closed guard (Section 9.1, gate #2). Include a comment explaining that deleting this file immediately halts all NanoClaw execution on new issue events. This is the "sentinel existence" check from the security model.

For `state/readable/sessions-index.json`: Initialize as an empty JSON object `{}`.

For `state/readable/latest-run.json`: Initialize as an empty JSON object `{}`.

For `state/groups/global/CLAUDE.md`: Initialize with a header comment stating this is the global memory file for NanoClaw GitHub mode, readable by all issues but writable only by maintainers.

Add `.github-nanoclaw/state/store/` to `.gitignore` (SQLite binary should not be committed directly — readable exports serve this purpose per Section 8.2). Also add `.github-nanoclaw/state/data/ipc/` to `.gitignore` since IPC files are transient. Add `.gitkeep` files to empty directories (`state/store/`, `state/data/sessions/`, `state/data/ipc/`, `state/issues/`, `state/readable/issues/`) so they are tracked by Git.

---

### Request 2: Add GitHub-mode path overrides to `src/config.ts`

Modify `src/config.ts` to support environment variable overrides for key directory paths used in GitHub Actions mode. This implements Section 6.2 of the v3 specification. Currently, paths like `STORE_DIR`, `GROUPS_DIR`, and `DATA_DIR` are derived from `process.cwd()`. Add the following overrides:

- `NANOCLAW_STORE_DIR` — overrides the default store directory (where SQLite DB lives)
- `NANOCLAW_GROUPS_DIR` — overrides the default groups directory (where per-group folders live)
- `NANOCLAW_DATA_DIR` — overrides the default data directory (where sessions, IPC, etc. live)

Implementation pattern: For each path constant, check if the corresponding environment variable is set. If set, use `path.resolve()` on that value. If not set, use the existing `process.cwd()` derivation as the default.

Also add the GitHub mode flags from Section 6.3:

- `IS_GITHUB_MODE`: exported boolean, `true` when `process.env.NANOCLAW_GITHUB_MODE === '1'`
- `ENABLE_SCHEDULER`: exported boolean, `true` unless `process.env.NANOCLAW_ENABLE_SCHEDULER === '0'`
- `ENABLE_CHANNEL_BOOT`: exported boolean, `true` unless `process.env.NANOCLAW_ENABLE_CHANNEL_BOOT === '0'`

In the GitHub Actions workflow, these will be set:
- `NANOCLAW_STORE_DIR=.github-nanoclaw/state/store`
- `NANOCLAW_GROUPS_DIR=.github-nanoclaw/state/groups`
- `NANOCLAW_DATA_DIR=.github-nanoclaw/state/data`
- `NANOCLAW_GITHUB_MODE=1`
- `NANOCLAW_ENABLE_SCHEDULER=0`
- `NANOCLAW_ENABLE_CHANNEL_BOOT=0`

Do not change any existing behavior when environment variables are not set. Existing tests must continue to pass.

---

### Request 3: Disable unsupported capabilities in GitHub mode

Modify `src/index.ts` to check the `IS_GITHUB_MODE`, `ENABLE_SCHEDULER`, and `ENABLE_CHANNEL_BOOT` flags from `src/config.ts`. This implements Section 6.3 of the v3 specification. When running in GitHub mode:

1. Do not start the scheduler loop (`startSchedulerLoop` or equivalent) when `ENABLE_SCHEDULER` is `false`.
2. Do not start the channel connect/polling loop when `ENABLE_CHANNEL_BOOT` is `false`.
3. Do not start the IPC watcher for persistent daemon-style task operations when `IS_GITHUB_MODE` is `true`.
4. Do not start the message loop when `IS_GITHUB_MODE` is `true`.

Agent execution via `runContainerAgent` must remain fully functional in GitHub mode — only the daemon loops are disabled. Guard each loop startup with the appropriate flag check. This ensures the main orchestrator can still be imported and used for single-shot execution without starting background processes.

Also set GitHub-mode container overrides per Section 6.5:
- Short idle timeout (e.g. 15 seconds) to avoid waiting 30 minutes on the default
- Bounded container timeout suitable for workflow constraints (e.g. 10 minutes)

Existing tests must continue to pass. Add a comment at each guard explaining why the capability is disabled in GitHub mode (ephemeral Actions runtime, one-shot execution model).

---

### Request 4: Create the authorization lifecycle module

Create `.github-nanoclaw/lifecycle/authorize.ts` that implements fail-closed authorization checking per Section 9.1 of the v3 specification.

The module must export an async function `authorize(actor: string, token: string, owner: string, repo: string): Promise<{ authorized: boolean; permission: string; reason?: string }>`.

Implementation:

1. Use the GitHub REST API (`GET /repos/{owner}/{repo}/collaborators/{username}/permission`) with the provided token to check the actor's permission level.
2. Authorize the actor only if their permission is `admin`, `maintain`, or `write` (Section 7.3, step 1).
3. Return `{ authorized: false, permission, reason: 'Insufficient permissions' }` for `read`, `none`, or any other level.
4. If the API call fails, return `{ authorized: false, permission: 'unknown', reason: 'Permission check failed' }` — this is the fail-closed behavior (Section 9.1).

Also export a function `isBotOrSelf(actor: string, botLogin?: string): boolean` that returns `true` if:
- The actor login ends with `[bot]`
- The actor equals the configured bot account login

This implements the loop prevention logic from Section 9.4.

Use `fetch` (Node.js built-in) for HTTP calls. Accept the GitHub API base URL via optional parameter to support testing.

---

### Request 5: Create the indicator lifecycle module

Create `.github-nanoclaw/lifecycle/indicator.ts` that manages GitHub issue reactions as status indicators per Section 10.1 of the v3 specification.

Export the following async functions:

- `addReaction(token: string, owner: string, repo: string, targetId: number, reaction: string, isComment: boolean): Promise<void>` — Adds a reaction to an issue or comment via the GitHub REST API. For comments: `POST /repos/{owner}/{repo}/issues/comments/{comment_id}/reactions`. For issues: `POST /repos/{owner}/{repo}/issues/{issue_number}/reactions`.
- `indicateStart(token: string, owner: string, repo: string, targetId: number, isComment: boolean): Promise<void>` — Adds the 🚀 (rocket) reaction when processing starts.
- `indicateSuccess(token: string, owner: string, repo: string, targetId: number, isComment: boolean): Promise<void>` — Adds the 👍 (+1) reaction when complete.
- `indicateFailure(token: string, owner: string, repo: string, targetId: number, isComment: boolean): Promise<void>` — Adds the 👎 (-1) reaction on auth failure or runtime error.

Use `fetch` for HTTP calls. Set headers: `Authorization: Bearer <token>`, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`. Handle errors gracefully (log and continue — indicator failure must not block agent execution). Accept GitHub API base URL via optional parameter for testing.

---

### Request 6: Create the commit-state lifecycle module

Create `.github-nanoclaw/lifecycle/commit-state.ts` that commits and pushes state changes after each successful agent run, with robust retry and rebase logic per Section 7.4 of the v3 specification.

Export an async function `commitAndPushState(options: { stateDir: string; message: string; maxAttempts?: number }): Promise<{ success: boolean; attempts: number; error?: string }>`.

Implementation:

1. Configure git user: `git config user.name "github-actions[bot]"` and `git config user.email "github-actions[bot]@users.noreply.github.com"`.
2. Stage all changes under the state directory: `git add <stateDir>`.
3. Check if there are staged changes (`git diff --cached --quiet`). If no changes, return success immediately.
4. Commit with the provided message.
5. Attempt `git push`.
6. If push fails due to conflict, run `git pull --rebase -X theirs` and retry push.
7. Retry up to `maxAttempts` (default 10) with backoff escalation: 1s → 3s → 5s → 7s → 10s → 15s → 20s → 25s → 30s → 30s (Section 7.4).
8. Return the result with attempt count.
9. If all retries fail, return `{ success: false, attempts: maxAttempts, error: '<actionable error message>' }`.

Use `child_process.execFile` (promisified) for git operations. Never include secrets or sensitive data in commit messages. Each commit message should follow the format: `chore(github-nanoclaw): state update for issue #<N>`.

This implements Non-Negotiable Invariant #5 ("Git as durable state: every run commits state changes").

---

## Phase 1 — Functional Core (v3.0)

### Request 7: Create the GitHub agent one-shot orchestrator

Create `.github-nanoclaw/lifecycle/github-agent.ts` as the main one-shot execution entrypoint for GitHub Actions mode. This is the centerpiece of the v3 architecture (Section 6.1) — it adapts NanoClaw's daemon-oriented execution model to GitHub's ephemeral, event-driven runtime.

Export an async function `runGitHubAgent(options: { eventPath: string; token: string; owner: string; repo: string }): Promise<{ success: boolean; response?: string; error?: string; issueNumber: number }>`.

Implementation:

1. **Read and parse the event payload** from `eventPath` (the `GITHUB_EVENT_PATH` environment variable). Determine event kind: `issues.opened`, `issues.reopened`, or `issue_comment.created` (Section 6.1, step 2). Extract: issue number, issue title, issue body, comment body (if comment event), comment ID (if comment event), actor login. Reject unrecognized event types with an error (fail-closed, Section 9.1 gate #3).

2. **Run security gates** (Section 9.1 — all must pass before execution):
   - **Bot-loop prevention** (gate #4): Call `isBotOrSelf(actor)` from `authorize.ts`. If true, return early with `{ success: true, response: undefined, issueNumber }` (silently skip, no error).
   - **Sentinel existence** (gate #2): Check that `.github-nanoclaw/github-nanoclaw-ENABLED.md` exists. If missing, return `{ success: false, error: 'Agent disabled — sentinel file missing', issueNumber }`.
   - **Actor authorization** (gate #1): Call `authorize(actor, token, owner, repo)`. If not authorized, return `{ success: false, error: 'Unauthorized: <reason>', issueNumber }`.

3. **Map event to NanoClaw identifiers** (Section 4.3):
   - `chat_jid`: `gh:<owner>/<repo>#<issue_number>`
   - `group_folder`: `github-issue-<issue_number>`
   - `isMain`: `false` for all issue groups

4. **Initialize state**: Ensure the group folder exists under `.github-nanoclaw/state/groups/github-issue-<N>/`. Create a `CLAUDE.md` memory file if it does not exist (initialize with issue title and number as context header). Write/update the canonical mapping object to `.github-nanoclaw/state/issues/<N>.json` per Section 8.3:
   ```json
   {
     "issueNumber": N,
     "chatJid": "gh:owner/repo#N",
     "groupFolder": "github-issue-N",
     "sessionId": "...",
     "sessionPath": "state/data/sessions/github-issue-N/.claude/...",
     "lastProcessedAt": "<ISO8601>"
   }
   ```

5. **Build the message prompt**: For `issues.opened`, combine issue title and body. For `issue_comment.created`, use the comment body. For `issues.reopened`, include a system note that the issue was reopened plus the original issue body. Normalize inbound message(s) to `NewMessage` format (Section 6.1, step 3) and format using `formatMessages()` from `src/router.ts`.

6. **Initialize database**: Call `initDatabase()` from `src/db.ts` (or `_initTestDatabase()` if using an in-memory DB for testing). Register the issue group in the database via `setRegisteredGroup()` if it does not already exist.

7. **Execute the agent**: Import and call `runContainerAgent` from `src/container-runner.ts` (Section 6.5). Construct the `ContainerInput` object:
   - `prompt`: the formatted message
   - `sessionId`: derived from issue number (e.g., `github-issue-<N>`)
   - `groupFolder`: `github-issue-<N>`
   - `chatJid`: `gh:<owner>/<repo>#<N>`
   - `isMain`: `false`
   - `secrets`: only required env vars (`ANTHROPIC_API_KEY`, `GITHUB_TOKEN`) — per Section 9.2, no secrets written into committed files
   Capture the output. This preserves the container boundary (Non-Negotiable Invariant #4).

8. **Format the response**: Import `formatOutbound` from `src/router.ts` to strip `<internal>...</internal>` tags from the agent output (Section 6.4). Return the cleaned response text.

The module must work both as an importable library (exporting `runGitHubAgent` for testing) and as a runnable CLI script. When executed directly (`npx tsx .github-nanoclaw/lifecycle/github-agent.ts`), read parameters from environment variables (`GITHUB_EVENT_PATH`, `GITHUB_TOKEN`, `GITHUB_REPOSITORY` for `owner/repo`) and call `runGitHubAgent`. Exit with code 0 on success, 1 on failure.

Export the mapping function `mapEventToGroup(owner: string, repo: string, issueNumber: number): { chatJid: string; groupFolder: string }` separately so tests can import it directly (Section 12).

---

### Request 8: Create the GitHub outbound adapter

Create `.github-nanoclaw/lifecycle/post-response.ts` that posts the agent's response back to the originating GitHub issue per Section 6.4.

Export an async function `postIssueComment(options: { token: string; owner: string; repo: string; issueNumber: number; body: string }): Promise<{ success: boolean; commentId?: number; error?: string }>`.

Implementation:

1. Use the GitHub REST API (`POST /repos/{owner}/{repo}/issues/{issue_number}/comments`) to create a new comment.
2. Set the `Authorization: Bearer <token>` header and `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`.
3. Strip `<internal>...</internal>` tags from the body before posting (Section 6.4).
4. If the response body exceeds 65536 bytes (GitHub comment limit — check with `Buffer.byteLength(body, 'utf8')` to account for multi-byte UTF-8 characters), truncate with a note: `\n\n---\n*Response truncated due to length.*`
5. v3.0 default response policy (Section 10.2): one consolidated issue reply per run; no internal reasoning tags; include brief failure reason when no model output.
6. Return the created comment ID on success.
7. On failure, return `{ success: false, error: <message> }`.
8. Accept GitHub API base URL via optional parameter for testing.

---

### Request 9: Create the readable state export module

Create `.github-nanoclaw/lifecycle/export-readable-state.ts` that produces human-readable state exports after each successful agent run. This implements Section 8.2 of the v3 specification and Non-Negotiable Invariant #6 ("Auditability: binary DB must have human-readable companion exports").

Export an async function `exportReadableState(options: { stateDir: string; issueNumber: number; chatJid: string; groupFolder: string; agentResponse?: string; issueTitle?: string; actor?: string; prompt?: string }): Promise<void>`.

Implementation — after each run, export the following:

1. **Issue summary markdown** (`state/readable/issues/<N>.md`): Write/update a markdown conversation log:
   ```markdown
   # Issue #<N>: <title>

   ## Latest Interaction

   ### <sender> — <timestamp>
   <message body>

   ### NanoClaw — <timestamp>
   <agent response>
   ```
   Append to the file if it already exists (conversation history accumulates).

2. **Session index** (`state/readable/sessions-index.json`): Update with the mapping:
   ```json
   {
     "<issue_number>": {
       "chatJid": "gh:owner/repo#N",
       "groupFolder": "github-issue-N",
       "lastActivity": "<ISO8601 timestamp>",
       "messageCount": <number>
     }
   }
   ```
   Read existing file, merge the update, write back.

3. **Latest run metadata** (`state/readable/latest-run.json`): Overwrite with:
   ```json
   {
     "issueNumber": N,
     "chatJid": "gh:owner/repo#N",
     "timestamp": "<ISO8601>",
     "status": "success" | "error",
     "actor": "<actor login>",
     "responseLength": <number>,
     "error": "<error message if failed>"
   }
   ```

4. **Issue mapping** (`state/issues/<N>.json`): Update the `lastProcessedAt` field to current timestamp per Section 8.3.

These readable exports are committed alongside the state, providing auditability. Use only Node.js built-in modules (`fs`, `path`).

---

### Request 10: Create the GitHub Actions workflow template

Create `.github-nanoclaw/install/github-nanoclaw-WORKFLOW.yml` — the workflow template that the installer will copy to `.github/workflows/github-nanoclaw-agent.yml`. This implements Sections 7.1–7.4 of the v3 specification.

Workflow specification:

```yaml
name: NanoClaw Agent

on:
  issues:
    types: [opened, reopened]
  issue_comment:
    types: [created]

concurrency:
  group: github-nanoclaw-${{ github.repository }}-issue-${{ github.event.issue.number }}
  cancel-in-progress: false

permissions:
  contents: write
  issues: write

jobs:
  nanoclaw-agent:
    runs-on: ubuntu-latest
    steps:
      - name: Ignore bot events
        if: >-
          github.event.sender.type == 'Bot' ||
          endsWith(github.event.sender.login, '[bot]')
        run: |
          echo "Skipping bot-generated event"
          exit 0

      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          fetch-depth: 0

      - name: Guard — check sentinel file
        run: |
          if [ ! -f ".github-nanoclaw/github-nanoclaw-ENABLED.md" ]; then
            echo "::error::Sentinel file missing. NanoClaw is disabled."
            exit 1
          fi

      - name: Authorize actor
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          PERMISSION=$(gh api "/repos/${{ github.repository }}/collaborators/${{ github.event.sender.login }}/permission" --jq '.permission')
          if [[ "$PERMISSION" != "admin" && "$PERMISSION" != "write" && "$PERMISSION" != "maintain" ]]; then
            echo "::error::Unauthorized: ${{ github.event.sender.login }} has '$PERMISSION' permission"
            exit 1
          fi

      - name: Indicate start
        uses: actions/github-script@v7
        with:
          script: |
            const isComment = !!context.payload.comment;
            if (isComment) {
              await github.rest.reactions.createForIssueComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: context.payload.comment.id,
                content: 'rocket'
              });
            } else {
              await github.rest.reactions.createForIssue({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                content: 'rocket'
              });
            }

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run NanoClaw agent
        id: agent
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          NANOCLAW_GITHUB_MODE: '1'
          NANOCLAW_ENABLE_SCHEDULER: '0'
          NANOCLAW_ENABLE_CHANNEL_BOOT: '0'
          NANOCLAW_STORE_DIR: .github-nanoclaw/state/store
          NANOCLAW_GROUPS_DIR: .github-nanoclaw/state/groups
          NANOCLAW_DATA_DIR: .github-nanoclaw/state/data
        run: |
          npx tsx .github-nanoclaw/lifecycle/github-agent.ts

      - name: Post response to issue
        if: always() && steps.agent.outcome == 'success'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const responsePath = '.github-nanoclaw/state/readable/last-response.md';
            if (fs.existsSync(responsePath)) {
              const body = fs.readFileSync(responsePath, 'utf8');
              if (body.trim()) {
                await github.rest.issues.createComment({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  issue_number: context.payload.issue.number,
                  body: body
                });
              }
            }

      - name: Export readable state
        if: always() && steps.agent.outcome == 'success'
        run: |
          npx tsx .github-nanoclaw/lifecycle/export-readable-state.ts

      - name: Commit and push state
        if: always() && steps.agent.outcome == 'success'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .github-nanoclaw/state/ || true
          if git diff --cached --quiet; then
            echo "No state changes to commit"
            exit 0
          fi
          git commit -m "chore(github-nanoclaw): state update for issue #${{ github.event.issue.number }}"
          MAX_ATTEMPTS=10
          BACKOFFS=(1 3 5 7 10 15 20 25 30 30)
          for i in $(seq 0 $((MAX_ATTEMPTS - 1))); do
            if git push; then
              echo "Push succeeded on attempt $((i + 1))"
              exit 0
            fi
            echo "Push failed on attempt $((i + 1)), rebasing..."
            git pull --rebase -X theirs
            sleep ${BACKOFFS[$i]}
          done
          echo "::error::Failed to push state after $MAX_ATTEMPTS attempts"
          exit 1

      - name: Indicate success
        if: success()
        uses: actions/github-script@v7
        with:
          script: |
            const isComment = !!context.payload.comment;
            if (isComment) {
              await github.rest.reactions.createForIssueComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: context.payload.comment.id,
                content: '+1'
              });
            } else {
              await github.rest.reactions.createForIssue({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                content: '+1'
              });
            }

      - name: Indicate failure
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const isComment = !!context.payload.comment;
            if (isComment) {
              await github.rest.reactions.createForIssueComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: context.payload.comment.id,
                content: '-1'
              });
            } else {
              await github.rest.reactions.createForIssue({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                content: '-1'
              });
            }
```

Key design decisions in this workflow:
- `issue_comment.edited` is **excluded** in v3.0 to avoid replay/duplicate semantics (Section 7.1). It can be added in Phase 3.
- `cancel-in-progress: false` ensures messages queue rather than cancel (Section 7.2).
- Bot filtering uses `endsWith(github.event.sender.login, '[bot]')` for loop prevention (Section 9.4).
- Backoff uses escalating delays (1s → 3s → 5s ...) not exponential, per Section 7.4.
- The workflow sets all GitHub mode environment variables (`NANOCLAW_GITHUB_MODE=1`, `NANOCLAW_ENABLE_SCHEDULER=0`, `NANOCLAW_ENABLE_CHANNEL_BOOT=0`) and path overrides pointing into `.github-nanoclaw/state/`.

---

### Request 11: Create the GitHub issue template

Create `.github-nanoclaw/install/github-nanoclaw-ISSUE-TEMPLATE.md` — an issue template for starting NanoClaw conversations. This will be copied to `.github/ISSUE_TEMPLATE/github-nanoclaw-chat.md` by the installer (Section 5.2).

```markdown
---
name: NanoClaw Chat
about: Start a conversation with the NanoClaw AI agent
title: "[NanoClaw] "
labels: nanoclaw
assignees: ''
---

<!-- Type your message below. The NanoClaw agent will respond in a comment. -->
<!-- Every comment on this issue is a new message in the conversation. -->
<!-- The agent maintains context across all comments on this issue. -->


```

---

### Request 12: Create the installer agents documentation

Create `.github-nanoclaw/install/github-nanoclaw-AGENTS.md` documenting agent behavior for repos that install NanoClaw via Githubification.

Content: Explain that this repo uses NanoClaw as a GitHub-native AI agent. Describe the interaction model (open an issue or comment on an existing NanoClaw issue to chat with the agent — per Section 10.3, comment creation is itself the trigger; explicit `@Assistant` prefix is not required). Describe the security model (only users with write/maintain/admin permissions can trigger the agent; the sentinel file `.github-nanoclaw/github-nanoclaw-ENABLED.md` must exist; bot events are ignored for loop prevention — per Sections 9.1–9.4). Describe the state model (conversation state persists in `.github-nanoclaw/state/` via git commits; each issue is isolated; state survives across workflow runs — per Sections 3, 8). Describe how to disable the agent (delete the sentinel file). Describe the reaction UX (🚀 when processing, 👍 when complete, 👎 on failure — per Section 10.1).

---

### Request 13: Create the idempotent installer

Create `.github-nanoclaw/install/github-nanoclaw-INSTALLER.ts` — a TypeScript script that installs NanoClaw Githubification into a target repository. This implements Section 11.

The installer must:

1. **Create `.github/workflows/github-nanoclaw-agent.yml`** — Copy from `.github-nanoclaw/install/github-nanoclaw-WORKFLOW.yml`. Skip if the file already exists (never clobber user-customized files silently — Section 11, requirement #2). Allow override with explicit `--force` flag.

2. **Create `.github/ISSUE_TEMPLATE/github-nanoclaw-chat.md`** — Copy from `.github-nanoclaw/install/github-nanoclaw-ISSUE-TEMPLATE.md`. Skip if exists.

3. **Initialize `.github-nanoclaw/`** if absent — Create the full directory structure (README.md, AGENTS.md, sentinel file, state directories). If `.github-nanoclaw/` already exists, only create missing subdirectories and files. Never overwrite existing state, memory, or configuration files (idempotent, Section 11 requirement #1).

4. **Initialize clean state** — Do not copy any sessions, persona, or group state from the source repo. State directories must be empty on fresh install (Section 11, requirement #3).

5. **Verify required secrets** — Print a summary of what was created, what was skipped (already existed), and what secrets need to be configured (`ANTHROPIC_API_KEY`). Document that `GITHUB_TOKEN` is provided automatically by Actions (Section 11, requirement #5).

The script must be runnable via `npx tsx .github-nanoclaw/install/github-nanoclaw-INSTALLER.ts [--force]`. Use only Node.js built-in modules (`fs`, `path`). Exit with code 0 on success, 1 on error.

Uninstall support: Accept a `--uninstall` flag that removes `.github/workflows/github-nanoclaw-agent.yml` and `.github/ISSUE_TEMPLATE/github-nanoclaw-chat.md`. Prompt (via stdout message) whether to also remove `.github-nanoclaw/state/` (Section 11, uninstall requirements).

---

## Phase 2 — Hardening

### Request 14: Improve error taxonomy and user-facing failure messages

Update `.github-nanoclaw/lifecycle/github-agent.ts` to classify errors and produce actionable user-facing messages. This enhances the v3.0 response policy (Section 10.2: "include brief failure reason when no model output").

Define the following error categories and their issue-comment responses:

1. **Authorization failure**: "⚠️ **Unauthorized**: You do not have sufficient permissions to use this agent. Required: write, maintain, or admin access."

2. **Sentinel disabled**: "⚠️ **Agent Disabled**: The NanoClaw agent is currently disabled in this repository. A maintainer must re-enable it by restoring the sentinel file."

3. **Container execution failure**: "❌ **Execution Error**: The agent encountered an error during execution. A maintainer can check the workflow logs for details."

4. **State commit failure**: "⚠️ **State Warning**: The agent responded but could not persist conversation state. The response may not be available in future context."

5. **Unknown/unexpected error**: "❌ **Internal Error**: An unexpected error occurred. Please try again or contact a maintainer."

When a failure occurs, post the appropriate message as an issue comment (using the outbound adapter from Request 8) and add the 👎 reaction (using the indicator module from Request 5), then exit gracefully.

---

### Request 15: Enhance loop prevention logic

Update `.github-nanoclaw/lifecycle/authorize.ts` and `.github-nanoclaw/lifecycle/github-agent.ts` to implement stricter loop detection per Section 9.4.

Ignore event when ANY of the following are true:

1. Actor login ends with `[bot]`.
2. Actor equals the configured bot account (configurable via `NANOCLAW_BOT_LOGIN` env var, default `github-actions[bot]`).
3. Payload originated from an agent-authored comment — detect by checking if the comment body starts with a known agent response prefix, or if the comment author matches the bot login.

Add a `isAgentAuthored(payload: any, botLogin: string): boolean` function that checks all three conditions. Call this before any authorization or agent execution.

Log the skip reason (e.g., "Skipping: actor is bot", "Skipping: agent-authored comment") for diagnostics.

---

### Request 16: Create structural tests for event-to-group mapping

Create `.github-nanoclaw/tests/mapping.test.ts` using vitest (the project already has vitest configured).

Test cases (Section 12.1, test #1):

1. **Issue opened event maps to correct group**: Given a payload with `action: 'opened'` and `issue.number: 42` in repo `owner/repo`, verify the `chatJid` is `gh:owner/repo#42` and the `groupFolder` is `github-issue-42`.

2. **Issue comment event maps to correct group**: Given a payload with `action: 'created'` and `issue.number: 7`, verify the same mapping: `chatJid` is `gh:owner/repo#7`, `groupFolder` is `github-issue-7`.

3. **Issue reopened event maps correctly**: Given a payload with `action: 'reopened'` and `issue.number: 15`, verify: `chatJid` is `gh:owner/repo#15`, `groupFolder` is `github-issue-15`.

4. **Different issues produce isolated groups** (Section 12.2, test #4): Events for issue #1 and issue #2 must produce different `chatJid` values and different `groupFolder` values.

5. **Group folder name is filesystem-safe**: Verify `groupFolder` contains no characters that would be invalid in a POSIX path (`/`, `\0`).

6. **isMain is always false for issue groups**: Verify the mapping always returns `isMain: false` for issue-based groups.

Import the `mapEventToGroup` function from `.github-nanoclaw/lifecycle/github-agent.ts` for testing.

---

### Request 17: Create structural tests for authorization gate and sentinel

Create `.github-nanoclaw/tests/auth-guard.test.ts` using vitest.

Test cases (Section 12.1, tests #2–3):

1. **Admin user is authorized**: Mock GitHub API returning `permission: 'admin'` → authorize returns `{ authorized: true }`.

2. **Write user is authorized**: Mock returning `permission: 'write'` → authorized.

3. **Maintain user is authorized**: Mock returning `permission: 'maintain'` → authorized.

4. **Read user is rejected**: Mock returning `permission: 'read'` → `{ authorized: false }`.

5. **None/unknown user is rejected**: Mock returning `permission: 'none'` → rejected.

6. **API failure is fail-closed**: Mock fetch throwing an error → `{ authorized: false, reason: 'Permission check failed' }`.

7. **Bot actor is detected**: `isBotOrSelf('github-actions[bot]')` returns `true`. `isBotOrSelf('dependabot[bot]')` returns `true`. `isBotOrSelf('realuser')` returns `false`.

8. **Bot with custom login is detected**: `isBotOrSelf('my-bot', 'my-bot')` returns `true`.

9. **Sentinel file missing blocks execution**: Simulate a run where `.github-nanoclaw/github-nanoclaw-ENABLED.md` does not exist. Verify that no agent invocation occurs and an error result with reason containing 'disabled' or 'sentinel' is returned.

10. **Sentinel file present allows execution**: When the sentinel file exists, execution proceeds past the guard to the authorization check.

---

### Request 18: Create structural tests for workflow validity

Create `.github-nanoclaw/tests/workflow-structure.test.ts` using vitest.

Test cases (Section 12.1, test #6):

1. **Workflow file is valid YAML**: Parse `.github-nanoclaw/install/github-nanoclaw-WORKFLOW.yml` and verify it parses without errors.

2. **Workflow triggers are correct**: Verify `on.issues.types` includes `opened` and `reopened`. Verify `on.issue_comment.types` includes `created`. Verify `edited` is NOT in `on.issue_comment.types` (v3.0 excludes it per Section 7.1).

3. **Concurrency group is per-issue**: Verify `concurrency.group` contains `issue` reference (pattern match for issue number interpolation). Verify `cancel-in-progress` is `false`.

4. **Required permissions are declared**: Verify `permissions` includes `contents: write` and `issues: write`.

5. **Sentinel guard step exists**: Verify there is a step that checks for the sentinel file `.github-nanoclaw/github-nanoclaw-ENABLED.md`.

6. **Authorization step exists**: Verify there is a step that checks collaborator permission.

7. **Bot ignore step exists**: Verify there is a step or condition that filters bot-generated events.

8. **Indicator steps exist**: Verify rocket, +1, and -1 reaction steps are present.

9. **GitHub mode environment variables are set**: Verify the agent step sets `NANOCLAW_GITHUB_MODE: '1'`, `NANOCLAW_ENABLE_SCHEDULER: '0'`, `NANOCLAW_ENABLE_CHANNEL_BOOT: '0'`.

10. **Path overrides point to `.github-nanoclaw/state/`**: Verify `NANOCLAW_STORE_DIR`, `NANOCLAW_GROUPS_DIR`, `NANOCLAW_DATA_DIR` are set and point into `.github-nanoclaw/state/`.

11. **State commit step exists with retry**: Verify there is a step that runs `git add`, `git commit`, and `git push` with retry logic and `git pull --rebase -X theirs` on conflict.

---

### Request 19: Create tests for state exports and config path overrides

Create `.github-nanoclaw/tests/state-export.test.ts` using vitest.

Test cases for readable state exports (Section 12.1, test #4):

1. **Issue markdown is created**: After calling `exportReadableState`, verify `state/readable/issues/<N>.md` exists and contains the issue title and agent response.

2. **Sessions index is updated**: After calling `exportReadableState`, verify `sessions-index.json` contains the issue number key with correct `chatJid` and `groupFolder`.

3. **Latest run metadata is written**: After calling `exportReadableState`, verify `latest-run.json` contains the correct `issueNumber`, `timestamp`, and `status`.

4. **Issue mapping is updated**: After calling `exportReadableState`, verify `state/issues/<N>.json` has an updated `lastProcessedAt` timestamp.

5. **Multiple issues maintain separate exports**: Export state for issue #1 and issue #2. Verify both `readable/issues/1.md` and `readable/issues/2.md` exist independently.

Create `.github-nanoclaw/tests/config-overrides.test.ts` using vitest.

Test cases for path overrides (Section 12.1, test #7):

1. **Default paths are unchanged when env vars are not set**: Import config and verify `STORE_DIR`, `GROUPS_DIR`, `DATA_DIR` resolve to the `process.cwd()`-based defaults.

2. **Env overrides take effect**: Set `NANOCLAW_STORE_DIR`, `NANOCLAW_GROUPS_DIR`, `NANOCLAW_DATA_DIR` in `process.env`, re-import config, and verify the paths resolve to the overridden values.

3. **IS_GITHUB_MODE is true when NANOCLAW_GITHUB_MODE is '1'**: Set `process.env.NANOCLAW_GITHUB_MODE = '1'` and verify `IS_GITHUB_MODE` is `true`.

4. **IS_GITHUB_MODE is false by default**: Unset `NANOCLAW_GITHUB_MODE` and verify `IS_GITHUB_MODE` is `false`.

5. **ENABLE_SCHEDULER defaults to true**: Verify `ENABLE_SCHEDULER` is `true` when `NANOCLAW_ENABLE_SCHEDULER` is not set.

6. **ENABLE_SCHEDULER is false when set to '0'**: Set `process.env.NANOCLAW_ENABLE_SCHEDULER = '0'` and verify `ENABLE_SCHEDULER` is `false`.

---

### Request 20: Create commit retry loop tests

Create `.github-nanoclaw/tests/commit-state.test.ts` using vitest.

Test cases (Section 12.1, test #5):

1. **Clean push succeeds on first attempt**: Mock `git push` succeeding → returns `{ success: true, attempts: 1 }`.

2. **Retry with rebase on conflict**: Mock `git push` failing once then succeeding → returns `{ success: true, attempts: 2 }`. Verify `git pull --rebase -X theirs` was called between attempts.

3. **Backoff delays are applied**: Mock push failing multiple times. Verify the wait times follow the escalation pattern from Section 7.4 (1s → 3s → 5s → 7s ...).

4. **Gives up after max attempts**: Mock push failing 10 times → returns `{ success: false, attempts: 10 }`.

5. **No-op when no changes**: When `git diff --cached` shows no changes, returns success without pushing.

6. **Commit message follows expected format**: Verify the commit message matches `chore(github-nanoclaw): state update for issue #<N>`.

---

### Request 21: Create integration test fixtures

Create `.github-nanoclaw/tests/fixtures/` with test event payloads for integration testing (Section 12.2).

Create the following fixture files:

1. **`issue-opened.json`**: A valid GitHub `issues.opened` event payload with `issue.number: 1`, `issue.title: "Test issue"`, `issue.body: "Hello agent"`, `sender.login: "testuser"`, `repository.full_name: "owner/repo"`.

2. **`issue-reopened.json`**: A valid GitHub `issues.reopened` event payload with `issue.number: 1`.

3. **`issue-comment-created.json`**: A valid GitHub `issue_comment.created` event payload with `issue.number: 1`, `comment.body: "Follow-up question"`, `comment.id: 12345`, `sender.login: "testuser"`.

4. **`issue-comment-bot.json`**: A valid GitHub `issue_comment.created` event payload where `sender.login: "github-actions[bot]"` and `sender.type: "Bot"` — used to verify bot filtering.

5. **`issue-opened-different.json`**: A valid GitHub `issues.opened` event payload with `issue.number: 2` — used to verify issue isolation (Section 12.2, test #4).

6. **`issue-comment-unauthorized.json`**: A valid GitHub `issue_comment.created` event payload with `sender.login: "readonly-user"` — used to verify authorization rejection.

These fixtures enable the integration tests from Section 12.2:
- Simulate `issues.opened` (fixture 1)
- Simulate `issue_comment.created` (fixture 3)
- Verify resume on second comment same issue (fixtures 1 then 3)
- Verify no leakage between issue A and issue B (fixtures 1 and 5)

---

## Phase 3 — Optional Enhancements

### Request 22: Create a formal GitHub channel adapter for the channel registry

Create `src/channels/github.ts` that integrates GitHub Issues as a first-class channel in the NanoClaw channel registry, following the same factory pattern as other channels (Section 13, Phase 3 enhancement).

The channel factory must:

1. Register via `registerChannel('github', factory)`.
2. Return `null` if not in GitHub mode (`IS_GITHUB_MODE` is false) — this channel only activates in GitHub Actions.
3. Implement the `Channel` interface with:
   - `name`: `'github'`
   - `connect()`: No-op (stateless, runs in GitHub Actions context)
   - `sendMessage(jid, text)`: Posts text as an issue comment, extracting issue number from the `gh:<owner>/<repo>#<N>` JID format.
   - `isConnected()`: Returns `true` when `process.env.GITHUB_ACTIONS` is set.
   - `ownsJid(jid)`: Returns `true` for JIDs starting with `gh:`.
   - `disconnect()`: No-op.
   - `setTyping(jid, isTyping)`: When `isTyping` is true, add 🚀 reaction; when false, no action needed.
4. On channel initialization, read `GITHUB_EVENT_PATH` and parse the event to determine the target issue.

Register the channel in `src/channels/index.ts` by adding `import './github.js'` alongside the existing channel imports.

This enables the existing NanoClaw routing and message loop to work with GitHub Issues transparently, without special-casing in the orchestrator. The channel returns `null` from its factory when not running in GitHub Actions, so it has zero impact on local operation.

---

### Request 23: Add `issue_comment.edited` handling semantics

Update the workflow template and orchestrator to optionally support `issue_comment.edited` events (Section 13, Phase 3).

Implementation:

1. Add `edited` to the `issue_comment` types in the workflow template (behind a configuration flag `NANOCLAW_ALLOW_EDITED_COMMENTS`).
2. When an edited comment event is received, treat it as a new message with the updated body. Add a system note prefix: `[Comment edited] ` before the body to distinguish from new comments.
3. Add deduplication logic: if the last processed message for this issue had the same comment ID, skip processing to avoid replay.
4. Document the trade-offs in `README.md`: edited comments may cause duplicate processing if the deduplication logic fails; this is why they are excluded from v3.0 by default.

---

### Request 24: Create a GitHub-native scheduled task workflow

Create a separate workflow and lifecycle script for scheduled tasks (Section 13, Phase 3 — scheduler deferred to dedicated later phase).

1. Create `.github-nanoclaw/install/github-nanoclaw-SCHEDULED.yml`:
   ```yaml
   name: NanoClaw Scheduled Tasks

   on:
     schedule:
       - cron: '*/5 * * * *'
     workflow_dispatch:
       inputs:
         task_id:
           description: 'Specific task ID to run (optional)'
           required: false
           type: string

   concurrency:
     group: github-nanoclaw-scheduler
     cancel-in-progress: false

   permissions:
     contents: write
     issues: write
   ```

2. Create `.github-nanoclaw/lifecycle/schedule-bridge.ts` that:
   - Reads `scheduled_tasks` from the NanoClaw database
   - Finds due tasks (`status === 'active'` and `next_run <= now`)
   - Executes each due task via the same one-shot agent entrypoint
   - Posts results to the designated tracking issue
   - Updates `next_run` using `cron-parser` for cron tasks
   - Commits state after all tasks complete

3. Update the installer to optionally install this workflow when `--with-scheduler` flag is passed.

---

### Request 25: Final validation and acceptance testing

Run the complete acceptance criteria check from Section 14 against the implementation:

1. **Issue open triggers response on Actions**: Create a test issue payload for `issues.opened`. Run the workflow logic locally (mocked). Verify the agent is invoked and a response is generated.

2. **New comment on same issue resumes context**: Run two sequential comment events for the same issue. Verify the second run has access to the first run's conversation state (session continuity).

3. **Different issues remain isolated**: Run events for issue #1 and issue #2. Verify their state directories, session files, and memory files are completely separate. No session/memory leakage.

4. **Unauthorized actor cannot execute agent**: Run with a read-only user payload. Verify execution is blocked and a 👎 reaction is added.

5. **Sentinel removal stops all processing**: Delete `.github-nanoclaw/github-nanoclaw-ENABLED.md` and run an event. Verify execution halts immediately with an appropriate message.

6. **State persists across runs through git commits**: Run an agent, commit state, then run a second event. Verify the committed state is loaded correctly.

7. **Push conflicts resolve via retry/rebase loop**: Simulate a push conflict. Verify the retry loop with `git pull --rebase -X theirs` handles it correctly.

8. **Readable exports are updated each run**: Verify `state/readable/issues/<N>.md`, `sessions-index.json`, and `latest-run.json` are all updated after a successful run.

9. **No secrets appear in committed state**: Scan all files under `.github-nanoclaw/state/` for patterns matching API keys or tokens. Verify none are found.

10. **Workflow runtime is bounded and does not hang on idle timeout defaults**: Verify the GitHub mode sets a short idle timeout (Section 6.5) and the container timeout is bounded.

Run all structural tests (`npx vitest run .github-nanoclaw/tests/`). Ensure all pass. Run the existing NanoClaw test suite (`npm test`) and ensure no regressions.

---

## Summary

| Request | Phase | Creates / Modifies | Specification Section(s) |
|---|---|---|---|
| 1 | Phase 0 | `.github-nanoclaw/` folder structure, README, AGENTS, sentinel, state dirs | §3 Invariants, §4 Architecture, §5 Files |
| 2 | Phase 0 | `src/config.ts` path overrides + mode flags | §6.2, §6.3 |
| 3 | Phase 0 | `src/index.ts` GitHub mode guards | §6.3, §6.5 |
| 4 | Phase 0 | `.github-nanoclaw/lifecycle/authorize.ts` | §9.1, §9.4 |
| 5 | Phase 0 | `.github-nanoclaw/lifecycle/indicator.ts` | §10.1 |
| 6 | Phase 0 | `.github-nanoclaw/lifecycle/commit-state.ts` | §7.4 |
| 7 | Phase 1 | `.github-nanoclaw/lifecycle/github-agent.ts` | §6.1, §4.3, §8.3, §9.1 |
| 8 | Phase 1 | `.github-nanoclaw/lifecycle/post-response.ts` | §6.4, §10.2 |
| 9 | Phase 1 | `.github-nanoclaw/lifecycle/export-readable-state.ts` | §8.2 |
| 10 | Phase 1 | `.github-nanoclaw/install/github-nanoclaw-WORKFLOW.yml` | §7.1–§7.4 |
| 11 | Phase 1 | `.github-nanoclaw/install/github-nanoclaw-ISSUE-TEMPLATE.md` | §5.2 |
| 12 | Phase 1 | `.github-nanoclaw/install/github-nanoclaw-AGENTS.md` | §5.1, §10 |
| 13 | Phase 1 | `.github-nanoclaw/install/github-nanoclaw-INSTALLER.ts` | §11 |
| 14 | Phase 2 | Updates `lifecycle/github-agent.ts` error handling | §10.2 |
| 15 | Phase 2 | Updates `lifecycle/authorize.ts` + `github-agent.ts` loop detection | §9.4 |
| 16 | Phase 2 | `.github-nanoclaw/tests/mapping.test.ts` | §12.1 test #1 |
| 17 | Phase 2 | `.github-nanoclaw/tests/auth-guard.test.ts` | §12.1 tests #2–3 |
| 18 | Phase 2 | `.github-nanoclaw/tests/workflow-structure.test.ts` | §12.1 test #6 |
| 19 | Phase 2 | `.github-nanoclaw/tests/state-export.test.ts`, `config-overrides.test.ts` | §12.1 tests #4, #7 |
| 20 | Phase 2 | `.github-nanoclaw/tests/commit-state.test.ts` | §12.1 test #5 |
| 21 | Phase 2 | `.github-nanoclaw/tests/fixtures/*.json` | §12.2 |
| 22 | Phase 3 | `src/channels/github.ts`, updates `src/channels/index.ts` | §13 Phase 3 |
| 23 | Phase 3 | Updates workflow + orchestrator for edited comments | §13 Phase 3 |
| 24 | Phase 3 | Scheduler workflow + bridge script | §13 Phase 3 |
| 25 | Phase 3 | Acceptance testing against §14 criteria | §14 |

---

## Acceptance Criteria Cross-Reference (Section 14)

| # | Criterion | Validated By |
|---|---|---|
| 1 | Issue open triggers response on Actions | Requests 7, 10, 25 |
| 2 | New comment on same issue resumes context | Requests 7, 21, 25 |
| 3 | Different issues remain isolated | Requests 7, 16, 21, 25 |
| 4 | Unauthorized actor cannot execute agent | Requests 4, 7, 17, 25 |
| 5 | Sentinel removal stops all processing | Requests 1, 7, 17, 25 |
| 6 | State persists across runs through git commits | Requests 6, 10, 25 |
| 7 | Push conflicts resolve via retry/rebase loop | Requests 6, 10, 20, 25 |
| 8 | Readable exports are updated each run | Requests 9, 10, 19, 25 |
| 9 | No secrets appear in committed state | Requests 7, 10, 25 |
| 10 | Workflow runtime is bounded and does not hang | Requests 3, 10, 25 |
