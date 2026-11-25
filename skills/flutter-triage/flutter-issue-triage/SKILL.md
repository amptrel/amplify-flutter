---
name: flutter-issue-triage
description: GitHub Issue Triage skill for Amplify Flutter (aws-amplify/amplify-flutter), generated from the aws-oss-issue-triage-skeleton
---

# Amplify Flutter Issue Triage Skill

Generated from the AWS Open-Source Issue Triage SOP (Skeleton). This skill guides the agent through five phases to triage a GitHub issue filed against Amplify Flutter (`aws-amplify/amplify-flutter`): (1) Gather Context, (2) Source Code Investigation, (3) Trusted Docs Investigation, (4) Reproduction, and (5) Triage Report Generation. The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", "MAY" are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

Assumptions:
- A working GitHub tool is available (GitHub MCP tools or `gh` CLI); GitHub access is not elaborated here.
- There is no agent-toolkit-for-aws core skill for Amplify Flutter. Domain knowledge comes from the repository checkout (`AGENTS.md`, package READMEs and CHANGELOGs under `packages/`), https://docs.amplify.aws/flutter/ and https://pub.dev/publishers/aws-amplify.com/packages.
- The agent instructions it was started with (the Trellis workflow) define which repository is read, which is written to, and how labels and the response are applied. Where those instructions and this skill differ, the agent instructions win.

## 0. Scope and Principles

- Single issue per run: triage directly, no subagents.
- MUST validate all tool parameters before execution; SHOULD keep a short log of tool actions for the report.
- The repository checkout in the working directory is read-only context. MUST NOT run `git fetch`, `git pull`, `git checkout`, `git switch`, `git reset`, `git clean` or any other command that changes it or talks to the network. `git log`, `git show`, `git grep` and `git blame` on what is already there are fine.
- Nothing posted to GitHub MAY reference an issue, pull request or discussion in any repository: no `#N`, no `owner/repo#N`, no issue, pull request or discussion URL. Describe related work in words ("an open pull request already adds this option", "a separate report tracks the same crash"). CHANGELOG entries and commit titles in this repository end in `(#N)`; strip that before quoting them. The report MAY keep the numbers in inline code.
- No identity lookups (`GET /user`, `gh api user`, viewer queries); nothing in the triage needs them.
- Voice: the response and the report are written in the first person singular ("I checked", "I could not reproduce", "I have labeled"). MUST NOT use "we", "our team" or "the team" for the triager's own actions.

## Phase 1: Gather Context

Goal: build a complete picture of the issue before touching code.

### 1.1 Fetch Issue

- MUST retrieve the full issue with all comments (`github_issue_read`, method `get` and `get_comments`).
- MUST identify: category (Auth, Storage, API GraphQL, API REST, Analytics, DataStore, Push Notifications, Authenticator, Core, Secure Storage, Codegen, Build), the `amplify_*` package and version (the "Amplify Flutter Version" field), Flutter and Dart SDK versions, platform(s) (iOS, Android, Web, macOS, Windows, Linux), deployment method (Amplify Gen 2 `amplify_outputs.dart`, Amplify CLI Gen 1 `amplifyconfiguration.dart`, CDK, custom), and the build mode when relevant (debug vs release, R8/ProGuard on Android).
- The bug template (`.github/ISSUE_TEMPLATE/BUG-REPORT.yaml`) has Description, Categories, Steps to Reproduce, Screenshots, Platforms, Flutter Version, Amplify Flutter Version, Deployment Method, Schema. A report written with the template usually has every field; a free-form report usually lacks versions and platform, so check each explicitly.

### 1.2 Project Profile (Amplify Flutter)

