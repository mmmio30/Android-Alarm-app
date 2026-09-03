# 11. データ設計

| 項目 | 内容 |
|---|---|
| フェーズ | B（設計） |
| ステータス | ドラフト |
| 最終更新 | 2026-09-03 |

対象: Room（SQLite）+ DataStore。要件は `03-domain-model.md` を正とする。

---

## 11.1 全体構成

```
┌──────────────────── Room (alarm.db) ────────────────────┐
│                                                          │
│  alarm_group ──1:N── alarm                               │
│       │                  │                               │
│       └──────┬───────────┘                               │
│              │ (scope + owner_id で参照)                  │
│         date_rule          … 除外日 / 予約日             │
│         date_override      … 衝突の手動解決結果           │
│         daily_skip         … その日限りのスキップ         │
│                                                          │
│  holiday_customization     … 内蔵祝日への差分            │
│  ring_session              … 鳴動中の状態（1行）         │
└──────────────────────────────────────────────────────────┘

┌──────────── DataStore (settings.preferences_pb) ────────┐
│  アプリ全体の既定値・表示設定・オンボーディング状態       │
└──────────────────────────────────────────────────────────┘

┌──────────── アプリリソース（DB に入れない / T-019） ─────┐
│  holidays.json  … 内蔵の祝日データ                       │
└──────────────────────────────────────────────────────────┘
```

---

## 11.2 テーブル定義

### alarm_group

```sql
CREATE TABLE alarm_group (
  id                    INTEGER PRIMARY KEY AUTOINCREMENT,
  name                  TEXT    NOT NULL,
  color_token           TEXT    NOT NULL DEFAULT 'blue',
  is_enabled            INTEGER NOT NULL DEFAULT 1,   -- マスタースイッチ (D-011)
  skip_rest_on_dismiss  INTEGER NOT NULL DEFAULT 1,   -- D-008 既定 ON
  skip_holiday          INTEGER NOT NULL DEFAULT 0,   -- 祝日は鳴らさない
  sort_order            INTEGER NOT NULL DEFAULT 0,

  -- グループ共通の鳴動設定（継承元 / D-010）
  s_sound_file          TEXT,                          -- NULL = 端末既定音
  s_sound_label         TEXT,
  s_volume_percent      INTEGER NOT NULL DEFAULT 100,
  s_fade_in_seconds     INTEGER NOT NULL DEFAULT 0,
  s_vibrate             INTEGER NOT NULL DEFAULT 1,
  s_snooze_minutes      INTEGER NOT NULL DEFAULT 5,
  s_snooze_max_count    INTEGER NOT NULL DEFAULT 3,
  s_auto_stop_minutes   INTEGER NOT NULL DEFAULT 5,
  s_auto_stop_action    TEXT    NOT NULL DEFAULT 'DISMISS'
);
```

### alarm

```sql
CREATE TABLE alarm (
  id                       INTEGER PRIMARY KEY AUTOINCREMENT,
  group_id                 INTEGER REFERENCES alarm_group(id) ON DELETE SET NULL,
  label                    TEXT    NOT NULL DEFAULT '',
  hour                     INTEGER NOT NULL,          -- 0-23
  minute                   INTEGER NOT NULL,          -- 0-59
  repeat_days_mask         INTEGER NOT NULL DEFAULT 0,-- ビットマスク。0 = 繰り返しなし
  is_enabled               INTEGER NOT NULL DEFAULT 1,-- 個別スイッチ (D-011)
  valid_from_epoch_day     INTEGER,                   -- NULL = 制限なし
  valid_until_epoch_day    INTEGER,
  sort_order               INTEGER NOT NULL DEFAULT 0,

  -- 設定の継承 (D-010)
  inherits_group_settings  INTEGER NOT NULL DEFAULT 1,
  s_sound_file             TEXT,
  s_sound_label            TEXT,
  s_volume_percent         INTEGER NOT NULL DEFAULT 100,
  s_fade_in_seconds        INTEGER NOT NULL DEFAULT 0,
  s_vibrate                INTEGER NOT NULL DEFAULT 1,
  s_snooze_minutes         INTEGER NOT NULL DEFAULT 5,
  s_snooze_max_count       INTEGER NOT NULL DEFAULT 3,
  s_auto_stop_minutes      INTEGER NOT NULL DEFAULT 5,
  s_auto_stop_action       TEXT    NOT NULL DEFAULT 'DISMISS',

  -- 派生値のキャッシュ（11.5 参照。信頼できる値ではない）
  cached_next_ring_at      INTEGER                    -- epoch millis / NULL = 予定なし
);

CREATE INDEX idx_alarm_group ON alarm(group_id);
CREATE INDEX idx_alarm_next  ON alarm(cached_next_ring_at);
```

**`repeat_days_mask`**: `java.time.DayOfWeek` の値（月=1 … 日=7）に対し `bit(dow - 1)`。
月〜金 = `0b0011111` = 31。0 は「繰り返しなし（ワンショット）」を意味する。

