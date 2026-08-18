# ChatGPT GitHub Write Guide

Last tested: 2026-08-18

Purpose: instructions for future ChatGPT chats that need to write directly to GitHub public or private repositories in this environment.

Test repositories:

- public: `qobeat/sandbox-public`
- private: `qobeat/sandbox-private`

## 1. Preferred path: direct GitHub connector

For repository-only changes, prefer the GitHub connector. A local Cursor checkout is not required just to create, replace, delete, commit, branch, review, or merge GitHub content.

Before any write:

1. Call `get_repo` for the exact `owner/repo`.
2. Resolve the default/intended branch; do not assume it is `main` unless verified or user-specified.
3. For private repositories, verify access to that exact repository and inspect returned permissions such as `push=true`.
4. Read existing target files before modifying them.
5. After every write, read the resulting file/commit back from GitHub and compare it with the requested result.

Test 1 proved direct writes to both repositories above. `get_repo` reported the public repo as `visibility=public`, the private repo as `visibility=private`, and `push=true` for both. `create_file` succeeded and was independently verified by `fetch_file` and `fetch_commit` in both repos.

## 2. High-level file path: Contents API

Checked tools:

- `fetch_file` — read content and current blob SHA.
- `create_file` — create a new UTF-8 text file on an existing branch.
- `update_file` — completely replace an existing UTF-8 text file; requires current blob SHA.
- `delete_file` — delete an existing file; requires current blob SHA.

Typical single-file edit:

1. `fetch_file`.
2. Keep the returned blob SHA.
3. `update_file(..., sha=<current blob SHA>)`.
4. `fetch_file` and `fetch_commit` to verify.

Important details:

- `create_file` cannot replace an existing path.
- `update_file` and `delete_file` require the current blob SHA. A concurrent commit can make a previously fetched SHA stale; fetch again instead of forcing it.
- Do not issue sequential writes to the same path in parallel.
- `create_file`/`update_file` are UTF-8 text operations. The low-level blob path supports `utf-8` or `base64` and is more suitable when binary/object-level handling is required.
- The checked file-tool group has no dedicated rename action. Therefore a rename through this high-level path is `create_file(new path)` + `delete_file(old path)` and normally creates two commits.

Test 2 public-repo coverage exercised `create_file`, `fetch_file`, `update_file`, and `delete_file`. It renamed the temporary `tests1.txt` into this `CHATGPT-GITHUB-WRITE-GUIDE.md` using the high-level path.

## 3. Atomic multi-file path: Git objects

Checked tools:

- `create_blob`
- `create_tree`
- `create_commit`
- `update_ref`
- `fetch_blob`
- `fetch_commit`
- `compare_commits`

Use this path when multiple path changes should land in one commit, including an atomic rename.

Canonical flow:

1. Resolve the current branch HEAD commit.
2. Resolve that commit's current tree SHA.
3. Create changed file contents with `create_blob`.
4. Call `create_tree(base_tree_sha=<current tree SHA>, tree_elements=[...])` and change only requested paths. For a rename, add the new path and delete the old path in the same tree update (Git tree deletion uses a null SHA entry for the old path).
5. `create_commit(tree_sha=<new tree>, parent_sha=<current HEAD>)`.
6. `update_ref(branch_name=<branch>, sha=<new commit>, force=false)`.
7. Verify with `fetch_commit`, `fetch_file`, and when useful `compare_commits`.

Safety rules:

- Derive both parent commit and base tree from the current branch state; do not guess them.
- Never build a replacement tree from scratch unless the complete repository tree has intentionally been enumerated; otherwise unrelated files can be dropped.
- Keep `force=false` unless the user explicitly requests history rewriting.
- `create_blob`, `create_tree`, and `create_commit` create Git objects but do not make the intended branch visibly change until `update_ref` succeeds.

Test 2 private-repo coverage used this path to replace `tests1.txt` with this guide in one commit.

## 4. Branch + Pull Request path

Exposed tools include:

- `create_branch`
- `create_pull_request`
- `update_pull_request`
- `request_pull_request_reviewers`
- `add_review_to_pr`
- `mark_pull_request_ready_for_review`
- `convert_pull_request_to_draft`
- `merge_pull_request`
- `enable_auto_merge`
- PR comment/review-thread mutation tools

Recommended production flow:

1. Resolve exact base branch/ref.
2. `create_branch`.
3. Write files on that branch using Contents API tools or Git objects.
4. Inspect changes with `compare_commits`, `fetch_pr`, `fetch_pr_patch`, or related PR diff tools.
5. `create_pull_request`.
6. Check required reviews/statuses/CI.
7. `merge_pull_request` or `enable_auto_merge` when repository policy permits.

This is preferable when branch protection, review, CI, or auditability should gate the change. Test 2 verified these tools are exposed but intentionally did not create a disposable branch/PR because that would leave additional repository state beyond the requested rename.

## 5. Public and private repository behavior

There is one set of direct GitHub repository tools; private access depends on connector authorization for the exact repo.