- **Repo**: `aws-amplify/amplify-flutter`, default branch `main`; `stable` is the release branch. Monorepo managed by the in-repo `aft` tool (`packages/aft`, components declared under `aft:` in the root `pubspec.yaml`), not melos. Packages live under `packages/<category>/<package>`:
  - Auth: `packages/auth/amplify_auth_cognito` (Flutter plugin; Hosted UI platform code under `android/` and `darwin/`) and `packages/auth/amplify_auth_cognito_dart` (Dart implementation: state machines, flows, credentials, JWT, generated Cognito SDK under `lib/src/sdk`). Most Auth logic is in the `_dart` package.
  - Storage: `packages/storage/amplify_storage_s3` and `amplify_storage_s3_dart` (S3 transfer tasks, options, exceptions).
  - API: `packages/api/amplify_api` and `amplify_api_dart` (GraphQL, REST, subscriptions).
  - Analytics: `packages/analytics/amplify_analytics_pinpoint` and `_dart`.
  - DataStore: `packages/amplify_datastore` (Flutter plugin over native code: on Android it depends on `com.amplifyframework:aws-datastore` from Amplify Android; on iOS and macOS the Swift sources live under `darwin/`), plus `packages/amplify_datastore_plugin_interface`.
  - Push notifications: `packages/notifications/push/amplify_push_notifications` and `amplify_push_notifications_pinpoint`.
  - Authenticator UI: `packages/authenticator/amplify_authenticator`.
  - Secure storage: `packages/secure_storage/amplify_secure_storage` and `_dart`.
  - Core: `packages/amplify_core` (categories, exceptions, config, `AmplifyOutputs`), `packages/amplify/amplify_flutter` (the `Amplify` entry point), `packages/aws_common`, `packages/aws_signature_v4`, `packages/smithy`.
  - Conventions: `AGENTS.md` at the repo root (state machines, `AmplifyException` vs `AmplifyError`, `_dart` split, `.g.dart` generated files).
- **Sibling repos where root cause may live**: `aws-amplify/amplify-android` and `aws-amplify/amplify-swift` (native code behind DataStore and parts of Auth; labels `requires-android-fix` / `requires-ios-fix`), `aws-amplify/amplify-backend` (Gen 2 backend and `amplify_outputs.json`), `aws-amplify/amplify-cli` (Gen 1 config; `requires-cli-fix`), `aws-amplify/amplify-codegen` (`ModelProvider.dart` and model generation; `requires-codegen-fix`), `aws-amplify/amplify-js` (feature parity reference; `feature-parity`). SHOULD note in the report when the fix belongs in a sibling repo.
- **Labels** (exact names on the repository being written to; verify with `github_list_label` if unsure):
  - Queue: `pending-triage`.
  - Classification: `bug`, `feature-request`, `documentation`, `question`.
  - Category: `auth`, `core`, `storage`, `analytics`, `datastore`, `GraphQL API`, `REST API`, `Authenticator`, `Secure Storage`, `push notifications`, `Codegen`, `Build`.
  - Platform: `iOS`, `Android`, `Web`, `macOS`, `Linux`, `windows` (lowercase w).
  - Status: `pending-community-response`, `pending-maintainer-response`, `pending-close-response-required`, `to-be-reproduced`, `not-reproducible`, `pending-release`, `found-in-release-candidate`, `fixed-in-release-candidate`, `duplicate`, `platform-discrepancy`.
  - Cross-repo: `requires-android-fix`, `requires-ios-fix`, `requires-cli-fix`, `requires-codegen-fix`, `requires-api-data-fix`, `feature-parity`.
  - There are **no priority labels** (no `p1`, `p2`). Severity is stated in prose (5.4).
  - Repository automation: adding `bug`, `question` or `feature-request` removes `pending-triage` (`.github/workflows/issue_labeled.yml`); a community comment removes `pending-community-response` and adds `pending-maintainer-response`, a maintainer comment removes `pending-maintainer-response` (`issue_comment.yml`). The triager MUST NOT add `pending-maintainer-response`.
- **Release model**: every package is published to pub.dev under the `aws-amplify.com` publisher. The "Amplify Flutter" component (`amplify_flutter`, `amplify_core`, `amplify_auth_cognito`, `amplify_storage_s3`, `amplify_api`, `amplify_analytics_pinpoint`, `amplify_datastore`, push notifications) is versioned in unison; `_dart` packages, `amplify_authenticator`, `amplify_secure_storage`, `aws_common`, `smithy` and `worker_bee` have their own lines. Each package has a `CHANGELOG.md` generated by `aft version-bump` from conventional-commit titles (`fix(auth): ...`, `feat(storage): ...`); a release may list only "Minor bug fixes and improvements". Tags: `<package>-v<version>` per package (for example `amplify_auth_cognito-v2.16.0`, `aws_common-v0.7.16`); releases up to 2.15.0 also carry a component tag `v<version>` (`v2.15.0`), 2.16.0 does not. Look for the per-package tag first (`github_list_tags` or the matching-refs API). To map a fix to a shipping release: find the fix commit on `main`, then the first tag containing it (`github_list_tags`, `github_get_release_by_tag`) or the CHANGELOG entry; if it is on `main` but in no tag yet, it is unreleased (`pending-release`).

