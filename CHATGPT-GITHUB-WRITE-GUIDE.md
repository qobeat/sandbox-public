# ChatGPT GitHub Write Guide

Tested in this ChatGPT environment on 2026-08-18 against:

- public repository: `qobeat/sandbox-public`
- private repository: `qobeat/sandbox-private`

This document is for future ChatGPT chats that need to read or write GitHub repositories directly.

## 1. Preferred direct path: GitHub connector

For repository-only changes, use the GitHub connector directly. Do not route through Cursor only to edit or commit GitHub files when the direct connector is sufficient.

First verify the repository and permissions with `get_repo`. For a private repository, require evidence that the connector can resolve the repository and that the returned permissions allow the intended operation (for example `push=true` for direct branch writes).

### High-level Contents API tools

Available and tested in this environment:

- `fetch_file` — read a file and obtain its current blob SHA.
- `create_file` — create a new UTF-8 text file and commit it to an existing branch.
- `update_file` — replace an existing UTF-8 text file; requires the current blob SHA.
- `delete_file` — delete an existing file; requires the current blob SHA.

Recommended sequence for one-file edits:

1. `get_repo(repository_full_name="owner/repo")`.
2. If the file exists, `fetch_file` and keep its returned SHA.
3. Use `create_file`, `update_file`, or `delete_file` as appropriate.
4. Verify with `fetch_file` and `fetch_commit`.

Important behavior:

- `create_file` cannot replace an existing path.
- `update_file` and `delete_file` require the current blob SHA, so a stale SHA can fail after a concurrent change.
- Do not run sequential writes to the same path in parallel.
- `create_file`/`update_file` operate on UTF-8 text. For binary data, prefer the Git-object path with base64 blobs when appropriate.
- There is no dedicated rename action in the checked file-tool group. A rename through the Contents API is therefore a create-new-path plus delete-old-path operation, normally producing separate commits.
- Omitting `branch` targets the repository default branch. For production repositories, resolve the intended branch explicitly when ambiguity matters.

## 2. Atomic multi-file path: Git objects

Available in this environment:

- `create_blob`
- `create_tree`
- `create_commit`
- `update_ref`
- `fetch_blob`
- `fetch_commit`
- `compare_commits`

Use this path when several files should land in one commit, or when a rename should be represented atomically in one commit.

Canonical flow:

1. Resolve the current branch HEAD commit.
2. Resolve the HEAD commit's tree SHA.
3. Create new content objects with `create_blob`.
4. Build a new tree with `create_tree(base_tree_sha=...)`, changing only requested paths. A deletion is represented in the Git tree update by removing/nulling the old path entry according to the tool contract.
5. Create one commit with `create_commit(tree_sha=..., parent_sha=<current HEAD>)`.
6. Advance the branch with `update_ref(..., force=false)`.
7. Verify the resulting commit and files.

Safety requirements:

- Always use the current HEAD as the parent; do not guess it.
- Always derive the base tree from the current commit before modifying it. Building a tree from scratch can drop unrelated files.
- Keep `force=false` unless the user explicitly requests history rewriting.
- Treat `update_ref` as the point where previously unreferenced Git objects become a visible branch change.

Test 2 on 2026-08-18 used this low-level path for the private sandbox repository to rename `tests1.txt` to this guide in one commit.

## 3. Branch + Pull Request path

Exposed GitHub tools include:

- `create_branch`
- `create_pull_request`
- `update_pull_request`
- `request_pull_request_reviewers`
- `add_review_to_pr`
- `merge_pull_request`
- `enable_auto_merge`
- `mark_pull_request_ready_for_review`
- `convert_pull_request_to_draft`
- PR comment/review-thread mutation tools

Recommended production workflow:

1. Resolve the base branch.
2. `create_branch` from an exact base ref or commit.
3. Write files on that branch using either Contents API tools or the Git-object path.
4. Verify the branch diff with `compare_commits` / PR diff tools.
5. `create_pull_request`.
6. Review status/checks as required.
7. Merge with `merge_pull_request` or enable auto-merge if repository policy allows it.

This path is preferable when branch protection, review, CI, or auditability should gate changes. Test 2 verified that these PR/merge tools are exposed, but did not create an unnecessary test PR/branch because that would leave extra repository state beyond the requested file rename.

## 4. Public versus private repositories

The direct GitHub write tools use `repository_full_name` such as `qobeat/sandbox-private`; there is not a separate private-repository write API.

Actual authorization is determined by the connected GitHub account/app. In Test 1 on 2026-08-18:

- `qobeat/sandbox-public` returned `visibility=public` and `push=true`.
- `qobeat/sandbox-private` returned `visibility=private` and `push=true`.
- `create_file` succeeded in both repositories.
- Read-back and commit verification succeeded in both repositories.

