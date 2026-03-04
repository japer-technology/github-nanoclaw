# AI Agent Requests to Complete NanoClaw v2

### Precise order of AI Agent requests to fully implement the [NanoClaw v2 Specification](https://github.com/japer-technology/githubification/blob/main/.githubification/specification/nanoclaw-v2.md)

Each request below is a self-contained prompt for an AI coding agent (e.g., Claude Code, GitHub Copilot). Execute them in order. Each request builds on the output of the previous ones. The specification source is `japer-technology/githubification/.githubification/specification/nanoclaw-v2.md`.

---

## Phase 1 — Core Agent Loop

### Request 1: Create the `.githubification/` folder structure and `package.json`

> Create the `.githubification/` directory structure for the NanoClaw v2 Githubification layer. Create the following directories and files:
>
> ```
> .githubification/
> ├── lifecycle/           # (empty — scripts added next)
> ├── state/
> │   ├── issues/          # Per-issue session mappings
> │   ├── sessions/        # Agent session transcripts (JSONL)
> │   ├── groups/
> │   │   ├── global/
> │   │   │   └── CLAUDE.md    # Shared global memory (initialize with: "# Global Memory\n\nShared context accessible by all issues.\n")
> │   │   └── .gitkeep
> │   ├── task-logs/       # Task execution history
> │   ├── router-state.json    # Initialize as: {}
> │   └── scheduled-tasks.json # Initialize as: []
> └── package.json
> ```
>
> The `package.json` should contain:
> - `name`: `"nanoclaw-githubification"`
> - `private`: `true`
> - `type`: `"module"`
> - `dependencies`: `{ "@anthropic-ai/claude-code": "latest", "cron-parser": "^4.9.0" }` (the Claude Agent SDK runtime and cron expression parser for scheduled tasks)
>
> Add `.gitkeep` files to empty directories (`state/issues/`, `state/sessions/`, `state/task-logs/`, `state/groups/global/`) so they are tracked by Git. Do NOT add `.gitkeep` to directories that already contain files.

---

### Request 2: Write `lifecycle/indicator.ts` — processing indicator