### 1.3 Fast-Path Classification

MUST classify into a track before deep analysis. The track names are the output values.

| Track | Criteria | Action |
|-------|----------|--------|
| `already-fixed` | Fix merged on `main` (or shipped) | Ack, name the fix in words, name the shipping version or say it is unreleased |
| `duplicate` | Same root cause as an existing open issue, same change would fix both | Describe the original in words, recommend `duplicate` label |
| `feature-request-with-pr` | A pull request already implements the request | Ack, label `feature-request`, skip deep analysis |
| `security` | Vulnerability in a declared or bundled dependency | Verify the published pub.dev package, point to the AWS vulnerability reporting page, no exploit details |
| `build-time-bug` | Fails at `flutter pub get`, `dart analyze`, `flutter build`, Gradle or CocoaPods/SwiftPM resolution, no device needed | Check constraints in the package `pubspec.yaml`, `android/build.gradle`, `darwin/*.podspec` or `Package.swift`; label `Build` |
| `missing-context` | Sufficiency checklist fails (1.5) | Post an info request, stop analysis |
| `full-triage` | None of the above | Execute all phases |

### 1.4 Duplicate Search

- MUST search `repo:aws-amplify/amplify-flutter` with `github_search_issues`, `perPage: 3`, using the exception name, API name or symptom words from the report.
- A duplicate shares the same root cause and is resolved by the same change; MUST trace both to the cause before declaring it. Same symptom is not enough.
- When the symptom is platform-specific native behaviour (Hosted UI on Android or iOS, DataStore sync, push tokens), SHOULD also search the matching native repo (`repo:aws-amplify/amplify-android` or `repo:aws-amplify/amplify-swift`).

### 1.5 Context Sufficiency Gate

- Behavioral bug: problem description, repro steps or minimal Dart snippet, expected vs actual (error message, exception type, log lines), `amplify_*` version, Flutter version, platform. All present?
- Security/CVE: affected package, advisory ID or vulnerable version, reported `amplify_*` version. Repro steps NOT required.
- Feature request: the gap and the desired behaviour are enough; versions are optional.
- If insufficient: MUST NOT proceed to Phase 2 for root cause. The track is `missing-context`, classification stays unconfirmed (see 5.4 and 5.3 for labels and the response).

## Phase 2: Source Code Investigation

Goal: locate root cause in the actual shipped/HEAD source, not in assumptions.

### 2.1 Locate Relevant Code

- The working directory checkout can be searched with the shell (`grep -rn`, `rg` if present) and read with `read`; prefer it over remote search for speed. For remote search use `github_search_code` with `repo:aws-amplify/amplify-flutter` and `perPage: 3`, then `github_get_file_contents`.
- Map the category to the package per 1.2; for Auth and Storage start in the `_dart` package. Native symptoms: look under `android/src/main/kotlin` or `darwin/` of the Flutter plugin, then the sibling native repo.

### 2.2 Version-Accurate Analysis

- MUST analyze the source at the tag matching the reported version (`<package>-v<version>`, else `v<version>`; `github_get_file_contents` with `ref`), then compare against `main`.
- Ground-truth hierarchy for "does X exist / is X supported" claims, stop at the first definitive answer:
  1. The published package on pub.dev (`https://pub.dev/packages/<package>/versions/<version>`, `https://pub.dev/documentation/<package>/<version>/`) reflects what users actually run.
  2. GitHub source at the matching tag or commit.
  3. AWS service documentation for the underlying service (Cognito, S3, AppSync, Pinpoint).
  4. https://docs.amplify.aws/flutter/ (rendered, can lag or predate the version). A gap here is inconclusive, never proof of absence.
- MUST NOT conclude "not supported" from source (4) alone; an unconfirmed finding MUST be labeled as such and name which higher source would settle it.

### 2.3 Fix Status Verification

- MUST check whether the issue is already fixed on `main` before proposing a fix: search the package CHANGELOG (unreleased section, if any) and `github_list_commits` with `path` set to the package directory, then compare the fixed-in version with the reported version.
- A fix on `main` that is in no tag yet is `pending-release`; a fix in a tag newer than the reported version means the user should upgrade.
- For CVEs: the fixed dependency version MUST be read from the published package's `pubspec.yaml` on pub.dev, not from the repo lockfile or constraint.

