# SPEC.md — Daily Tech Digest Agentic Workflow 仕様書

関連Issue: https://github.com/yuriemori/daily-tech-digest/issues/1

---

## 背景

このリポジトリは、GitHub / DevOps / DevSecOps / Platform Engineering / Platform Protection に関するアップデートを日次で収集し、日本語で要約したダイジェストを管理するためのものです。

利用者は Microsoft の Software Solution Engineer（Developer Platform 専門の Pre-sales エンジニア）であり、顧客提案・社内共有・技術キャッチアップに再利用しやすい形式で情報を整理することを目的とします。

---

## 目的

GitHub Agentic Workflows を利用して、指定した情報源を毎日確認し、新規アップデートが存在する場合のみ GitHub Issue を 1 件作成します。

**重視する観点:**

- GitHub 管理者向けアップデートの優先抽出
- DevSecOps / Platform Protection 関連情報の把握
- DevOps / Platform Engineering 関連情報の把握
- Pre-sales 活動で再利用しやすいビジネス要約
- 必要に応じて深掘りできる技術詳細

---

## 参照仕様

GitHub Agentic Workflows の作成方法として以下を参照すること:

- https://raw.githubusercontent.com/github/gh-aw/main/create.md

---

## 実行スケジュール

| 項目 | 値 |
|------|-----|
| 定期実行 | 毎日 08:00 JST |
| 手動実行 | 有効（`workflow_dispatch`） |
| UTC cron | `0 23 * * *` |

---

## 収集対象ソース

| 優先度 | ソース名 | URL | 備考 |
|--------|----------|-----|------|
| 1（最優先） | GitHub Changelog | https://github.blog/changelog/ | GitHub 管理者向け更新を最優先で確認 |
| 2 | GitHub Blog | https://github.blog/ | — |
| 3 | Microsoft DevBlogs | https://devblogs.microsoft.com/ | — |
| 4 | Microsoft Security Blog | https://www.microsoft.com/en-us/security/blog/ | — |

> **変更方法:** ソースURLは設定ファイルまたはワークフロー先頭のパラメータとして定義し、ここを変更するだけで監視対象を追加・削除できるようにする。

---

## 対象期間

- 固定の 24 時間ではなく「**前回成功実行以降**」を対象とする
- 前回実行が失敗していた場合は、最後に成功した実行時刻を基準にする
- 初回実行時は安全な既定値（例: 直近 24 時間または直近 3 日間）を設定し、変更しやすくする

### 状態の永続化

実行時刻および処理済み URL の状態は以下のいずれかで管理する（実装時に選択）:

| 方式 | 説明 | 備考 |
|------|------|------|
| リポジトリファイル（推奨） | `data/state.json` をリポジトリにコミット | 履歴追跡が容易。`contents: write` 権限が追加で必要 |
| GitHub Actions キャッシュ | `actions/cache` を利用 | 権限追加不要だがキャッシュ消失リスクあり |
| GitHub Actions アーティファクト | アーティファクトとして保存 | 明示的に管理しやすい |

推奨はリポジトリファイル方式とし、追加権限（`contents: write`）が必要な場合はその旨をワークフロー定義にコメントで明記すること。

---

## 重複排除

- 過去にダイジェスト Issue へ含めた記事は再掲しない
- **重複判定の優先順位:**
  1. 原文 URL
  2. canonical URL
  3. タイトル + 公開日
- 新規更新が 0 件の場合は Issue を作成しない

### 状態ファイルの保持ポリシー

処理済み URL を記録する状態ファイルが無制限に肥大化するのを防ぐため、以下の保持ポリシーを適用する:

- 直近 **30 日分** の処理済み URL のみを保持する（デフォルト値。変更しやすい設定値として外出しすること）
- 30 日より古いエントリは次回実行時に自動的に削除する
- 前回実行時刻は別フィールドで管理し、URL の保持期間とは独立して扱う

---

## 優先度ロジック

優先度は以下の順とする:

1. GitHub 管理者向けアップデート
2. DevSecOps / Platform Protection
3. DevOps / Platform Engineering
4. 一般開発者向けニュース

### GitHub 管理者向け判定キーワード（初期案）

> **変更方法:** 以下のキーワードリストは設定ファイルで管理し、追加・変更が容易な構成にする。

```
enterprise
org policy
audit log
SSO
compliance
secret protection
governance
enterprise managed users
rulesets
branch protection
security settings
billing/admin controls
```

### 重要度の定義

| 重要度 | 説明 |
|--------|------|
| **High** | 管理者、セキュリティ、ガバナンス、コンプライアンス、組織運用に影響するもの。GitHub Enterprise / Organization 管理者が対応を検討すべきもの。 |
| **Medium** | DevOps / DevSecOps / Platform Engineering の実務に影響するもの。開発チームやプラットフォームチームに共有すべきもの。 |
| **Low** | 一般的な技術記事、参考情報、影響範囲が限定的なもの。 |

