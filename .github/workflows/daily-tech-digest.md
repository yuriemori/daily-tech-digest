---
emoji: 📰
name: Daily Tech Digest
description: Create a Japanese daily digest issue for GitHub, DevOps, DevSecOps, and platform updates.
# スケジュール変更ポイント: JST は通年 UTC+09:00。08:00 JST に相当する 23:00 UTC で実行。
on:
  schedule:
    - cron: "0 23 * * *"
  workflow_dispatch:
    inputs:
      dry_run:
        description: "Preview only; use noop instead of creating an issue."
        type: boolean
        default: true
      lookback_hours:
        description: "Fallback lookback window when no previous successful run exists."
        type: number
        default: 24
permissions:
  contents: read
  issues: read
  actions: read
  copilot-requests: write
strict: true
network:
  allowed:
    - defaults
    - github
    - github.blog
    - devblogs.microsoft.com
    - www.microsoft.com
    - microsoft.com
tools:
  github:
    mode: gh-proxy
    toolsets: [repos, issues, actions, search]
  web-fetch:
  cli-proxy: true
  bash: [gh, jq, date, cat, grep, sed, awk, printf, mkdir, ls]
safe-outputs:
  create-issue:
    labels: [daily-digest, japanese-summary, github-updates]
    deduplicate-by-title: true
    max: 1
---

# Daily Tech Digest

Create one Japanese daily digest issue for `yuriemori/daily-tech-digest` only when there are qualifying new updates.

## Trigger context

- Scheduled runs execute daily at 08:00 JST.
- Manual runs support `dry_run` for preview. When `dry_run` is `true`, do not create an issue; call `noop` with a concise preview summary instead.
- Manual runs may set `lookback_hours` as the fallback window only when no previous successful run can be found.

## Editable configuration comments

- Source URL変更ポイント: collect from `https://github.blog/changelog/` first, then `https://github.blog/` excluding changelog URLs, `https://devblogs.microsoft.com/`, and `https://www.microsoft.com/en-us/security/blog/`.
- 優先度キーワード変更ポイント: administrator-oriented keywords are `enterprise`, `org policy`, `audit log`, `SSO`, `compliance`, `secret protection`, `governance`, `enterprise managed users`, `rulesets`, `branch protection`, `security settings`, and `billing/admin controls`.
- Issueテンプレート変更ポイント: the required title and body structure is defined in the "Issue format" section below.
- スケジュール変更ポイント: the frontmatter cron is UTC; keep `0 23 * * *` for 08:00 JST (JST is always UTC+09:00).

## Required setup behavior

This workflow is designed to run with the repository `GITHUB_TOKEN`, read-only agent permissions, and the configured `create-issue` safe output. The agent job remains read-only; the generated safe-output job performs issue creation with its own scoped `issues: write` permission. If a first run discovers missing or unusable authentication, token permissions, safe-output capability, or required labels/secrets, do not fail silently. Instead, create one help issue with:

- title: `Daily Tech Digest Setup Help - YYYY-MM-DD (JST)`
- the missing item and observed error
- setup steps to enable GitHub Agentic Workflows on the default branch, allow `actions: read` and `issues: read` for the agent job, and allow the generated safe-output job to use scoped `issues: write` for creating issues with labels `daily-digest`, `japanese-summary`, `github-updates`

If issue creation itself is unavailable, call `noop` and include the same setup guidance in the noop explanation.

## Collection and window

1. Determine the reporting window:
   - Prefer the timestamp of the previous successful run of this workflow before the current run.
   - Use `gh` read commands/API calls against the current repository to inspect workflow runs. Exclude the current run.
   - If there is no previous successful run, use `lookback_hours` for manual dispatch or 24 hours for scheduled runs.
2. Read up to the newest 45 digest issues from the last 60 days with label `daily-digest` and titles matching `Daily Tech Digest -` to collect already-published source URLs. To keep runs bounded, do not fetch every historical issue body up front; after collecting candidate URLs from the sources, batch URL searches against all digest issue bodies where possible, and fall back to individual URL searches only for candidates not covered by the recent digest set. Treat source URL as the primary deduplication key.
3. Use past digest issue bodies as the source of truth for deduplication. Do not persist separate processed-URL state before issue creation, because a failed safe-output write must not mark unpublished items as processed.
4. Fetch the source pages and identify updates published after the window start and before the run time. Resolve timestamps in this priority order: RSS/Atom feed timestamp, page metadata such as `article:published_time`, then visible source listing date. If no reliable timestamp is available, exclude the item unless the source listing shows a date inside the window and the item is not already present in past digest issues.
5. For `https://github.blog/`, exclude URLs whose path starts with `/changelog/` because changelog items are collected from the highest-priority GitHub Changelog source first.
6. Skip any source URL already included in a previous digest issue.
7. If there are no new qualifying updates, call `noop` with the window and reason. Do not create an issue.
8. Before creating an issue, search for an existing issue titled `Daily Tech Digest - YYYY-MM-DD (JST)`. If one exists, call `noop` and reference it instead of creating a duplicate.

## Priority logic

Classify and order items as follows:

1. Highest priority: GitHub administrator updates. Match administrator intent using the keywords listed above, case-insensitively, in the title, summary, URL, or source content.
2. DevSecOps / Platform Protection.
3. DevOps / Platform Engineering.
4. General developer news.

Set impact scope to one or more of `Admin`, `Platform`, `Security`, `Dev`. Set importance to `High`, `Medium`, or `Low` based on admin/security impact, rollout urgency, and action required.

## Issue format

Create exactly one issue for days with updates. Title:

`Daily Tech Digest - YYYY-MM-DD (JST)`

Body must be Japanese GitHub-flavored Markdown and follow this structure:

```markdown
## エグゼクティブサマリー
- プリセールスで再利用しやすいビジネス価値・影響を、箇条書き3〜5点で簡潔にまとめる。

## 管理者向けハイライト
- 管理者向け該当項目がある場合は最重要順に記載。
- ない場合は「該当なし」。

## カテゴリ別サマリー

### GitHub Changelog
- **タイトル**: ...
  - **日本語要約**: 2〜4行
  - **影響範囲**: Admin / Platform / Security / Dev
  - **重要度**: High / Medium / Low
  - **原文URL**: https://...

### GitHub Blog
該当なし

### Microsoft DevBlogs
該当なし

### Microsoft Security Blog
該当なし

## メタデータ
- 対象期間: YYYY-MM-DD HH:mm JST 〜 YYYY-MM-DD HH:mm JST（UTCの実行時刻・前回成功時刻をJST、通年UTC+09:00に変換して表示）
- 重複排除キー: 原文URL
- 収集ソース: GitHub Changelog / GitHub Blog / Microsoft DevBlogs / Microsoft Security Blog
```

Use `該当なし` for empty categories. Keep the executive summary first and technical details later. Do not invent facts; cite the original URL for each item.

## Safe outputs

- Use `create-issue` only for the final digest issue or the setup-help issue.
- Use `noop` when this is a dry run, no updates were found, a same-day digest already exists, or issue creation cannot be used.