### 2.4 Root Cause Statement

- MUST state root cause with code snippets and source permalinks to `aws-amplify/amplify-flutter` at a commit SHA. Resolve the SHA first: for `main`, `github_list_commits` (owner `aws-amplify`, repo `amplify-flutter`, `perPage: 1`); for a release, the tag's commit (`github_get_tag` or `github_get_release_by_tag`). Confirm the file exists at that SHA (`github_get_file_contents` with `ref`) before linking.
- The working directory HEAD is a fork commit, not an upstream commit. MUST NOT build permalinks from `git rev-parse HEAD` or any local SHA, even though GitHub may resolve such a URL (it serves fork commits through the shared network).
- If root cause cannot be determined: MUST say so explicitly, identify the most likely area, and suggest next investigation steps. No speculation presented as fact.

## Phase 3: Trusted Docs Investigation

Goal: validate intended behavior against authoritative documentation, in trust order.

### 3.1 Trust Hierarchy (Amplify Flutter)

1. AWS service documentation (docs.aws.amazon.com: Cognito, S3, AppSync, Pinpoint) and the service API references.
2. In-repo docs: package READMEs and dartdoc comments under `packages/<category>/<package>/lib`, `AGENTS.md`, `CONTRIBUTING.md`, RFC issues labeled `RFC`.
3. Package CHANGELOGs and GitHub release notes.
4. https://docs.amplify.aws/flutter/ (lowest; see 2.2).

Community sources (blog posts, Stack Overflow, forum answers) MAY inform hypotheses but MUST NOT be cited as ground truth.

### 3.2 Intended vs Actual Behavior

- MUST determine whether the reported behavior is: a bug (docs or dartdoc contract contradict behavior), a doc gap (behavior correct, docs unclear or wrong), or working-as-intended (user expectation mismatch, for example a Cognito or S3 service limit).
- MUST capture the doc URLs used for the determination.

### 3.3 Feature Requests and Known Limitations

- SHOULD check for documented limitations (package README support tables, docs.amplify.aws platform notes) and open `RFC` issues that explain the behaviour.
- For a feature request: check whether the capability exists in the underlying service and in another Amplify platform (`amplify-js`, `amplify-android`, `amplify-swift`); if it does, recommend `feature-parity` in addition to `feature-request`. State an existing workaround explicitly when one exists (plugin options, `pluginOptions`, the generated service clients under `lib/src/sdk` of the `_dart` package, or a custom `AWSHttpClient` from `aws_common`).
- If the request needs a service or native-SDK capability that does not exist, name that upstream dependency as the blocker.

## Phase 4: Reproduction

Goal: empirically validate the issue and the root cause hypothesis formed in Phases 2 and 3.

### 4.1 When to Reproduce

- Reproduction needs the Flutter SDK. Check once with `flutter --version`. If `flutter` is not on PATH, the outcome is **Not attempted** and the comment MUST say verbatim: "I did not attempt a reproduction: the triage environment has no Flutter SDK." Source-only analysis (Phase 2) then stands in, and the comment MUST say that the findings come from reading the source.
- NOT required for: security/CVE issues (verify the published package instead), pure feature requests, doc gaps, already-fixed issues, and `missing-context`.
- Even with Flutter present, anything needing a device, emulator, browser, AWS backend, sign-in or the `amplify` CLI is out of scope; record **Not attempted** with that reason.

### 4.2 Minimal Reproduction (Flutter recipe)

- Only when `flutter` is available and the issue gives a small self-contained snippet: in a folder under `$TMPDIR`, `flutter create --platforms=<platform named in the report, else android> repro`, add the reported `amplify_*` packages at the reported versions to `pubspec.yaml`, paste the snippet into `lib/main.dart`, run `flutter pub get` and `flutter analyze`; `dart test` when the snippet is plain Dart. SHOULD also try the latest release to detect already-fixed status.
- Cheapest sufficient level, in order: `dart analyze` (API shape, missing members, type errors), `dart test` on pure Dart logic, `flutter build` for build-time bugs. No `flutter run`, no deploy.
- Stop after about five minutes and record **Not attempted** with the reason.

### 4.3 Validate Against Hypothesis

