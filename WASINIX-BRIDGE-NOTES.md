# Wasinix command bridge: what works and what does not

`.github/workflows/wasinix-command.yml` runs `/wasinix` comments on wasixcc
pull requests through the wasinix machinery. File and line references are
against wasix-org/wasinix `main` as of 2026-08-18.

## Verified to work cross-repo

- **Authorization** (`tools/wasinix/src/ci/origin.rs`): `authorize` accepts
  any event whose PR base repo equals the event repo (line 288-294) and whose
  owner passes `--allowed-owner` (line 48-54). Every API read it and `verify`
  (line 129-173) perform targets `origin.repository`, i.e. wasix-org/wasixcc:
  the comment read-back, the commenter's collaborator permission (line
  112-126), and the PR head. The wasixcc workflow's own `GITHUB_TOKEN` grants
  all of these, so no extra secret is needed for authorization or reporting.
- **Build commands** (`tools/wasinix/src/ci/normalize.rs`): a bare command
  defaults `--from-pr` to `current` (`types.rs` `default_from_pr`, line
  250-268), which resolves through the recorded origin (`current_pull_request`,
  line 53-63). `apply_pull_request` (line 135-184) sees a base repo that is
  not the wasinix checkout and maps it through `update_sources` (line
  105-130) onto the one update target whose `source` is
  `github:wasix-org/wasixcc`: `wasixcc`, declared in
  `pkgs/products/wasixcc/package.nix` (line 46-55, the only pin with that
  repo). The PR head sha becomes a revision override, materialized by
  `wasinix update wasixcc@rev:<sha>` in a wasinix worktree
  (`ci/workspace.rs` `materialize_overrides`, line 43-59), so the wasixcc
  commit is fetched from GitHub and never needs to exist in the wasinix
  checkout.
- **Publishing**: `ci publish` and `ci reply` resolve their repository from
  `$GITHUB_REPOSITORY` (`github/surfaces.rs` `resolve_repository`, line
  51-62), which in this workflow is wasix-org/wasixcc, so the report comment
  and check run land on the wasixcc PR.

## Wasinix-side wrinkle the workflow works around

`normalize::current_repository` (`ci/normalize.rs` line 34-36) decides "is
this PR against the repo being built" via `detected_repository`
(`github/surfaces.rs` line 66-74), which prefers `$GITHUB_REPOSITORY` over
the checkout's origin remote. In a wasixcc workflow that env names wasixcc,
so a wasixcc PR would be misread as a wasinix revision and the run would fail
at worktree creation. The `Run command` step therefore exports
`GITHUB_REPOSITORY=wasix-org/wasinix` for that one invocation. A `ci command
--repository` flag (or resolving against the checkout before the env) would
remove the workaround.

## Deliberately not bridged: mutation commands

`/wasinix update`, `bump`, and `regenerate` (kind `mutation`) rewrite the PR
branch by running the wasinix tree's update machinery in a worktree of the PR
head and pushing the result back:

- `github/mutation.rs` line 316-321 refuses any origin whose repository is
  not the repository the job represents, by design ("mutation comments are
  accepted only on {current}").
- Even without that guard, `mutate` adds a worktree of the wasinix checkout
  at the PR head sha (line 355), which for a wasixcc PR is a foreign commit,
  and the update drivers it then runs (line 357-380) operate on a wasinix
  tree, not a wasixcc one.

Cross-repo mutations would need a wasinix-side notion of "mutate the pin
override's repository", which does not exist. The workflow's
`reply-mutation` job replies to such comments instead of running them.