> Create `.githubification/lifecycle/indicator.ts`. This TypeScript script runs as a GitHub Actions step to provide immediate feedback when a workflow is triggered.
>
> **Behavior:**
> 1. Read `GITHUB_EVENT_PATH` to get the event payload JSON.
> 2. Determine the event type from `GITHUB_EVENT_NAME` environment variable.
> 3. If the event is `issue_comment`: add a 🚀 (rocket) reaction to the comment using the GitHub API (`POST /repos/{owner}/{repo}/issues/comments/{comment_id}/reactions` with `content: "rocket"`).
> 4. If the event is `issues` (new issue opened): add a 🚀 reaction to the issue using the GitHub API (`POST /repos/{owner}/{repo}/issues/{issue_number}/reactions` with `content: "rocket"`).
> 5. Use the `GITHUB_TOKEN` environment variable for authentication.
> 6. Use `fetch()` (built into Bun) for HTTP requests. Set headers: `Authorization: token ${GITHUB_TOKEN}`, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`.
> 7. Extract `owner` and `repo` from `GITHUB_REPOSITORY` environment variable (format: `owner/repo`).
> 8. Log what reaction was added and to what. If the API call fails, log the error but do NOT throw — the workflow should continue even if the reaction fails.

---

### Request 3: Write `lifecycle/agent.ts` — main agent execution script

> Create `.githubification/lifecycle/agent.ts`. This is the main lifecycle script that reads the GitHub issue/comment context, invokes the Claude agent, posts the reply as an issue comment, and commits state changes to Git.
>
> **Behavior:**
>
> **1. Read event context:**
> - Parse `GITHUB_EVENT_PATH` for the event payload.
> - Extract `issue.number`, `issue.title`, `issue.body`, and (if `issue_comment` event) `comment.body` and `comment.user.login`.
> - Extract `repository.full_name` and `repository.default_branch` from the payload.
> - Read `GITHUB_EVENT_NAME`, `GITHUB_TOKEN`, `ANTHROPIC_API_KEY` from environment.
>
> **2. Gather conversation context:**
> - Use the GitHub API to fetch all comments on the issue: `GET /repos/{owner}/{repo}/issues/{issue_number}/comments` (paginate if needed).
> - Format messages using XML format:
>   ```xml
>   <messages>
>   <message sender="{username}" time="{ISO8601}">{text}</message>
>   ...
>   </messages>
>   ```
> - The first message is the issue body (sender = issue author, time = issue created_at).
> - Subsequent messages are comments, in chronological order.
> - Filter out comments from bots (where `user.login` ends with `[bot]`).
>
> **3. Load or create session:**
> - Check if `.githubification/state/issues/{issue_number}.json` exists.
> - If it exists, read the `sessionFile` path from it.
> - If it does not exist, create a new session mapping:
>   ```json
>   {
>     "issueNumber": <number>,
>     "sessionFile": "sessions/<ISO8601-timestamp>.jsonl",
>     "createdAt": "<ISO8601>",
>     "lastActivity": "<ISO8601>",
>     "groupFolder": "github_issue-<number>"
>   }
>   ```
> - Write the mapping to `.githubification/state/issues/{issue_number}.json`.
>
> **4. Ensure per-issue group folder:**
> - Create `.githubification/state/groups/github_issue-{issue_number}/` if it does not exist.
> - Create a `CLAUDE.md` inside it if it does not exist, initialized with: `# Issue #{issue_number} Memory\n\nConversation context for issue #{issue_number}.\n`
>
> **5. Invoke the agent:**
> - Use `child_process.spawn` to run the Claude agent (try `npx claude` or `npx pi` — whichever is available):
>   ```
>   npx claude --session-id <sessionId-derived-from-issue-number> \
>              --prompt "<formatted-messages>" \
>              --output-format json \
>              --max-turns 25
>   ```
> - Set `ANTHROPIC_API_KEY` in the child process environment.
> - Set the working directory to `.githubification/state/groups/github_issue-{issue_number}/`.
> - Capture stdout. Parse the JSON output to extract the agent's final text response.
>
> **6. Post the reply:**
> - Post the agent's response as a comment on the issue: `POST /repos/{owner}/{repo}/issues/{issue_number}/comments` with `body: <agent_response>`.
> - If the response exceeds `NANOCLAW_MAX_COMMENT_LENGTH` (default 60000), truncate with a note: `\n\n---\n*Response truncated (exceeded maximum comment length)*`.
>
> **7. Update session state:**
> - Update `lastActivity` in the issue state JSON.
> - Write the updated state to `.githubification/state/issues/{issue_number}.json`.
>
> **8. Add outcome reaction:**
> - If the agent succeeded: add a 👍 reaction to the triggering comment (or issue if it was a new issue).
> - If the agent failed: add a 👎 reaction instead.
>
> **9. Commit and push state:**
> - Run: `git add .githubification/state/`
> - Run: `git -c user.name="nanoclaw[bot]" -c user.email="nanoclaw[bot]@users.noreply.github.com" commit -m "nanoclaw: update state for issue #${issue_number}" --allow-empty`
> - Implement push-with-retry: attempt `git push` up to 10 times. On failure, run `git pull --rebase -X theirs` before retrying. Use exponential backoff (starting at 1 second, doubling each retry).
> - If all retries fail, log the error but do NOT throw — the agent response was already posted as a comment.

---

### Request 4: Create the GitHub Actions workflow file

> Create `.github/workflows/nanoclaw-agent.yml` with the following exact content:
>
> ```yaml
> name: nanoclaw-agent
>
> on:
>   issues:
>     types: [opened]
>   issue_comment:
>     types: [created]
>
> permissions:
>   contents: write
>   issues: write
>
> jobs:
>   run-agent:
>     runs-on: ubuntu-latest
>     concurrency:
>       group: nanoclaw-${{ github.repository }}-issue-${{ github.event.issue.number }}
>       cancel-in-progress: false
>     if: >-
>       (github.event_name == 'issues')
>       || (github.event_name == 'issue_comment'
>           && !endsWith(github.event.comment.user.login, '[bot]'))
>     steps:
>       - name: Authorize
>         id: authorize
>         env:
>           GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
>         run: |
>           PERM=$(gh api "repos/${{ github.repository }}/collaborators/${{ github.actor }}/permission" \
>             --jq '.permission' 2>/dev/null || echo "none")
>           if [[ "$PERM" != "admin" && "$PERM" != "maintain" && "$PERM" != "write" ]]; then
>             echo "::error::Unauthorized: ${{ github.actor }} has '$PERM' permission"
>             exit 1
>           fi
>
>       - name: Reject unauthorized
>         if: ${{ failure() && steps.authorize.outcome == 'failure' }}
>         env:
>           GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
>         run: |
>           if [[ "${{ github.event_name }}" == "issue_comment" ]]; then
>             gh api "repos/${{ github.repository }}/issues/comments/${{ github.event.comment.id }}/reactions" \
>               -f content=-1
>           else
>             gh api "repos/${{ github.repository }}/issues/${{ github.event.issue.number }}/reactions" \
>               -f content=-1
>           fi
>
>       - name: Checkout
>         uses: actions/checkout@v4
>         with:
>           ref: ${{ github.event.repository.default_branch }}
>           fetch-depth: 0
>
>       - name: Setup runtime
>         uses: oven-sh/setup-bun@v2
>         with:
>           bun-version: latest
>
>       - name: Indicate processing
>         env:
>           GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
>         run: bun .githubification/lifecycle/indicator.ts
>
>       - name: Install dependencies
>         run: cd .githubification && bun install
>
>       - name: Run agent
>         env:
>           ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
>           GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
>         run: bun .githubification/lifecycle/agent.ts
> ```
>
> This workflow:
> - Triggers on new issues and new issue comments.
> - Uses per-issue concurrency groups with `cancel-in-progress: false` (queues rather than cancels).
> - Checks collaborator permissions (admin, maintain, or write required) — rejects with 👎 reaction.
> - Filters out bot comments in the `if` condition.
> - Uses Bun as the runtime (fast TypeScript execution without compilation).
> - Injects `ANTHROPIC_API_KEY` and `GITHUB_TOKEN` as environment variables.