- MUST record expected vs actual output and confirm it matches the reporter's claim.
- MUST check observed behavior against the Phase 2 root cause and Phase 3 intended behavior; if they disagree, MUST revisit the earlier phase rather than force the conclusion.
- Outcome MUST be one of: **Reproduced** (steps + output), **Not reproduced** (works; capture environment delta candidates: Flutter version, platform, deployment method), or **Not attempted** (with reason). MUST NOT claim a reproduction that did not happen.

### 4.4 Artifacts

- MUST capture exact commands, versions and relevant output snippets in the report. Nothing under `$TMPDIR` is kept after the run.

## Phase 5: Triage Report Generation

Goal: a self-contained report per issue and one response comment.

### 5.1 Report Delivery

- The report goes in the `report` output (Markdown). Header MUST include timestamp, issue URL, repo and reported `amplify_*` and Flutter versions.

### 5.2 Report Sections

1. **Issue Summary**: what was reported, track.
2. **Context Gathered**: category, package, versions, platform, deployment method, sufficiency gate result.
3. **Duplicate Analysis**: candidates checked (numbers in inline code), verdict, Mermaid diagram (5.5).
4. **Source Investigation**: root cause with snippets and permalinks, fix status.
5. **Docs Investigation**: intended behavior verdict with citations.
6. **Reproduction Summary**: Reproduced / Not reproduced / Not attempted, steps, output, reason if skipped.
7. **Recommended Classification**: bug / feature / docs / question / working-as-intended, severity in prose (5.4), labels set and removed.
8. **Possible Fixes/Solutions**: approaches, no implementation code.
9. **Proposed Response**: the comment, identical to what is posted.

### 5.3 Proposed Response Format

- MUST start with `Hi @<author>,` and @mention nobody else.
- MUST continue (after the greeting) with the "Issue at a glance" box in a fenced code block (5.6). Every response has it, including `missing-context`, `duplicate` and `already-fixed`.
- MUST include a reproduction status statement: "I was able to reproduce this with Flutter <version> on <platform> ..." / "I did not attempt a reproduction: the triage environment has no Flutter SDK. From the source code ..." / "I could not reproduce this ...".
- Every link to a repository file MUST be a commit-SHA permalink to `aws-amplify/amplify-flutter` (`https://github.com/aws-amplify/amplify-flutter/blob/<sha>/<path>#L<a>-L<b>`), including CHANGELOGs, READMEs and `pubspec.yaml`. `blob/main`, `blob/stable` and tag names in the path are forbidden. Symbol names MAY be markdown links; a permalink to specific lines MUST also appear bare on its own line so GitHub expands the snippet.
- MUST NOT use blockquote (`>`) formatting.
- MUST be brief and actionable: SHOULD stay under 2,500 characters; an info request SHOULD list at most 5 numbered asks, the most decisive first. Encourage a pull request when a fix is small and the user is willing (CONTRIBUTING: `aft bootstrap`, `flutter analyze`, `dart format`).
- First person singular throughout ("I checked", "I did not find", "I have labeled this"); humble phrasing ("it appears", "from what I can see"). MUST NOT use "we".
- Security-sensitive: point to https://aws.amazon.com/security/vulnerability-reporting/ and MUST NOT discuss exploit details.
- `missing-context` response: say what is missing and why it decides the next step, then the checklist (versions, platform, deployment method, minimal snippet, exact error text or logs, whether it reproduces in a fresh `flutter create` app). MUST NOT state a root cause or a classification; MAY name the area the symptom usually points to, marked as a guess.

### 5.4 Labels

- Labels are applied by the triager as the agent instructions describe (one complete label list). Rules for that list:
  - Always remove `pending-triage`.
  - `full-triage`, `already-fixed`, `build-time-bug`, `security`, `feature-request-with-pr`: add one classification label (`bug`, `feature-request`, `documentation` or `question`), the category label(s), and the platform label(s) the report names. Add `to-be-reproduced` when the report has reproduction steps that were not run. MAY also add, when the finding is certain: `pending-release` (fix on `main`, unreleased), `requires-android-fix` / `requires-ios-fix` / `requires-cli-fix` / `requires-codegen-fix` (root cause in that sibling), `feature-parity`, `platform-discrepancy`.
  - `duplicate`: add the category label and MAY add `duplicate`; no `to-be-reproduced`.
  - `missing-context`: add `pending-community-response` plus the category and platform labels the report names, and nothing else. MUST NOT add `bug`, `feature-request`, `question`, `documentation` or `to-be-reproduced`: nothing is confirmed yet. (`pending-triage` is removed with the rest, so the maintainers can find the issue by `pending-community-response`.) The `classification` output still needs a value: use `question` and say in the report that it is provisional; `priority` is `none`.
  - `pending-community-response` is also added on any other track when the response asks the author for something that blocks the next step.
  - Never add `pending-maintainer-response`, `pending-close-response-required`, `not-reproducible` (only a human who ran the steps may set it), `good first issue`, `help wanted`, `wontfix`, `invalid`.