### 影響範囲の定義

| 影響範囲 | 対象 |
|----------|------|
| Admin | GitHub Enterprise / Organization 管理者 |
| Platform | プラットフォームエンジニアリングチーム |
| Security | セキュリティ・コンプライアンス担当 |
| Dev | 一般開発者・開発チーム |

---

## 出力言語とトーン

- Issue 本文は**日本語**
- **2 段構成:**
  1. エグゼクティブ / ビジネス要約（Pre-sales で再利用しやすい、3〜5 行程度）
  2. 技術詳細（記事ごとの要約、影響範囲、重要度、原文 URL）

---

## Issue 作成仕様

### 基本ルール

- 更新がある日のみ **1 日 1 Issue** を作成
- 全ソースの更新を **1 つの Issue に統合**
- 更新がない場合は Issue を作成しない

### タイトル形式

```
Daily Tech Digest - YYYY-MM-DD (JST)
```

### 本文構成

```markdown
## エグゼクティブサマリー
<!-- Pre-sales で再利用しやすい、3〜5 行のビジネス要約 -->

---

## 管理者向けハイライト
<!-- 最優先。GitHub Enterprise / Organization 管理者が対応すべき更新 -->

---

## カテゴリ別サマリー

### GitHub Changelog
<!-- 該当なしの場合は「該当なし」と記載 -->

#### [記事タイトル]
- **日本語要約:** （2〜4 行）
- **影響範囲:** Admin / Platform / Security / Dev
- **重要度:** High / Medium / Low
- **原文URL:** https://...

### GitHub Blog
<!-- 該当なしの場合は「該当なし」と記載 -->

### Microsoft DevBlogs
<!-- 該当なしの場合は「該当なし」と記載 -->

### Microsoft Security Blog
<!-- 該当なしの場合は「該当なし」と記載 -->
```

### Issue ラベル

以下のラベルを付与する:

- `daily-digest`
- `japanese-summary`
- `github-updates`

---

## 失敗時の扱い

| 状況 | 対応 |
|------|------|
| 更新なし | Issue を作成せず正常終了 |
| 収集元の一部失敗 | **最低 1 ソース以上**が取得成功している場合、取得できた結果で Issue を作成する。ただし優先度最高の「GitHub Changelog」が失敗した場合は、その旨を Issue 本文の冒頭に明記する。失敗したソース・理由・再試行要否を Issue 本文に記録する。全ソースが失敗した場合は Issue を作成しない。 |
| 必須設定不足 | secret / token / permission が不足している場合は、黙って失敗させずセットアップ手順を明記したヘルプ Issue を作成する |

---

## 権限とセキュリティ

最小権限で実行すること。想定権限:

| 権限 | レベル |
|------|--------|
| `contents` | `read` |
| `issues` | `write` |
| `actions` | `read` |

> 状態ファイル更新などで追加権限が必要な場合は、理由を明記して必要最小限にする。

---

## 保守性

以下の項目を変更しやすい構成にすること:

| 変更対象 | 対応方法 |
|----------|----------|
| 監視対象ソース URL | 設定ファイルまたはワークフロー先頭のパラメータで管理 |
| 優先度判定キーワード | 設定ファイルで一元管理 |
| Issue テンプレート | テンプレートファイルを別ファイルで管理 |
| 実行スケジュール | ワークフロー定義の cron 式を変更するだけで対応可能 |
| 初回実行時の対象期間 | 設定値として外出しし、変更しやすくする |

---

## 実装成果物

`gh-aw` の慣例に沿って、動作に必要なファイル一式を作成する。

| ファイル | 説明 |
|----------|------|
| Agentic Workflow 定義ファイル | ワークフローのトリガー・権限・実行手順を定義 |
| ワークフロー実行用 prompt / spec ファイル | Copilot への指示内容を記載 |
| 設定ファイル | ソース URL、キーワード、スケジュール等のパラメータ |
| 状態管理ファイル | 前回成功実行時刻・処理済み URL の記録 |
| README または運用メモ | セットアップ手順・運用ガイドを記載 |

---

## 完了条件

以下をすべて満たすこと:

- [ ] デフォルトブランチで Agentic Workflow を有効化できる
- [ ] 手動実行で動作確認できる
- [ ] 対象期間が前回成功実行以降になっている
- [ ] 過去 Issue との重複排除が機能する
- [ ] 更新なしの日は Issue が作成されない
- [ ] 更新ありの日は指定フォーマットの Issue が 1 件作成される
- [ ] Issue 本文が日本語で生成される
- [ ] 管理者向けアップデートが優先的に表示される