**設定の継承を「NULL 許容の埋め込み」にしない理由**:
Room で `@Embedded` を nullable にすると、全カラムが NULL のときだけ null と解釈されるなど
挙動が分かりにくい。代わりに `inherits_group_settings` フラグ + 常在の設定カラムとする。
継承 → 個別に切り替えた瞬間に**グループの現在値をコピーする**ので、
UI 上も「今の見た目のまま個別編集に移る」という自然な挙動になる。

### date_rule（除外日 / 予約日 / T-004）

```sql
CREATE TABLE date_rule (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  scope            TEXT    NOT NULL,          -- 'APP' | 'GROUP' | 'ALARM'
  owner_id         INTEGER,                   -- APP のときは NULL
  kind             TEXT    NOT NULL,          -- 'EXCLUDE' | 'INCLUDE'
  start_epoch_day  INTEGER NOT NULL,
  end_epoch_day    INTEGER NOT NULL,          -- 単日なら start と同値
  yearly_repeat    INTEGER NOT NULL DEFAULT 0,
  note             TEXT
);

CREATE INDEX idx_date_rule_owner ON date_rule(scope, owner_id);
CREATE INDEX idx_date_rule_range ON date_rule(start_epoch_day, end_epoch_day);
```

### date_override（衝突の手動解決 / D-015・T-018）

```sql
CREATE TABLE date_override (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  scope        TEXT    NOT NULL,              -- 'APP' | 'GROUP' | 'ALARM'
  owner_id     INTEGER,
  epoch_day    INTEGER NOT NULL,              -- 単日のみ（期間は日ごとに展開）
  decision     TEXT    NOT NULL,              -- 'RING' | 'SILENT'
  decided_at   INTEGER NOT NULL               -- epoch millis
);

CREATE UNIQUE INDEX idx_override_key ON date_override(scope, owner_id, epoch_day);
```

### daily_skip（その日限りのスキップ / D-008）

```sql
CREATE TABLE daily_skip (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  epoch_day   INTEGER NOT NULL,
  group_id    INTEGER,                        -- グループ単位のスキップ
  alarm_id    INTEGER,                        -- アラーム単位のスキップ
  created_at  INTEGER NOT NULL
);

CREATE INDEX idx_daily_skip_day ON daily_skip(epoch_day);
```

`group_id` と `alarm_id` はどちらか一方のみを設定する。

### holiday_customization（D-014）

```sql
CREATE TABLE holiday_customization (
  epoch_day  INTEGER PRIMARY KEY,
  kind       TEXT NOT NULL,                   -- 'ADD' | 'REMOVE'
  name       TEXT
);
```

### ring_session（鳴動中の状態）

```sql
CREATE TABLE ring_session (
  id            INTEGER PRIMARY KEY CHECK (id = 0),   -- 常に1行のみ
  alarm_id      INTEGER NOT NULL,
  fired_at      INTEGER NOT NULL,
  snooze_count  INTEGER NOT NULL DEFAULT 0,
  state         TEXT    NOT NULL                      -- 'RINGING'|'SNOOZING'|'DISMISSED'
);
```

**永続化する理由**: 鳴動中にアプリが落ちても、スヌーズ回数と鳴動状態を復元するため。

---

## 11.3 参照整合性の扱い（設計上の妥協点）

`date_rule` / `date_override` / `daily_skip` は `scope` によって参照先テーブルが変わるため、
**SQL の外部キー制約を張れない**（多態参照）。

| 案 | 内容 | 採否 |
|---|---|---|
| A | スコープごとに 3 テーブルに分ける | ✕ カレンダー UI と判定ロジックが 3 重化し、T-004 の狙い（共通化）が崩れる |
| B | 1 テーブル + 多態参照。整合性はアプリ側で担保 | ✅ **採用** |

**B を安全に運用するための対策**

1. グループ / アラームの削除は**必ずトランザクション内**で関連レコードも削除する
2. Room の `id` は `AUTOINCREMENT` を使い、**ID の再利用を防ぐ**
   （再利用されると、削除済みオーナーの孤児レコードが別レコードに誤って効いてしまう）
3. 日次セーフティネット（`03-domain-model.md` 3.4）で孤児レコードを掃除する
4. 過去の `daily_skip` と、終了済みで繰り返さない `date_rule` は同時に整理する（FR-4.1.7）

---

## 11.4 DateRule の評価はメモリ上で行う

`yearly_repeat = 1` のルールは「月日が一致するか」で判定するため、
epoch day を使った SQL の範囲検索では表現できない。

**方針**: `date_rule` / `date_override` / `holiday_customization` は
**対象オーナー分をすべてメモリに読み込んでから判定する**。SQL は CRUD にのみ使う。