---

### Request 5: Initialize state directories with `.gitkeep` and run `bun install`

> Ensure all `.githubification/state/` subdirectories are tracked by Git:
>
> 1. Add `.gitkeep` to: `state/issues/`, `state/sessions/`, `state/task-logs/`.
> 2. Ensure `state/groups/global/CLAUDE.md` exists (created in Request 1).
> 3. Run `cd .githubification && bun install` to generate `bun.lock` from the `package.json`.
> 4. Verify the folder structure matches:
>    ```
>    .githubification/
>    ├── lifecycle/
>    │   ├── indicator.ts
>    │   └── agent.ts
>    ├── state/
>    │   ├── issues/.gitkeep
>    │   ├── sessions/.gitkeep
>    │   ├── groups/
>    │   │   └── global/
>    │   │       └── CLAUDE.md
>    │   ├── task-logs/.gitkeep
>    │   ├── router-state.json
>    │   └── scheduled-tasks.json
>    ├── package.json
>    └── bun.lock
>    ```
> 5. Commit all files.

---

## Phase 2 — Session Continuity

### Request 6: Implement issue-to-session mapping and session resume

> Update `.githubification/lifecycle/agent.ts` to implement full session continuity across multiple comments on the same issue.
>
> **Changes:**
>
> 1. **Session ID derivation:** Derive a stable session ID from the issue number. Use format: `nanoclaw-issue-{issue_number}`. This ID is passed to the Claude agent via `--session-id` so the agent resumes the same conversation across comments.
>
> 2. **Session file tracking:** When creating a new session mapping in `state/issues/{N}.json`, set `sessionFile` to `sessions/{ISO8601-timestamp}.jsonl`. On subsequent invocations, read the existing `sessionFile` path.
>
> 3. **Pass `--resume` / `--session-id` to agent:** When invoking the Claude CLI:
>    - If a session already exists (state file found): pass `--resume --session-id nanoclaw-issue-{issue_number}`
>    - If new session: pass `--session-id nanoclaw-issue-{issue_number}`
>
> 4. **Commit session transcripts:** After the agent completes, `git add` any new/modified files in `.githubification/state/sessions/` along with the issue state JSON.
>
> 5. **Handle session compaction:** Claude Agent SDK automatically compacts long sessions. No special handling needed — just ensure the session directory is committed after each run.
>
> 6. **Update `lastActivity`:** Always update `lastActivity` in the issue state JSON to the current ISO8601 timestamp after each agent run.

---

## Phase 3 — Memory System

### Request 7: Implement per-issue and global memory with proper hierarchy