Therefore private-repository write capability must be verified per connected repository/account context, not assumed globally.

## 5. Read and verification tools useful around writes

Checked tools include:

- `get_repo` — repository metadata, default branch, visibility, permissions.
- `fetch_file` — exact file content and blob SHA.
- `search_branches` — branch discovery.
- `search_commits` — recent or matching commits.
- `fetch_commit` — commit metadata and diff.
- `compare_commits` — compare refs/commits.
- PR diff/patch tools — inspect proposed changes.
- `get_commit_combined_status` — commit status/check information.
- GitHub Actions run/job/log/artifact read tools.

Verification rule for future chats: do not claim a write succeeded solely because the mutation returned success. Read the changed file or commit back from GitHub and compare it with the requested result.

## 6. GitHub Actions mutations currently exposed

The checked `workflow` tool group exposed two write-like retry operations:

- `rerun_workflow_job`
- `rerun_failed_workflow_run_jobs`

The same checked group exposed workflow/job/log/artifact reads. It did not expose a general workflow-dispatch/start action in this environment at test time. Treat that statement as scoped to the tool registry checked on 2026-08-18, because connector capabilities can change.

## 7. Indirect local-repository path through Cursor or another coding agent

A different architecture is:

`ChatGPT -> local coding agent -> local checkout -> git commit -> git push -> GitHub`

Use that path when the requested work requires local execution: builds, tests, installed dependencies, code generation, deployment, or repository-wide refactoring that should be validated before push.

In prior project work, `gpt-cursor-mcp` provided such a path when its MCP tools were exposed. In this specific Test 2 chat, `gpt-cursor-mcp` was not exposed through the current `api_tool` connector list, and a plugin search for `gpt cursor mcp OR cursor` returned no installable result. Therefore a future chat must first verify that the Cursor/MCP tools are actually present before relying on this route.

Do not confuse this indirect local path with the direct GitHub connector. Direct GitHub writes do not require a local checkout.

## 8. Notable issues and edge cases

1. **Stale SHA races.** `update_file` and `delete_file` can fail if another commit changed the path after `fetch_file`; fetch again rather than forcing an old SHA.
2. **Rename semantics.** The high-level file API has no checked rename action, so create+delete is not atomic. Use the Git-object path for an atomic multi-path commit.
3. **Branch protection.** A connector may have `push=true` while repository rules still require a PR/checks for a particular branch. Prefer the PR path when repository policy is important.
4. **Private-repository authorization.** Never infer private access from public access. Verify the exact repository.
5. **Default-branch drift.** Do not hard-code `main` unless it has been resolved from repository metadata or supplied by the user.
6. **Direct `fetch` contract discrepancy.** Its tool description says it fetches approved public GitHub resources. During Test 2 it successfully fetched a private repository Git-commit REST URL through the authenticated connector. Because this behavior is broader than the description, do not depend on it for private reads; prefer repository-specific authenticated tools such as `fetch_file`, `fetch_commit`, and `get_repo`.
7. **Git-object safety.** `create_blob`/`create_tree`/`create_commit` create objects before `update_ref`; if ref advancement is not performed, the intended branch is unchanged. Verify the ref/commit after the update.
8. **Force updates.** Keep `update_ref(force=false)` by default. Force-moving a branch can rewrite history.

## 9. Tool-selection rule

Use the smallest path that satisfies the task:

- Single text file: `fetch_file` + `update_file`, or `create_file`.
- Delete one file: `fetch_file` + `delete_file`.
- Several files in one atomic commit: `create_blob` + `create_tree` + `create_commit` + `update_ref`.
- Review/CI-gated change: `create_branch` + writes + `create_pull_request` + review/checks + merge.
- Local build/test/deploy required: use an available coding-agent/local-repository connector, then `git push`; verify GitHub afterward.

## 10. Minimal instruction for a future ChatGPT chat

Use the GitHub connector directly. First call `get_repo` for the exact `owner/repo` and verify the intended branch and write permission. Read existing files before changing them. For one-file text changes use the Contents API tools; for atomic multi-file changes use the Git-object path; for protected/reviewed work use a branch and PR. Never force-update refs unless explicitly requested. After every write, read the resulting file/commit back from GitHub and report the exact commit SHA. For a private repository, prove access to that exact repository rather than assuming it from another repo. If local builds/tests/deployment are required, verify that a local coding-agent/MCP connector is actually exposed before using that indirect path.

## 11. Evidence from Test 1

Public Test 1 commit:

`d02dcbacd27910f898e226a9811658ea93cc2ea6`

Private Test 1 commit:

`67b57904a887c85e1e3de2b9ec1276fb02c58ae1`

Both created `tests1.txt`; Test 2 replaces that temporary test file with this permanent guide.
