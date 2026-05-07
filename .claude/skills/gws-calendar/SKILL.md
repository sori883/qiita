---
name: gws-calendar
description: "Google Calendar: Manage calendars and events via the gws CLI."
---

# gws calendar

Google カレンダーの予定・カレンダーを `gws` CLI で操作するためのスキル。

```bash
gws calendar <resource> <method> [flags]
```

## Installation & Authentication

`gws` バイナリが `$PATH` にあること。インストール手順はプロジェクト README を参照。

```bash
# ブラウザ OAuth (interactive)
gws auth login

# Service Account
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json
```

## Global Flags

| Flag | Description |
|------|-------------|
| `--format <FORMAT>` | 出力形式: `json` (default), `table`, `yaml`, `csv` |
| `--dry-run` | API を呼ばずローカル検証のみ |
| `--sanitize <TEMPLATE>` | Model Armor でレスポンスをスクリーニング |

### Method Flags

| Flag | Description |
|------|-------------|
| `--params '{"key": "val"}'` | URL/query パラメータ |
| `--json '{"key": "val"}'` | リクエストボディ |
| `-o, --output <PATH>` | バイナリレスポンスをファイル保存 |
| `--page-all` | 自動ページネーション (NDJSON 出力) |
| `--page-limit <N>` | `--page-all` 時の最大ページ数 (default: 10) |
| `--page-delay <MS>` | ページ間ディレイ ms (default: 100) |

## Security Rules

- シークレット (API key / token) を出力しない
- write / delete 系コマンドは **必ずユーザーに確認してから実行**
- 破壊的操作は `--dry-run` を優先
- PII / コンテンツ安全性スクリーニングが必要なら `--sanitize` を使う

## Shell Tips

- **zsh の `!` 展開**: `--params` / `--json` の値は **シングルクォート** で囲む（インナーのダブルクォートをそのまま渡すため）。
  ```bash
  gws calendar events list --params '{"calendarId": "primary"}'
  ```

---

## Helper Commands

`gws` には Calendar 用の便利なヘルパー (`+`) コマンドがある。日常用途はまずこちらで足りる。

### `+agenda` — 予定の確認 (read-only)

```bash
gws calendar +agenda [flags]
```

| Flag | Description |
|------|-------------|
| `--today` | 今日の予定 |
| `--tomorrow` | 明日の予定 |
| `--week` | 今週の予定 |
| `--days <N>` | 今後 N 日分 |
| `--calendar <NAME_OR_ID>` | 特定カレンダーで絞り込み |
| `--timezone <IANA>` | タイムゾーン上書き (例: `America/Denver`)。デフォルトは Google アカウントの TZ |

**Examples**

```bash
gws calendar +agenda
gws calendar +agenda --today
gws calendar +agenda --week --format table
gws calendar +agenda --days 3 --calendar 'Work'
gws calendar +agenda --today --timezone America/New_York
```

**Tips**

- 読み取り専用 — イベントを変更しない
- デフォルトで全カレンダー横断。`--calendar` で絞り込む

### `+insert` — 予定の作成 (write)

```bash
gws calendar +insert --summary <TEXT> --start <TIME> --end <TIME> [flags]
```

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--calendar` | — | primary | カレンダー ID |
| `--summary` | ✓ | — | 予定タイトル |
| `--start` | ✓ | — | 開始時刻 (RFC3339, 例 `2026-06-17T09:00:00-07:00`) |
| `--end` | ✓ | — | 終了時刻 (RFC3339) |
| `--location` | — | — | 場所 |
| `--description` | — | — | 説明本文 |
| `--attendee` | — | — | 参加者メール (複数指定可) |
| `--meet` | — | — | Google Meet リンクを自動付与 |

**Examples**

```bash
gws calendar +insert --summary 'Standup' --start '2026-06-17T09:00:00-07:00' --end '2026-06-17T09:30:00-07:00'
gws calendar +insert --summary 'Review' --start ... --end ... --attendee alice@example.com
gws calendar +insert --summary 'Meet' --start ... --end ... --meet
```

> [!CAUTION]
> `+insert` は **書き込みコマンド**。実行前に必ずユーザーに確認すること。

---

## API Resources (low-level)

ヘルパーで足りない場合は raw API を直接叩く。

### Worked example: events.patch (write)

`--params` (path/query) と `--json` (body) を組み合わせる write 系の典型形。**他のリソースの patch / update / insert もこの形を踏襲**する。

```bash
gws calendar events patch \
  --params '{"calendarId": "primary", "eventId": "abc123xyz", "sendUpdates": "all"}' \
  --json   '{"start": {"dateTime": "2026-05-10T11:00:00+09:00", "timeZone": "Asia/Tokyo"}}' \
  --dry-run
```

- `--params`: URL/query パラメータ。よく使うのは `calendarId`, `eventId`, `sendUpdates` (`all`/`externalOnly`/`none`), `conferenceDataVersion` (1 で Meet 操作を許可)
- `--json`: リクエストボディ。`patch` は **変更したいフィールドのみ**でよい（`update` は全置換）
- 外側はシングルクォート / 内側 JSON はダブルクォート
- 書き込みはまず `--dry-run` で構文検証 → ユーザー確認 → 本実行
- **時刻範囲を変更するとき**: `start` だけ送ると `end` は据え置きになり 0 長/負長イベントになりうる。**「duration を維持して両端を動かすか / 片端だけ動かすか」と「`sendUpdates` の値」をユーザーに確認**してから body を組む

### acl
- `delete` / `get` / `insert` / `list` / `patch` / `update` / `watch` — アクセス制御ルール

### calendarList
- `delete` / `get` / `insert` / `list` / `patch` / `update` / `watch` — ユーザーのカレンダー一覧管理

### calendars
- `clear` — primary カレンダーの全イベント削除
- `delete` — secondary カレンダー削除
- `get` / `insert` / `patch` / `update` — カレンダーメタデータ操作

> Note: `calendars.insert` で作成すると認証ユーザーがオーナーになる。サービスアカウントで認証するとサービスアカウントがオーナーになり想定外の挙動になりうるため、ドメイン全体の委任で実ユーザーとして認証することを推奨。

### channels
- `stop` — チャンネル経由のリソース監視を停止

### colors
- `get` — カレンダー / イベントの色定義

### events
- `delete` — イベント削除
- `get` — イベント取得 (Google Calendar ID)。iCalendar ID で取りたい場合は `events.list` + `iCalUID` パラメータ
- `import` — 既存イベントのプライベートコピーを追加 (`eventType=default` のみ)
- `insert` — イベント作成
- `instances` — 繰り返しイベントのインスタンス一覧
- `list` — イベント一覧
- `move` — イベントを別カレンダーへ移動 (default 以外は移動不可)
- `patch` / `update` — イベント更新
- `quickAdd` — 自然文からイベント作成
- `watch` — Events リソースの変更監視

### freebusy
- `query` — 複数カレンダーの空き / busy 情報を取得

### settings
- `get` / `list` / `watch` — ユーザー設定

## Discovering Commands

API メソッドを呼ぶ前に、必ずスキーマを確認する。

```bash
# リソース・メソッドを一覧
gws calendar --help

# 特定メソッドの必須パラメータ・型・デフォルトを確認
gws schema calendar.<resource>.<method>
```

`gws schema` の出力をもとに `--params` / `--json` を組み立てる。

## Community & Feedback

- 役に立ったら star: `https://github.com/googleworkspace/cli`
- バグ / 機能要望: `https://github.com/googleworkspace/cli/issues`
- 新規 issue を立てる前に既存 issue を検索すること