> Update `.githubification/lifecycle/agent.ts` to implement NanoClaw's three-level memory hierarchy.
>
> **Changes:**
>
> 1. **Global memory:** Ensure `.githubification/state/groups/global/CLAUDE.md` exists before every agent run. This file is readable by all issue agents.
>
> 2. **Per-issue memory:** Ensure `.githubification/state/groups/github_issue-{N}/CLAUDE.md` exists before every agent run. Initialize with issue-specific context if creating for the first time:
>    ```markdown
>    # Issue #{N} Memory
>
>    Conversation context for issue #{N}.
>    ```
>
> 3. **Agent working directory:** Set the agent's working directory to `.githubification/state/groups/github_issue-{N}/` so that Claude Agent SDK's project-level settings source loads `./CLAUDE.md` as the per-issue memory.
>
> 4. **Global memory access:** Before running the agent, symlink or copy `.githubification/state/groups/global/CLAUDE.md` into the agent's working directory as `../global/CLAUDE.md` (creating the relative directory structure so the agent can read global memory). The simplest approach: ensure the directory layout naturally provides this — since global is a sibling of `github_issue-{N}/` under `state/groups/`, the path `../global/CLAUDE.md` already resolves correctly.
>
> 5. **Commit memory changes:** After the agent run, `git add .githubification/state/groups/` to capture any memory file modifications the agent made.
>
> 6. **Admin-only global memory writes:** Add a check: if the agent modified `.githubification/state/groups/global/CLAUDE.md`, verify the triggering user has `admin` or `maintain` permission. If not, revert the global memory change with `git checkout -- .githubification/state/groups/global/CLAUDE.md` before committing, and post a note in the issue comment: "⚠️ Global memory changes require admin/maintainer permissions and were reverted."

---

## Phase 4 — Scheduled Tasks

### Request 8: Add scheduled task support to the workflow

> Update `.github/workflows/nanoclaw-agent.yml` to add a `schedule` trigger for checking due tasks and a `workflow_dispatch` trigger for manual testing:
>
> ```yaml
> on:
>   issues:
>     types: [opened]
>   issue_comment:
>     types: [created]
>   schedule:
>     - cron: '*/5 * * * *'
>   workflow_dispatch:
>     inputs:
>       issue_number:
>         description: 'Issue number to process'
>         required: true
>         type: number
>       prompt:
>         description: 'Override prompt (optional)'
>         required: false
>         type: string
> ```
>
> Add a conditional step at the beginning of the `run-agent` job:
> - If the trigger is `schedule`, run `.githubification/lifecycle/scheduler.ts` instead of `agent.ts`.
> - If the trigger is `workflow_dispatch`, use the `inputs.issue_number` and optional `inputs.prompt` to invoke the agent for that specific issue.
> - The existing `issues` and `issue_comment` triggers continue to run `agent.ts` as before.
>
> Update the `if` condition on the job to also allow `schedule` and `workflow_dispatch`:
> ```yaml
> if: >-
>   (github.event_name == 'schedule')
>   || (github.event_name == 'workflow_dispatch')
>   || (github.event_name == 'issues')
>   || (github.event_name == 'issue_comment'
>       && !endsWith(github.event.comment.user.login, '[bot]'))
> ```
>
> For the `schedule` trigger, remove the concurrency group (scheduled runs should not be per-issue):
> Use a separate job or a conditional concurrency group:
> ```yaml
> concurrency:
>   group: ${{ github.event_name == 'schedule' && 'nanoclaw-scheduler' || format('nanoclaw-{0}-issue-{1}', github.repository, github.event.issue.number) }}
>   cancel-in-progress: false
> ```

---

### Request 9: Create `lifecycle/scheduler.ts` — scheduled task runner

> Create `.githubification/lifecycle/scheduler.ts`. This script checks for due scheduled tasks and executes them.
>
> **Behavior:**
>
> 1. **Read task state:** Parse `.githubification/state/scheduled-tasks.json`. This is a JSON array of task objects:
>    ```json
>    {
>      "id": "task-<timestamp>-<random>",
>      "groupFolder": "github_issue-<N>",
>      "chatJid": "gh:<N>",
>      "prompt": "<task prompt text>",
>      "schedule_type": "cron" | "interval" | "once",
>      "schedule_value": "<cron expression>" | "<milliseconds>" | "<ISO8601 timestamp>",
>      "status": "active" | "paused" | "completed",
>      "next_run": "<ISO8601>",
>      "created_at": "<ISO8601>"
>    }
>    ```
>
> 2. **Find due tasks:** Filter tasks where `status === "active"` and `next_run <= new Date().toISOString()`.
>
> 3. **Execute each due task:**
>    - For each due task, invoke the Claude agent with the task's `prompt` as the message.
>    - Set the agent's working directory to `.githubification/state/groups/{groupFolder}/`.
>    - Use the task's `chatJid` to determine the issue number (parse from `gh:<N>` format).
>    - Post the agent's response as a comment on that issue with a prefix: `🔔 **Scheduled Task:** {task.prompt}\n\n---\n\n{agent_response}`.
>
> 4. **Update `next_run`:**
>    - For `cron` type: calculate the next occurrence after now using the cron expression (use `cron-parser` from the `.githubification/package.json` dependencies).
>    - For `interval` type: set `next_run` to `new Date(Date.now() + parseInt(schedule_value)).toISOString()`.
>    - For `once` type: set `status` to `"completed"` and `next_run` to `null`.
>
> 5. **Log execution:** Write a log entry to `.githubification/state/task-logs/{task.id}-{ISO8601}.json`:
>    ```json
>    {
>      "taskId": "<id>",
>      "executedAt": "<ISO8601>",
>      "status": "success" | "error",
>      "issueNumber": <N>,
>      "error": "<error message if failed>"
>    }
>    ```
>
> 6. **Save state:** Write updated tasks back to `scheduled-tasks.json`.
>
> 7. **Commit and push:** Use the same push-with-retry logic from `agent.ts`:
>    - `git add .githubification/state/`
>    - `git commit -m "nanoclaw: scheduled task run"`
>    - Push with retry (up to 10 times with exponential backoff and `git pull --rebase -X theirs` on conflict).
>
> 8. **No-op on empty:** If no tasks are due, exit cleanly without committing.