- 想定件数は除外日 500 件程度（NFR-3.1）であり、全件をメモリに置いても問題にならない
- 判定ロジックを Android 非依存の純粋関数に保てる（NFR-6.1）ので、テストが書ける

---

## 11.5 `cached_next_ring_at` の扱い（派生値）

ホーム画面で 200 件のアラームに「次に鳴る日時」を出すため（FR-1.6）にキャッシュを持つ。
**これは真実の値ではなく、常に再計算で上書きされる。**

| 論点 | 方針 |
|---|---|
| 更新する主体 | `AlarmScheduler` の再計算処理のみ。UI からは書き込まない |
| 更新タイミング | 再計算トリガ（設定変更 / 発火後 / 起動 / 再起動 / TZ 変更 / 日次） |
| 信頼できないケース | 日付をまたいだ直後、端末の時刻を手動変更した直後 |
| 対策 | アプリ起動時と日付変更（`ACTION_DATE_CHANGED`）で必ず再計算する |
| 実装上の注意 | この値を条件にアラームを登録してはいけない。**登録は必ず再計算結果を使う** |

---

## 11.6 DataStore（アプリ設定）

| キー | 型 | 既定値 | 用途 |
|---|---|---|---|
| `default_sound_file` | String? | null | 新規アラームの既定音 |
| `default_volume_percent` | Int | 100 | |
| `default_vibrate` | Boolean | true | |
| `default_snooze_minutes` | Int | 5 | |
| `default_snooze_max_count` | Int | 3 | |
| `default_auto_stop_minutes` | Int | 5 | |
| `conflict_default` | Enum | `SILENT` | 未解決の衝突の既定（T-017 / FR-4.5.8） |
| `schedule_preview_days` | Int | 30 | S-06 の表示範囲 |
| `theme_mode` | Enum | `SYSTEM` | |
| `onboarding_completed` | Boolean | false | |
| `last_bulk_create_days_mask` | Int | 0 | 一括作成の前回値 |

---

## 11.7 内蔵祝日データ（リソース / T-019）

DB に入れず、アプリのアセットとして持つ。アプリ更新でファイルごと差し替えられる。

```json
{
  "version": "2026.1",
  "coveredFrom": "2026-01-01",
  "coveredUntil": "2029-12-31",
  "holidays": [
    { "date": "2026-09-15", "name": "敬老の日" },
    { "date": "2026-09-23", "name": "秋分の日" }
  ]
}
```

| 論点 | 方針 |
|---|---|
| 振替休日 | 事前に展開済みの状態で収録する（実行時に計算しない） |
| 収録期間 | 現在年 + 3 年分を既定とする |
| `coveredUntil` の用途 | S-09 に「〜2029年まで収録」と表示し、期限が近づいたら更新を促す（FR-4.3.7 / 4.3.8） |
| 読み込み | 起動時に一度読み込み、メモリに保持する（数百件程度） |
| 秋分・春分の日 | 年によって日付が変わるため、必ず確定値を収録する。計算式による近似は使わない |

---

## 11.8 マイグレーション方針

| 項目 | 方針 |
|---|---|
| 初期バージョン | `version = 1` |
| `fallbackToDestructiveMigration` | **使用禁止**（NFR-4.4）。アラームが消えることは致命的 |
| スキーマ出力 | `room.schemaLocation` を有効にし、`schemas/*.json` を Git 管理する |
| マイグレーションのテスト | `MigrationTestHelper` で各バージョン間の移行をテストする |
| 祝日データの更新 | DB を触らないためマイグレーション不要（T-019 の狙い） |
| バックアップとの関係 | フェーズ F の JSON エクスポートは**スキーマ版数を含める**。古い版数の取り込み時は変換する |

---

## 11.9 想定データ量と性能

| データ | 想定件数 | 備考 |
|---|---|---|
| alarm_group | 〜20 | |
| alarm | 〜200 | NFR-3.1 の想定上限 |
| date_rule | 〜500 | 期間指定は 1 レコードなので、実日数はこれより多い |
| date_override | 〜100 | 衝突が起きた日だけ |
| daily_skip | 〜30 | 過去分は日次で整理 |
| holiday_customization | 〜50 | |

`willRing()` の全件再計算は 200 アラーム × 400 日探索が最悪ケース。
実際は「見つかった時点で打ち切る」ため、通常は 1 アラームあたり数日分の評価で済む。
NFR-3.2（100ms 以内）は達成可能と見込む。**実装時にベンチマークで確認する。**

---

## 11.10 未確定の論点

| # | 論点 | 備考 |
|---|---|---|
| DB-1 | 音源ファイルをアプリ内にコピーする際の保存先と命名（T-016） | 同じ音源を複数アラームで使うときの重複をどうするか |
| DB-2 | `date_override` を期間で持てるようにするか | 現状は日ごとに展開。長期間の衝突でレコードが増える |
| DB-3 | 削除の Undo（NFR-2.5）を論理削除で実現するか、メモリ保持で足りるか | |