- Severity (the `priority` output) has no label. Rubric, stated in prose in the report and, when useful, the comment:
  - `p1`: security issue, crash or data loss on a documented main path with no workaround, regression in the latest release blocking sign-in, sync or upload for all users of a platform.
  - `p2`: bugs with a workaround, platform discrepancies, build failures with a known pin, feature requests that block a documented use case with no workaround, already-fixed regressions awaiting release.
  - `none`: `missing-context`, `duplicate`, questions, doc gaps, feature requests that have a workaround.

### 5.5 Mermaid Diagram Requirements (report only)

- MUST use `flowchart TD`; MUST use `subgraph` to separate contrasting flows (for example "Reported" vs "Expected").
- MUST quote node text containing special characters; one node or connection per line.
- MUST NOT use color fills; SHOULD be dark theme friendly. Mermaid never goes in the comment.

### 5.6 "Issue at a Glance" Box

- Fenced code block (triple backticks, no language tag), solid box-drawing characters (`┌ ┐ └ ┘ ─ │ ┬ ┴ ├ ┤ ┼ ▼ ▶`).
- Exactly three stages in order: Reported State, Root Cause, Suggested Fix. Boxes the same width; content at most 60 characters per line; `│` and `▼` between boxes; no Mermaid here; no `#N` and no URLs inside the box.
- Per track, the second and third boxes read:
  - `full-triage`, `build-time-bug`: the cause (with file name and line) and the fix or workaround.
  - `missing-context`: "Not determined yet: <what is missing>" and "Next step: share the details requested below".
  - `duplicate`: "Same cause as an existing report (described below)" and "Follow the existing report; this one can be closed".
  - `already-fixed`: the cause and "Fixed in <package> <version>" or "Fixed on main, not released yet".
  - feature request: "Not a defect: <capability> is not exposed today" and the proposed API or workaround.

Template:

````
```
┌──────────────────────────────────────────┐
│ 🔴 Reported State                        │
│ <one-line description of the problem>    │
└─────────────────────┬────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────┐
│ 🔍 Root Cause                            │
│ <one-line root cause>                    │
│ <optional: source file + line ref>       │
└─────────────────────┬────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────┐
│ 🟢 Suggested Fix                         │
│ <one-line proposed fix>                  │
└──────────────────────────────────────────┘
```
````

## Troubleshooting

- No repro steps or no versions (behavioral bugs only): `missing-context` (1.5). MUST NOT attempt root cause analysis; ask with a checklist (5.3). Does NOT apply to security/CVE issues or feature requests.
- Repro not attempted or inconclusive: record the outcome per 4.3 with the verbatim reason; never claim a reproduction that did not happen.
- Root cause inconclusive: state it explicitly (2.4); identify the most likely package and why it is inconclusive.
- Noisy search: refine terms, use `repo:` scoping; MUST NOT raise `perPage` beyond 3.
- Docs-site gotcha: docs.amplify.aws may describe a newer or older API than the reported version; verify against the tag (2.2) before declaring a doc contradiction.
- CHANGELOG gotcha: "Minor bug fixes and improvements" hides the fix list; use `github_list_commits` on the package path between the two tags instead.
- Release-build gotcha (Android): symptoms only in release builds often come from R8 or ProGuard renaming (for example exception class names); check for `consumer-rules.pro` or `proguard` in the plugin's `android/` folder before calling it a logic bug.
- Native gotcha: DataStore and Hosted UI behaviour differs per platform because the native layer differs; when only one platform fails, check that platform's code under `android/` or `darwin/` and the matching native repo, and consider `platform-discrepancy`.
- Deployment gotcha: Gen 1 (`amplifyconfiguration.dart`) and Gen 2 (`amplify_outputs.dart`) configs are parsed by different code in `amplify_core`; confirm which one the report uses before reading config code.
- Fork gotcha: the checkout HEAD is not an upstream commit; permalinks come from `github_list_commits` or the tag (2.4).
