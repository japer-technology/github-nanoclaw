# AI Agent Requests to Complete NanoClaw Githubification

> Precise, ordered sequence of AI agent requests that fully implement the [NanoClaw Githubification Specification](https://github.com/japer-technology/githubification/blob/main/.githubification/specification/nanoclaw.md).
>
> Each request builds on the results of previous requests. Execute them in order.

---

## Phase 0 — Foundations

### Request 1: Create the `.github-nanoclaw/` directory structure and documentation

Create the self-contained `.github-nanoclaw/` folder structure for the NanoClaw Githubification. This folder must contain all GitHub-mode runtime orchestration, state, templates, and documentation — following the self-contained folder invariant from the Githubification playbook.

Create the following directory tree:

```
.github-nanoclaw/
├── README.md
├── AGENTS.md
├── github-nanoclaw-ENABLED.md
├── lifecycle/
├── install/
├── state/
│   ├── groups/
│   │   └── global/
│   ├── sessions/
│   ├── ipc/
│   └── readable/
│       ├── issues/
│       └── sessions-index.json
└── tests/
```

For `README.md`: Document that this folder is the NanoClaw Githubification runtime. Explain the GitHub primitive mapping (Actions = compute, Git = persistent state, Issues = chat UI, Secrets = credentials). Describe the one-shot runtime model: workflow receives issue event → authorization checks → event converted to NanoClaw message → agent executes once → response posted to issue → state committed. Reference the group model where `chat_jid` format is `gh:<owner>/<repo>#<issue_number>` and group folder is `github-issue-<issue_number>`. Include a "Key Design Decisions" section documenting: (1) Channel Addition strategy chosen because NanoClaw already has channel abstraction, (2) One-shot GitHub runtime mode because Actions are ephemeral, (3) Container isolation kept as a first-class invariant, (4) State persisted in git with readable exports to balance compatibility and auditability, (5) Fail-closed controls maintained (auth + sentinel + loop prevention), (6) Shipped as self-contained `.github-nanoclaw/` unit following Githubification composition principles.

For `AGENTS.md`: Document the agent instructions for GitHub-mode execution. Specify that the agent runs in one-shot mode per issue event, that each issue is an isolated conversation thread, that state persists via git commits, and that the agent must never write secrets into committed state.

For `github-nanoclaw-ENABLED.md`: Create the sentinel file that acts as the fail-closed guard. Include a comment explaining that deleting this file immediately halts all NanoClaw execution on new issue events.

For `state/readable/sessions-index.json`: Initialize as an empty JSON object `{}`.

For `state/groups/global/CLAUDE.md`: Initialize with a header comment stating this is the global memory file for NanoClaw GitHub mode, readable by all issues but writable only by maintainers.

Add `.github-nanoclaw/state/messages.db` to `.gitignore` (SQLite binary should not be committed directly — readable exports serve this purpose). Also add `.github-nanoclaw/state/ipc/` to `.gitignore` since IPC files are transient.

---

### Request 2: Add GitHub-mode path overrides to `src/config.ts`

Modify `src/config.ts` to support environment variable overrides for key directory paths used in GitHub Actions mode. Currently, paths are derived from `process.cwd()`. Add the following overrides:

- `NANOCLAW_STORE_DIR` — overrides the default `store/` directory (where SQLite DB lives)
- `NANOCLAW_GROUPS_DIR` — overrides the default `groups/` directory (where per-group folders live)
- `NANOCLAW_DATA_DIR` — overrides the default `data/` directory (where sessions, IPC, etc. live)

Implementation pattern: For each path constant (`STORE_DIR`, `GROUPS_DIR`, `DATA_DIR`), check if the corresponding environment variable is set. If set, use `path.resolve()` on that value. If not set, use the existing `process.cwd()` derivation as the default.

Also add an exported boolean `IS_GITHUB_MODE` that is `true` when `process.env.GITHUB_ACTIONS === 'true'`. This flag will be used by subsequent changes to disable unsupported capabilities.

Do not change any existing behavior when environment variables are not set. Existing tests must continue to pass.

---

### Request 3: Disable unsupported capabilities in GitHub mode

Modify `src/index.ts` to check the `IS_GITHUB_MODE` flag from `src/config.ts`. When running in GitHub mode (v1):

1. Do not start the scheduler loop (`startSchedulerLoop` or equivalent).
2. Do not start the channel connect/polling loop.
3. Do not start the IPC watcher for persistent daemon-style task operations.

Agent execution via `runContainerAgent` must remain fully functional in GitHub mode — only the daemon loops are disabled.

Guard each loop startup with `if (!IS_GITHUB_MODE) { ... }`. This ensures the main orchestrator can still be imported and used for single-shot execution without starting background processes.

Existing tests must continue to pass. Add a comment at each guard explaining why the capability is disabled in GitHub mode (ephemeral Actions runtime).

---

### Request 4: Create the authorization lifecycle module

Create `.github-nanoclaw/lifecycle/authorize.ts` that implements fail-closed authorization checking.

The module must export an async function `authorize(actor: string, token: string, owner: string, repo: string): Promise<{ authorized: boolean; permission: string; reason?: string }>`.

Implementation:

1. Use the GitHub REST API (`GET /repos/{owner}/{repo}/collaborators/{username}/permission`) with the provided token to check the actor's permission level.
2. Authorize the actor only if their permission is `maintain`, `write`, or `admin`.
3. Return `{ authorized: false, permission, reason: 'Insufficient permissions' }` for `read`, `none`, or any other level.
4. If the API call fails, return `{ authorized: false, permission: 'unknown', reason: 'Permission check failed' }` — this is the fail-closed behavior.
5. Also export a function `isBotOrSelf(actor: string, botLogin?: string): boolean` that returns `true` if the actor matches the bot account login or has the `[bot]` suffix, for loop prevention.

Use `fetch` (Node.js built-in) for HTTP calls. Accept the GitHub API base URL via optional parameter to support testing.

---

### Request 5: Create the indicator lifecycle module

Create `.github-nanoclaw/lifecycle/indicator.ts` that manages GitHub issue reactions as status indicators.

Export the following async functions:

- `addReaction(token: string, owner: string, repo: string, issueOrCommentId: number, reaction: string, isComment: boolean): Promise<void>` — Adds a reaction to an issue or comment via the GitHub REST API.
- `indicateStart(token: string, owner: string, repo: string, issueOrCommentId: number, isComment: boolean): Promise<void>` — Adds the 🚀 (rocket) reaction.
- `indicateSuccess(token: string, owner: string, repo: string, issueOrCommentId: number, isComment: boolean): Promise<void>` — Adds the 👍 (+1) reaction.
- `indicateFailure(token: string, owner: string, repo: string, issueOrCommentId: number, isComment: boolean): Promise<void>` — Adds the 👎 (-1) reaction.

Use `fetch` for HTTP calls. Handle errors gracefully (log and continue — indicator failure must not block agent execution). Accept GitHub API base URL via optional parameter for testing.

---

### Request 6: Create the commit-state lifecycle module

Create `.github-nanoclaw/lifecycle/commit-state.ts` that commits and pushes state changes after each successful agent run, with robust retry and rebase logic.

Export an async function `commitAndPushState(options: { stateDir: string; message: string; maxAttempts?: number }): Promise<{ success: boolean; attempts: number; error?: string }>`.

Implementation:

1. Stage all changes under the state directory: `git add <stateDir>`.
2. Check if there are staged changes. If no changes, return success immediately.
3. Commit with the provided message.
4. Attempt `git push`.
5. If push fails due to conflict, run `git pull --rebase -X theirs` and retry push.
6. Retry up to `maxAttempts` (default 10) with exponential backoff: wait `min(2^attempt * 1000, 30000)` ms between attempts.
7. Return the result with attempt count.

Use `child_process.execFile` (promisified) for git operations. Never include secrets or sensitive data in commit messages. Each commit message should follow the format: `chore(github-nanoclaw): state update for issue #<N>`.

---

### Request 7: Create the GitHub agent one-shot entrypoint

Create `.github-nanoclaw/lifecycle/github-agent.ts` as the main one-shot execution entrypoint for GitHub Actions mode.

Export an async function `runGitHubAgent(options: { eventPath: string; token: string; owner: string; repo: string }): Promise<{ success: boolean; response?: string; error?: string; issueNumber: number }>`.

Implementation:

1. **Read and parse the event payload** from `eventPath` (the `GITHUB_EVENT_PATH` environment variable). Support both `issues` (opened, reopened) and `issue_comment` (created, edited) events. Extract: issue number, issue title, issue body, comment body (if comment event), actor login.

2. **Map event to NanoClaw identifiers**:
   - `chat_jid`: `gh:<owner>/<repo>#<issue_number>`
   - `group_folder`: `github-issue-<issue_number>`

3. **Initialize state**: Ensure the group folder exists under `.github-nanoclaw/state/groups/github-issue-<N>/`. Create a `CLAUDE.md` memory file if it does not exist (initialize with issue title and number as context header).

4. **Build the message prompt**: For `issues.opened`, combine issue title and body. For `issue_comment.created`, use the comment body. For `issues.reopened`, include a system note that the issue was reopened plus the original issue body. For `issue_comment.edited`, use the new comment body.

5. **Prepare container input**: Construct the `ContainerInput` object compatible with `runContainerAgent` from `src/container-runner.ts`. Set paths using the GitHub-mode overrides (`NANOCLAW_STORE_DIR`, `NANOCLAW_GROUPS_DIR`, `NANOCLAW_DATA_DIR` pointing into `.github-nanoclaw/state/`). Include the formatted message as the prompt. Reuse NanoClaw's existing credential filtering model: pass only required auth environment variables to the container via stdin (never via mounted files), and never write secrets into committed state files.

6. **Execute the agent**: Import and call `runContainerAgent`. Capture the output.

7. **Format the response**: Import `formatOutbound` from `src/router.ts` to strip `<internal>...</internal>` tags from the agent output. Return the cleaned response text.

8. If any step fails, return `{ success: false, error: <message>, issueNumber }` with an actionable error description.

---

### Request 8: Create the GitHub outbound adapter

Create `.github-nanoclaw/lifecycle/post-response.ts` that posts the agent's response back to the originating GitHub issue.

Export an async function `postIssueComment(options: { token: string; owner: string; repo: string; issueNumber: number; body: string }): Promise<{ success: boolean; commentId?: number; error?: string }>`.

Implementation:

1. Use the GitHub REST API (`POST /repos/{owner}/{repo}/issues/{issue_number}/comments`) to create a new comment.
2. Set the `Authorization: Bearer <token>` header and `Accept: application/vnd.github+json`.
3. If the response body exceeds 65536 characters (GitHub comment limit), truncate with a note: `\n\n---\n*Response truncated due to length.*`
4. Return the created comment ID on success.
5. On failure, return `{ success: false, error: <message> }`.
6. Accept GitHub API base URL via optional parameter for testing.

---

## Phase 1 — Functional v1

### Request 9: Create the GitHub Actions workflow file

Create `.github-nanoclaw/install/github-nanoclaw-workflow.yml` — the workflow template that the installer will copy to `.github/workflows/`.

Workflow specification:

```yaml
name: NanoClaw Agent

on:
  issues:
    types: [opened, reopened]
  issue_comment:
    types: [created, edited]

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
          github.event.sender.login == 'github-actions[bot]'
        run: |
          echo "Skipping bot-generated event"
          exit 0

      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Guard — check sentinel file
        run: |
          if [ ! -f ".github-nanoclaw/github-nanoclaw-ENABLED.md" ]; then
            echo "::error::Sentinel file missing. NanoClaw is disabled."
            exit 1
          fi

      - name: Authorize actor
        id: auth
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
            const context_id = context.payload.comment
              ? context.payload.comment.id
              : context.payload.issue.id;
            const is_comment = !!context.payload.comment;
            if (is_comment) {
              await github.rest.reactions.createForIssueComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: context_id,
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
          NANOCLAW_STORE_DIR: .github-nanoclaw/state
          NANOCLAW_GROUPS_DIR: .github-nanoclaw/state/groups
          NANOCLAW_DATA_DIR: .github-nanoclaw/state
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
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.payload.issue.number,
                body: body
              });
            }

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
          for i in $(seq 1 $MAX_ATTEMPTS); do
            if git push; then
              echo "Push succeeded on attempt $i"
              exit 0
            fi
            echo "Push failed on attempt $i, rebasing..."
            git pull --rebase -X theirs
            BACKOFF=$((2 ** i))
            if [ $BACKOFF -gt 30 ]; then BACKOFF=30; fi
            sleep $BACKOFF
          done
          echo "::error::Failed to push state after $MAX_ATTEMPTS attempts"
          exit 1

      - name: Indicate success
        if: success()
        uses: actions/github-script@v7
        with:
          script: |
            const is_comment = !!context.payload.comment;
            if (is_comment) {
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
            const is_comment = !!context.payload.comment;
            if (is_comment) {
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

---

### Request 10: Create the GitHub issue template

Create `.github-nanoclaw/install/github-nanoclaw-issue-template.md` — an issue template for starting NanoClaw conversations.

```markdown
---
name: NanoClaw Chat
about: Start a conversation with the NanoClaw AI agent
title: "[NanoClaw] "
labels: nanoclaw
assignees: ''
---

<!-- Type your message below. The NanoClaw agent will respond in a comment. -->


```

---

### Request 11: Create the installer agents documentation

Create `.github-nanoclaw/install/github-nanoclaw-AGENTS.md` documenting agent behavior for repos that install NanoClaw via Githubification.

Content: Explain that this repo uses NanoClaw as a GitHub-native AI agent. Describe the interaction model (open an issue or comment on an existing NanoClaw issue to chat with the agent). Describe the security model (only users with write/maintain/admin permissions can trigger the agent; the sentinel file `.github-nanoclaw/github-nanoclaw-ENABLED.md` must exist; bot events are ignored for loop prevention). Describe the state model (conversation state persists in `.github-nanoclaw/state/` via git commits; each issue is isolated; state survives across workflow runs). Describe how to disable the agent (delete the sentinel file).

---

### Request 12: Create the idempotent installer

Create `.github-nanoclaw/install/github-nanoclaw-INSTALLER.ts` — a TypeScript script that installs NanoClaw Githubification into a target repository.

The installer must:

1. **Create `.github/workflows/github-nanoclaw-agent.yml`** — Copy from `.github-nanoclaw/install/github-nanoclaw-workflow.yml`. Skip if the file already exists (never overwrite user-customized workflows without explicit `--force` flag).

2. **Create `.github/ISSUE_TEMPLATE/github-nanoclaw-chat.md`** — Copy from `.github-nanoclaw/install/github-nanoclaw-issue-template.md`. Skip if exists.

3. **Initialize `.github-nanoclaw/`** if absent — Create the full directory structure (README.md, AGENTS.md, sentinel file, state directories). If `.github-nanoclaw/` already exists, only create missing subdirectories and files. Never overwrite existing state, memory, or configuration files.

4. **Initialize clean state** — Do not copy any sessions, persona, or group state from the source repo. State directories must be empty on fresh install.

5. **Print a summary** of what was created, what was skipped (already existed), and what secrets need to be configured (`ANTHROPIC_API_KEY`).

The script must be runnable via `npx tsx .github-nanoclaw/install/github-nanoclaw-INSTALLER.ts [--force]`. Use only Node.js built-in modules (`fs`, `path`). Exit with code 0 on success, 1 on error.

---

### Request 13: Create structural tests for event-to-group mapping

Create `.github-nanoclaw/tests/mapping.test.ts` using vitest.

Test cases:

1. **Issue opened event maps to correct group**: Given a payload with `action: 'opened'` and `issue.number: 42` in repo `owner/repo`, verify the `chat_jid` is `gh:owner/repo#42` and the `group_folder` is `github-issue-42`.

2. **Issue comment event maps to correct group**: Given a payload with `action: 'created'` and `issue.number: 7`, verify the same mapping: `chat_jid` is `gh:owner/repo#7`, `group_folder` is `github-issue-7`.

3. **Different issues produce isolated groups**: Events for issue #1 and issue #2 must produce different `chat_jid` values and different `group_folder` values.

4. **Group folder name is filesystem-safe**: Verify `group_folder` contains no characters that would be invalid in a POSIX path (`/`, `\0`).

Export the mapping function being tested from `.github-nanoclaw/lifecycle/github-agent.ts` (or a shared utility) so tests can import it directly.

---

### Request 14: Create structural tests for authorization gate

Create `.github-nanoclaw/tests/github-agent.test.ts` using vitest.

Test cases:

1. **Admin user is authorized**: Mock GitHub API returning `permission: 'admin'` → authorize returns `{ authorized: true }`.

2. **Write user is authorized**: Mock returning `permission: 'write'` → authorized.

3. **Maintain user is authorized**: Mock returning `permission: 'maintain'` → authorized.

4. **Read user is rejected**: Mock returning `permission: 'read'` → `{ authorized: false }`.

5. **None/unknown user is rejected**: Mock returning `permission: 'none'` → rejected.

6. **API failure is fail-closed**: Mock fetch throwing an error → `{ authorized: false, reason: 'Permission check failed' }`.

7. **Bot actor is detected**: `isBotOrSelf('github-actions[bot]')` returns `true`. `isBotOrSelf('dependabot[bot]')` returns `true`. `isBotOrSelf('realuser')` returns `false`.

8. **Sentinel file missing blocks execution**: When `.github-nanoclaw/github-nanoclaw-ENABLED.md` does not exist, agent execution must not proceed.

9. **Sentinel file present allows execution**: When the sentinel file exists, execution proceeds to authorization check.

---

### Request 15: Create structural tests for workflow validity

Create `.github-nanoclaw/tests/workflow-structure.test.ts` using vitest.

Test cases:

1. **Workflow file is valid YAML**: Parse `.github-nanoclaw/install/github-nanoclaw-workflow.yml` and verify it parses without errors.

2. **Workflow triggers are correct**: Verify `on.issues.types` includes `opened` and `reopened`. Verify `on.issue_comment.types` includes `created` and `edited`.

3. **Concurrency group is per-issue**: Verify `concurrency.group` contains `issue` reference (pattern match for issue number interpolation). Verify `cancel-in-progress` is `false`.

4. **Required permissions are declared**: Verify `permissions` includes `contents: write` and `issues: write`.

5. **Sentinel guard step exists**: Verify there is a step that checks for the sentinel file `.github-nanoclaw/github-nanoclaw-ENABLED.md`.

6. **Authorization step exists**: Verify there is a step that checks collaborator permission.

7. **Bot ignore step exists**: Verify there is a step that exits early for bot-generated events.

8. **Indicator steps exist**: Verify rocket, +1, and -1 reaction steps are present.

9. **State commit step exists**: Verify there is a step that runs `git add`, `git commit`, and `git push` with retry logic.

---

## Phase 2 — Hardening

### Request 16: Add readable state exports

Modify `.github-nanoclaw/lifecycle/github-agent.ts` to produce readable state exports after each successful agent run. This is required because SQLite diffs are opaque in git.

After the agent produces a response:

1. Write the agent's response to `.github-nanoclaw/state/readable/last-response.md`.

2. Write/update `.github-nanoclaw/state/readable/issues/<issue_number>.md` with a markdown conversation log:
   ```markdown
   # Issue #<N>: <title>

   ## Messages

   ### <sender> — <timestamp>
   <message body>

   ### NanoClaw — <timestamp>
   <agent response>
   ```

3. Update `.github-nanoclaw/state/readable/sessions-index.json` with the mapping:
   ```json
   {
     "<issue_number>": {
       "chat_jid": "gh:owner/repo#N",
       "group_folder": "github-issue-N",
       "last_activity": "<ISO timestamp>",
       "message_count": <number>
     }
   }
   ```

These readable exports are committed alongside the binary state, providing auditability.

---

### Request 17: Improve error taxonomy and user-facing failure messages

Update `.github-nanoclaw/lifecycle/github-agent.ts` to classify errors and produce actionable user-facing messages.

Define the following error categories and their issue-comment responses:

1. **Authorization failure**: "⚠️ **Unauthorized**: You do not have sufficient permissions to use this agent. Required: write, maintain, or admin access."

2. **Sentinel disabled**: "⚠️ **Agent Disabled**: The NanoClaw agent is currently disabled in this repository. A maintainer must re-enable it."

3. **Container execution failure**: "❌ **Execution Error**: The agent encountered an error during execution. A maintainer can check the workflow logs for details."

4. **State commit failure**: "⚠️ **State Warning**: The agent responded but could not persist conversation state. The response may not be available in future context."

5. **Unknown/unexpected error**: "❌ **Internal Error**: An unexpected error occurred. Please try again or contact a maintainer."

When a failure occurs, post the appropriate message as an issue comment (using the outbound adapter from Request 8) and add the 👎 reaction, then exit gracefully.

---

### Request 18: Add GitHub-mode path override tests

Add test cases to `.github-nanoclaw/tests/mapping.test.ts` (or a new file `.github-nanoclaw/tests/path-overrides.test.ts`) that verify:

1. **Default paths are unchanged when env vars are not set**: Import config and verify `STORE_DIR`, `GROUPS_DIR`, `DATA_DIR` resolve to the `process.cwd()`-based defaults.

2. **Env overrides take effect**: Set `NANOCLAW_STORE_DIR`, `NANOCLAW_GROUPS_DIR`, `NANOCLAW_DATA_DIR` in `process.env`, re-import config, and verify the paths resolve to the overridden values.

3. **IS_GITHUB_MODE is true when GITHUB_ACTIONS is set**: Set `process.env.GITHUB_ACTIONS = 'true'` and verify `IS_GITHUB_MODE` is `true`.

4. **IS_GITHUB_MODE is false by default**: Unset `GITHUB_ACTIONS` and verify `IS_GITHUB_MODE` is `false`.

5. **State files are placed under overridden directories**: When overrides point into `.github-nanoclaw/state/`, verify group folders, session directories, and IPC directories resolve within the override paths.

---

### Request 19: Add commit retry loop tests

Add test cases to `.github-nanoclaw/tests/github-agent.test.ts` (or a new file `.github-nanoclaw/tests/commit-state.test.ts`) that verify:

1. **Clean push succeeds on first attempt**: Mock `git push` succeeding → returns `{ success: true, attempts: 1 }`.

2. **Retry with rebase on conflict**: Mock `git push` failing once then succeeding → returns `{ success: true, attempts: 2 }`. Verify `git pull --rebase -X theirs` was called between attempts.

3. **Exponential backoff is applied**: Mock push failing multiple times. Verify the delay between attempts follows `min(2^attempt * 1000, 30000)` pattern.

4. **Gives up after max attempts**: Mock push failing 10 times → returns `{ success: false, attempts: 10 }`.

5. **No-op when no changes**: When `git diff --cached` shows no changes, returns success without pushing.

---

## Phase 3 — Advanced (Optional)

### Request 20: Create a formal GitHub channel adapter for the channel registry

Create `src/channels/github.ts` that integrates GitHub Issues as a first-class channel in the NanoClaw channel registry, following the same factory pattern as other channels (`src/channels/registry.ts`).

The channel factory must:

1. Register via `registerChannel('github', factory)`.
2. Return `null` if not in GitHub mode (`IS_GITHUB_MODE` is false) — this channel only activates in GitHub Actions.
3. Implement the `Channel` interface with:
   - `name`: `'github'`
   - `sendMessage(jid, text)`: Posts text as an issue comment (extracting issue number from the `gh:owner/repo#N` JID format).
   - `getJidFromEvent()`: Returns the `chat_jid` derived from the GitHub event payload.
4. On channel initialization, read `GITHUB_EVENT_PATH` and parse the event to determine the target issue.

This enables the existing NanoClaw routing and message loop to work with GitHub Issues transparently, without special-casing in the orchestrator.

---

### Request 21: Add optional maintainer control for global memory

Add a mechanism for maintainers to update the global memory file (`.github-nanoclaw/state/groups/global/CLAUDE.md`) via a dedicated control issue.

Implementation:

1. In the workflow, detect if the issue has a specific label (e.g., `nanoclaw-global-memory`).
2. If the label is present AND the actor has `admin` or `maintain` permission, allow the agent to write to the global memory file.
3. For all other issues, the global memory file is read-only.
4. Document this behavior in `.github-nanoclaw/README.md`.

---

### Request 22: Add GitHub-native scheduled task model (optional)

Create a bridge between NanoClaw's scheduled task system and GitHub Actions cron workflows.

Implementation:

1. Create `.github-nanoclaw/lifecycle/schedule-bridge.ts` that reads `scheduled_tasks` from the NanoClaw database and generates/updates cron workflow files.
2. Create a template workflow `.github-nanoclaw/install/github-nanoclaw-scheduled.yml` with a `workflow_dispatch` trigger and a `schedule` trigger.
3. The bridge maps NanoClaw cron expressions to GitHub Actions cron syntax.
4. Scheduled runs execute the same one-shot agent entrypoint but with the task prompt instead of an issue message.
5. Results are posted to a designated tracking issue or logged in state.

---

### Request 23: Final validation and acceptance testing

Run the complete acceptance criteria check against the implementation:

1. **Verify issue trigger**: Create a test issue payload for `issues.opened`. Run the workflow logic locally (mocked). Verify the agent is invoked and a response is generated.

2. **Verify conversation resume**: Run two sequential comment events for the same issue. Verify the second run has access to the first run's conversation state.

3. **Verify issue isolation**: Run events for issue #1 and issue #2. Verify their state directories, session files, and memory files are completely separate.

4. **Verify authorization gate**: Run with a read-only user payload. Verify execution is blocked and a 👎 reaction is added.

5. **Verify state persistence**: Run an agent, commit state, then run a second event. Verify the committed state is loaded correctly.

6. **Verify concurrency handling**: Verify the workflow concurrency group is correctly scoped per issue number so parallel runs on different issues don't conflict.

7. **Verify sentinel disable**: Delete `.github-nanoclaw/github-nanoclaw-ENABLED.md` and run an event. Verify execution halts immediately with an appropriate message.

Run all structural tests (`vitest run .github-nanoclaw/tests/`). Ensure all pass. Run the existing NanoClaw test suite (`npm test`) and ensure no regressions.