---

### Request 10: Add task creation support to the agent

> Update `.githubification/lifecycle/agent.ts` to detect when the agent creates or modifies scheduled tasks.
>
> **Behavior:**
>
> 1. **Post-agent task detection:** After the agent completes, check if `.githubification/state/scheduled-tasks.json` was modified by the agent (compare before/after content).
>
> 2. **Task validation:** If the agent wrote new tasks, validate each task object has the required fields: `id`, `groupFolder`, `chatJid`, `prompt`, `schedule_type`, `schedule_value`, `status`, `next_run`, `created_at`. If any required field is missing, log a warning and remove the invalid task.
>
> 3. **Bind tasks to issue:** When a task is created from an issue conversation, automatically set `groupFolder` to `github_issue-{issue_number}` and `chatJid` to `gh:{issue_number}`.
>
> 4. **Confirmation:** If new tasks were added, append to the agent's reply comment: `\n\n---\n📅 *Scheduled task created: "{task.prompt}" ({task.schedule_type}: {task.schedule_value})*` for each new task.

---

## Phase 5 — GitHub Issues Channel Adapter

### Request 11: Create the GitHub Issues channel adapter

> Create `src/channels/github-issues.ts` implementing the NanoClaw `Channel` interface. This channel adapter allows NanoClaw to recognize GitHub Issues as a first-class messaging channel.
>
> **Implementation:**
>
> ```typescript
> import { registerChannel } from './registry.js';
> import type { Channel, ChannelOpts } from '../types.js';
>
> class GitHubIssuesChannel implements Channel {
>   name = 'github-issues';
>
>   constructor(private opts: ChannelOpts) {}
>
>   async connect(): Promise<void> {
>     // No-op — stateless, runs in GitHub Actions context
>   }
>
>   async sendMessage(jid: string, text: string): Promise<void> {
>     // Parse issue number from JID format "gh:<number>"
>     const issueNumber = jid.replace(/^gh:/, '');
>     const [owner, repo] = (process.env.GITHUB_REPOSITORY || '').split('/');
>     const token = process.env.GITHUB_TOKEN;
>     if (!owner || !repo || !token) {
>       throw new Error('Missing GITHUB_REPOSITORY or GITHUB_TOKEN');
>     }
>
>     // Post comment via GitHub API
>     await fetch(`https://api.github.com/repos/${owner}/${repo}/issues/${issueNumber}/comments`, {
>       method: 'POST',
>       headers: {
>         'Authorization': `token ${token}`,
>         'Accept': 'application/vnd.github+json',
>         'X-GitHub-Api-Version': '2022-11-28',
>       },
>       body: JSON.stringify({ body: text }),
>     });
>   }
>
>   isConnected(): boolean {
>     // Always true when running in GitHub Actions
>     return !!process.env.GITHUB_ACTIONS;
>   }
>
>   ownsJid(jid: string): boolean {
>     return jid.startsWith('gh:');
>   }
>
>   async disconnect(): Promise<void> {
>     // No-op
>   }
>
>   async setTyping(jid: string, isTyping: boolean): Promise<void> {
>     // When isTyping=true, add 🚀 reaction; when false, no action needed
>     // (reaction removal is not necessary — it persists as a "processing started" indicator)
>     if (!isTyping) return;
>     const issueNumber = jid.replace(/^gh:/, '');
>     const [owner, repo] = (process.env.GITHUB_REPOSITORY || '').split('/');
>     const token = process.env.GITHUB_TOKEN;
>     if (!owner || !repo || !token) return;
>
>     await fetch(`https://api.github.com/repos/${owner}/${repo}/issues/${issueNumber}/reactions`, {
>       method: 'POST',
>       headers: {
>         'Authorization': `token ${token}`,
>         'Accept': 'application/vnd.github+json',
>         'X-GitHub-Api-Version': '2022-11-28',
>       },
>       body: JSON.stringify({ content: 'rocket' }),
>     }).catch(() => {}); // Non-critical
>   }
> }
>
> registerChannel('github-issues', (opts) => {
>   // Only activate when running in GitHub Actions
>   if (!process.env.GITHUB_ACTIONS) return null;
>   return new GitHubIssuesChannel(opts);
> });
> ```
>
> This follows the same factory self-registration pattern as all other NanoClaw channels. The channel returns `null` from its factory when not running in GitHub Actions, so it has zero impact on local operation.

---

### Request 12: Register the GitHub Issues channel in the barrel file

> Edit `src/channels/index.ts` to import the GitHub Issues channel so it self-registers at startup.
>
> Add the following import line alongside the existing channel imports:
>
> ```typescript
> import './github-issues.js';
> ```
>
> This import triggers the `registerChannel('github-issues', ...)` call at module load time, making the channel available to the NanoClaw orchestrator when running in GitHub Actions.

---

### Request 13: Add JID format `gh:<issue_number>` support

> Verify and update the following files to support the `gh:<issue_number>` JID format used by the GitHub Issues channel:
>
> 1. **`src/router.ts`:** Ensure `routeOutbound()` and `findChannel()` work with `gh:` JIDs. Since these functions iterate over connected channels and call `ownsJid()`, and the GitHubIssuesChannel.ownsJid() handles `gh:` prefix, no changes should be needed — but verify.
>
> 2. **`src/db.ts`:** Ensure the `chats` and `messages` tables accept `gh:` JIDs. Since JIDs are stored as strings, no schema changes should be needed — but verify that no validation regex rejects the format.
>
> 3. **`src/index.ts`:** Ensure the main orchestrator loop can process groups with `gh:` JIDs. Verify that `registerGroup()`, `processGroupMessages()`, and `runAgent()` work correctly with this JID format.
>
> Make any necessary changes to support the `gh:` JID format. If no changes are needed, confirm compatibility by inspection.

---

## Phase 6 — Skills and Polish

### Request 14: Create the `/githubify` skill

> Create the NanoClaw skill that packages Githubification for easy installation. Create the following files:
>
> **`.claude/skills/githubify/SKILL.md`:**
>
> ````markdown
> # Githubify
>
> Add GitHub Issues as a NanoClaw channel and deploy the agent on GitHub Actions.
>
> ## What This Skill Does
>
> - Creates the `.githubification/` directory with lifecycle scripts and state management
> - Adds `.github/workflows/nanoclaw-agent.yml` for GitHub Actions execution
> - Registers a GitHub Issues channel adapter (`src/channels/github-issues.ts`)
> - Enables conversation via GitHub Issues with session continuity and memory
> - Supports scheduled tasks via cron-triggered workflow runs
>
> ## Prerequisites
>
> - `ANTHROPIC_API_KEY` configured as a GitHub Secret
> - Repository collaborator permissions (write or above) for users who will interact with the agent
>
> ## Usage
>
> After applying this skill:
>
> 1. Push the repository to GitHub
> 2. Set `ANTHROPIC_API_KEY` in repository Settings → Secrets → Actions
> 3. Open an issue — the agent responds automatically
> 4. Comment on the issue — the agent continues the conversation
>
> ## Compatibility
>
> This skill coexists with local NanoClaw channels (WhatsApp, Telegram, Slack, etc.). The same installation can run locally via `npm run dev` AND on GitHub via Issues simultaneously.
> ````
>
> This skill follows the existing NanoClaw skill pattern (see `.claude/skills/add-whatsapp/` for reference).

---

### Request 15: Write `.githubification/README.md` — user documentation

> Create `.githubification/README.md` with comprehensive user documentation:
>
> ````markdown
> # NanoClaw v2 — GitHub Issues Channel
>
> NanoClaw v2 runs your personal Claude assistant on GitHub Actions, using GitHub Issues as the conversation interface.
>
> ## Quick Start
>
> 1. **Set up secrets:** Go to repository Settings → Secrets → Actions. Add:
>    - `ANTHROPIC_API_KEY`: Your Anthropic API key
>
> 2. **Open an issue:** Create a new issue in this repository. The agent will respond with a comment.
>
> 3. **Continue the conversation:** Add comments to the issue. The agent maintains conversation context across comments.
>
> ## How It Works
>
> - **Trigger:** New issues and issue comments trigger a GitHub Actions workflow
> - **Authorization:** Only repository collaborators (write permission or above) can interact with the agent
> - **Concurrency:** Each issue has its own concurrency group — messages are queued, not cancelled
> - **Memory:** The agent maintains per-issue memory in `.githubification/state/groups/github_issue-{N}/CLAUDE.md`
> - **Sessions:** Conversation sessions are tracked in `.githubification/state/issues/{N}.json`
>
> ## Scheduled Tasks
>
> The agent can schedule recurring tasks. The workflow checks for due tasks every 5 minutes via a cron trigger.
> Task state is stored in `.githubification/state/scheduled-tasks.json`.
>
> ## Configuration
>
> | Secret / Variable | Default | Purpose |
> |---|---|---|
> | `ANTHROPIC_API_KEY` | (required) | Claude API authentication |
> | `CLAUDE_CODE_OAUTH_TOKEN` | (optional) | Alternative to API key |
> | `ASSISTANT_NAME` | `Andy` | Name used in responses |
> | `NANOCLAW_MAX_COMMENT_LENGTH` | `60000` | Max characters per reply comment |
>
> ## Architecture
>
> See the full specification: [nanoclaw-v2.md](https://github.com/japer-technology/githubification/blob/main/.githubification/specification/nanoclaw-v2.md)
>
> ## Folder Structure
>
> ```
> .githubification/
> ├── lifecycle/           # Workflow step scripts
> │   ├── indicator.ts     # Adds 🚀 reaction on trigger
> │   ├── agent.ts         # Main agent loop
> │   └── scheduler.ts     # Scheduled task runner
> ├── state/               # Git-committed state (replaces SQLite)
> │   ├── issues/          # Per-issue session mappings
> │   ├── sessions/        # Agent session transcripts
> │   ├── groups/          # Memory hierarchy
> │   ├── scheduled-tasks.json
> │   └── router-state.json
> ├── package.json         # Runtime dependencies
> └── README.md            # This file
> ```
> ````

---

### Request 16: Add outcome reactions (👍/👎) for success/failure

> Review `.githubification/lifecycle/agent.ts` and ensure outcome reactions are correctly implemented:
>
> 1. **On success:** After the agent responds and the reply is posted, add a 👍 (`+1`) reaction:
>    - If triggered by `issue_comment`: react to the comment (`POST /repos/{owner}/{repo}/issues/comments/{comment_id}/reactions` with `content: "+1"`)
>    - If triggered by `issues` (new issue): react to the issue (`POST /repos/{owner}/{repo}/issues/{issue_number}/reactions` with `content: "+1"`)
>
> 2. **On failure:** If the agent invocation fails or the reply fails to post, add a 👎 (`-1`) reaction to the same target.
>
> 3. **Error handling:** Wrap the entire agent invocation and reply posting in a try/catch. In the catch block, add the 👎 reaction, and also post an error comment on the issue: `"❌ **Agent Error:** {error.message}"`. Reaction failures should be caught silently (don't fail the workflow if a reaction can't be added).

---

### Request 17: Add label-based issue filtering (optional enhancement)

> Add optional label-based filtering to the workflow and lifecycle scripts:
>
> 1. **Workflow filter:** Add an optional environment variable `NANOCLAW_ISSUE_LABEL` (default: empty, meaning all issues are processed). If set, the workflow should check if the issue has that label before running the agent.
>
> 2. **Implementation in `agent.ts`:** At the beginning of the script, after parsing the event:
>    ```typescript
>    const requiredLabel = process.env.NANOCLAW_ISSUE_LABEL;
>    if (requiredLabel) {
>      const labels = event.issue.labels.map((l: any) => l.name);
>      if (!labels.includes(requiredLabel)) {
>        console.log(`Skipping: issue #${issueNumber} does not have label "${requiredLabel}"`);
>        process.exit(0);
>      }
>    }
>    ```
>
> 3. **Documentation:** Add this to `.githubification/README.md` under Configuration:
>    ```
>    | `NANOCLAW_ISSUE_LABEL` | (none) | Only respond to issues with this label |
>    ```

---

### Request 18: Integrate with skills engine state tracking

> Update the NanoClaw skills engine to recognize the Githubification skill:
>
> 1. **Create `.claude/skills/githubify/manifest.yaml`:**
>    ```yaml
>    name: githubify
>    version: 2.0.0
>    description: Add GitHub Issues as a channel with GitHub Actions execution
>    dependencies: []
>    channels:
>      - github-issues
>    ```
>
> 2. **Skill file listing:** Create the `add/` directory in the skill with placeholder references to the files this skill creates:
>    - `.githubification/lifecycle/indicator.ts`
>    - `.githubification/lifecycle/agent.ts`
>    - `.githubification/lifecycle/scheduler.ts`
>    - `.githubification/state/` (directory structure)
>    - `.githubification/package.json`
>    - `.githubification/README.md`
>    - `.github/workflows/nanoclaw-agent.yml`
>
> 3. **Channel registration modification:** Create `modify/` directory with the patch for `src/channels/index.ts` (adding the `import './github-issues.js'` line).
>
> 4. **Channel implementation:** Include `src/channels/github-issues.ts` in the skill's `add/` directory.
>
> This ensures the skills engine can track, merge, and uninstall the Githubification layer cleanly.

---

### Request 19: Final integration test and verification

> Perform a final verification pass across all created files:
>
> 1. **TypeScript compilation:** Run `npx tsc --noEmit` from the repository root to verify `src/channels/github-issues.ts` compiles without errors against the existing NanoClaw type system.
>
> 2. **Lifecycle scripts:** Verify `.githubification/lifecycle/indicator.ts`, `agent.ts`, and `scheduler.ts` have no syntax errors by running `bun check` or `npx tsc --noEmit` on them (they use Bun APIs, so may need separate tsconfig).
>
> 3. **Workflow YAML validation:** Verify `.github/workflows/nanoclaw-agent.yml` is valid YAML and all GitHub Actions expressions (`${{ }}`) are syntactically correct.
>
> 4. **State files:** Verify `scheduled-tasks.json` contains `[]` and `router-state.json` contains `{}`.
>
> 5. **Git tracking:** Verify all `.gitkeep` files are in place and all state directories will be tracked by Git.
>
> 6. **Channel compatibility:** Verify `src/channels/github-issues.ts` returns `null` when `GITHUB_ACTIONS` is not set (zero impact on local operation).
>
> 7. **Documentation completeness:** Verify `.githubification/README.md` covers all configuration options, secrets, and usage instructions.
>
> 8. **Specification alignment:** Cross-reference every section of the [NanoClaw v2 specification](https://github.com/japer-technology/githubification/blob/main/.githubification/specification/nanoclaw-v2.md) against the implementation to confirm full coverage:
>    - Architecture Mapping ✓
>    - Workflow Design ✓
>    - State Management ✓
>    - Channel Adaptation ✓
>    - Agent Execution Model ✓
>    - Memory System ✓
>    - Session Continuity ✓
>    - Scheduled Tasks ✓
>    - Security Model ✓
>    - Skills Integration ✓
>    - Folder Structure ✓
>    - Configuration ✓
>    - Implementation Phases 1–6 ✓

---

## Summary

| Request | Phase | Creates / Modifies | Specification Section |
|---|---|---|---|
| 1 | Phase 1 | `.githubification/` folder structure, `package.json` | Folder Structure, State Management |
| 2 | Phase 1 | `lifecycle/indicator.ts` | Workflow Design (Indicate step) |
| 3 | Phase 1 | `lifecycle/agent.ts` | Workflow Design (Run step), Agent Execution, State Management |
| 4 | Phase 1 | `.github/workflows/nanoclaw-agent.yml` | Workflow Design, Security Model |
| 5 | Phase 1 | State directory initialization, `bun.lock` | Folder Structure |
| 6 | Phase 2 | Updates `lifecycle/agent.ts` | Session Continuity |
| 7 | Phase 3 | Updates `lifecycle/agent.ts` | Memory System |
| 8 | Phase 4 | Updates workflow YAML | Scheduled Tasks, Workflow Design |
| 9 | Phase 4 | `lifecycle/scheduler.ts` | Scheduled Tasks |
| 10 | Phase 4 | Updates `lifecycle/agent.ts` | Scheduled Tasks |
| 11 | Phase 5 | `src/channels/github-issues.ts` | Channel Adaptation |
| 12 | Phase 5 | Updates `src/channels/index.ts` | Channel Adaptation |
| 13 | Phase 5 | Verifies `src/router.ts`, `src/db.ts`, `src/index.ts` | Channel Adaptation |
| 14 | Phase 6 | `.claude/skills/githubify/SKILL.md` | Skills Integration |
| 15 | Phase 6 | `.githubification/README.md` | Configuration, Folder Structure |
| 16 | Phase 6 | Updates `lifecycle/agent.ts` | Workflow Design (Outcome step) |
| 17 | Phase 6 | Updates `lifecycle/agent.ts`, workflow, README | Configuration |
| 18 | Phase 6 | `.claude/skills/githubify/manifest.yaml`, skill directories | Skills Integration |
| 19 | Phase 6 | Verification pass (no new files) | All sections |