Do not infer private access from success against a public repo. Verify each private repository with `get_repo` and an authenticated read before writing.

Useful verification/read tools checked in this environment:

- `get_repo`
- `fetch_file`
- `search_branches`
- `search_commits`
- `fetch_commit`
- `compare_commits`
- PR diff/patch/read tools
- `get_commit_combined_status`
- GitHub Actions run/job/log/artifact read tools

## 6. GitHub Actions mutations exposed

The checked `workflow` tool group exposed these write-like retry operations:

- `rerun_workflow_job`
- `rerun_failed_workflow_run_jobs`

The same checked group exposed workflow/job/log/artifact reads. It did not expose a general workflow-dispatch/start action at test time. This negative claim is scoped only to the `workflow` tools enumerated on 2026-08-18; connector capabilities may change later.

## 7. Indirect local path through Cursor / another coding agent

Architecture:

`ChatGPT -> local coding agent -> local checkout -> git commit -> git push -> GitHub`

Use this when work requires local execution such as builds, tests, installed dependencies, code generation, deployment, or repo-wide refactoring that should be validated before push.

Prior project work used `gpt-cursor-mcp` for this kind of path when its MCP tools were exposed. In this Test 2 chat, it is not exposed in the current `api_tool` connector list, and a plugin search for `gpt cursor mcp OR cursor` returned no installable result. A future chat must therefore verify that a Cursor/MCP connector is actually present before relying on this route.

Direct GitHub writes and local-Cursor writes are different transports. Do not require Cursor when the direct GitHub connector fully satisfies the task.

## 8. Issues and edge cases

1. **Stale SHA race:** `update_file`/`delete_file` can reject a stale blob SHA after a concurrent change.
2. **High-level rename is non-atomic:** the checked Contents API route needs create + delete. Use Git objects when one atomic commit matters.
3. **Branch protection:** repository rules can still require PR/reviews/checks even when repository metadata reports push permission.
4. **Private authorization:** prove access to the exact private repository; do not generalize from another repo.
5. **Default branch:** resolve it instead of hard-coding a name.
6. **`fetch` contract discrepancy:** the `fetch` tool description says it fetches approved public GitHub resources, yet during Test 2 it successfully fetched a private repository Git-commit REST URL through the authenticated connector. Because runtime behavior is broader than the description, do not depend on generic `fetch` for private access; prefer repo-specific authenticated tools such as `get_repo`, `fetch_file`, and `fetch_commit`.
7. **Git-object safety:** object creation alone is not a branch update; verify `update_ref` and then verify the resulting commit/file.
8. **Force ref updates:** keep `force=false` by default to avoid unintended history rewriting.

## 9. Tool-selection rule

Use the smallest write path that meets the requirement:

- Create one text file: `create_file`.
- Replace one text file: `fetch_file` -> `update_file`.
- Delete one file: `fetch_file` -> `delete_file`.
- Atomic multi-file change/rename: `create_blob` -> `create_tree` -> `create_commit` -> `update_ref(force=false)`.
- Review/CI-gated change: `create_branch` -> writes -> `create_pull_request` -> checks/review -> merge.
- Local build/test/deploy required: use an actually available local coding-agent connector -> local git commit/push -> verify on GitHub.

## 10. Minimal prompt for a future ChatGPT agent

Use the GitHub connector directly. First call `get_repo` for the exact repository and verify the intended branch and write permission. Read existing target files before changing them. For one-file text changes use the Contents API tools; for atomic multi-file changes use `create_blob/create_tree/create_commit/update_ref`; for protected or reviewed work use a branch and PR. Never force-update refs unless explicitly requested. After every write, independently read the resulting file and commit back from GitHub and report the exact commit SHA. For private repositories, prove access to that exact repo. If builds/tests/deployment require a local checkout, first verify that a local coding-agent/MCP connector is actually exposed.

## 11. Test evidence

Test 1 public commit creating `tests1.txt`:

`d02dcbacd27910f898e226a9811658ea93cc2ea6`

Test 1 private commit creating `tests1.txt`:

`67b57904a887c85e1e3de2b9ec1276fb02c58ae1`

Test 2 replaced those temporary files with this permanent guide while exercising two distinct direct GitHub write architectures.

## 12. Repository-adjacent write capabilities

These operations mutate GitHub repository state but are not file-write transports.

The checked `issue` tool group exposed mutations including:

- `create_issue`
- `update_issue`
- `add_issue_assignees`
- `remove_issue_assignees`
- `add_issue_labels`
- `remove_issue_label`
- `lock_issue_conversation`
- `unlock_issue_conversation`
- `add_comment_to_issue`
- `update_issue_comment`
- issue-comment reaction add/remove operations

The checked PR group additionally exposed PR metadata/review/comment mutations described above.

Use these only when the user actually wants issue/PR state changed; they are separate from committing repository files.

Exact registry queries for `release` and `tag` returned no matching GitHub tools on 2026-08-18. Treat this only as a statement about those checked tool queries in this environment, not as a statement about GitHub's general capabilities.
