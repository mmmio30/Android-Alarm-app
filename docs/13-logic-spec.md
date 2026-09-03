# 13. ロジック仕様とテストケース

| 項目 | 内容 |
|---|---|
| フェーズ | B（設計） |
| ステータス | ドラフト |
| 最終更新 | 2026-09-03 |

> **このドキュメントがアプリの正しさそのもの。**
> ここに書かれたテストケースを全て通すことが、実装完了の定義になる。
> UI より先にこの層を実装する（`08-scope-and-roadmap.md` フェーズ C）。

---

## 13.1 RingDecider.decide()

```kotlin
fun decide(
  alarm: Alarm,
  group: AlarmGroup?,
  date: LocalDate,
  ctx: DecisionContext,      // 除外日/予約日/上書き/スキップ/祝日を保持
): RingDecision {

  // 1. スイッチ（何よりも強い）
  if (!alarm.isEnabled)              return Skip(ALARM_DISABLED)
  if (group != null && !group.isEnabled) return Skip(GROUP_DISABLED)

  // 2. 有効期間（両端を含む）
  alarm.validFrom?.let  { if (date < it) return Skip(OUT_OF_VALID_PERIOD) }
  alarm.validUntil?.let { if (date > it) return Skip(OUT_OF_VALID_PERIOD) }

  // 3. その日限りのスキップ
  if (ctx.isDailySkipped(alarm, date)) return Skip(DAILY_SKIPPED)

  // 4. 手動で解決済みの衝突（ALARM → GROUP → APP の順に探す）
  ctx.findOverride(alarm, date)?.let {
    return if (it.decision == RING) Ring(alarm.time) else Skip(OVERRIDE_SILENT)
  }

  // 5・6. 除外日と予約日（順序は設定で入れ替わる / FR-4.5.8）
  val excluded = ctx.findExclude(alarm, date)      // スコープを返す
  val included = ctx.matchesInclude(alarm, date)

  if (ctx.conflictDefault == SILENT) {
    if (excluded != null) return Skip(excluded.toReason())
    if (included)         return Ring(alarm.time)
  } else {
    if (included)         return Ring(alarm.time)
    if (excluded != null) return Skip(excluded.toReason())
  }

  // 7. 祝日（予約日より弱い。明示指定 > 自動ルール）
  if (effectiveSkipHoliday(alarm, group) && ctx.holidays.isHoliday(date))
    return Skip(HOLIDAY)

  // 8. 繰り返し条件
  return if (alarm.repeatDaysMask == 0) {
    if (ctx.isNextOneShotDate(alarm, date)) Ring(alarm.time) else Skip(NOT_SCHEDULED_DAY)
  } else {
    if (alarm.repeatDaysMask.hasDay(date.dayOfWeek)) Ring(alarm.time)
    else Skip(NOT_SCHEDULED_DAY)
  }
}
```

### 判定順序の根拠

| 順 | 条件 | なぜこの位置か |
|---|---|---|
| 1 | スイッチ | ユーザーが「止めた」ものは何があっても鳴らさない |
| 2 | 有効期間 | アラームの存在期間そのもの |
| 3 | 当日スキップ | その日の実行時判断（起床済み） |
| 4 | 手動解決 | **ユーザーが明示的に答えを出した日**。自動ルールより強い |
| 5・6 | 除外日 / 予約日 | どちらも日付を名指しした明示指定。同格なので衝突しうる（D-015） |
| 7 | 祝日 | 自動ルール。明示指定（5・6）より弱い |
| 8 | 曜日繰り返し | 最も基本的なパターン。他に何も該当しなければこれ |

> **設計中に見つけた不具合**: 初期案では祝日を予約日より先に評価していた。
> その順序だと「祝日は鳴らさない」グループで祝日に予約日を入れても鳴らず、
> ユーザーの明示的な指定が黙って無視されてしまう。
> **明示指定 > 自動ルール**という原則を立てて順序を入れ替えた。

---

## 13.2 NextRingCalculator.next()

```kotlin
fun next(alarm: Alarm, from: Instant): Instant? {
  val zone = ZoneId.systemDefault()
  var date = from.atZone(zone).toLocalDate()

  repeat(MAX_SEARCH_DAYS) {                    // 400 日（3.4）
    val decision = ringDecider.decide(alarm, group, date, ctx)
    if (decision is Ring) {
      val at = ZonedDateTime.of(date, decision.time, zone).toInstant()
      if (at > from) return at                 // 同じ日でも過ぎた時刻は対象外
    }
    date = date.plusDays(1)
  }
  return null                                  // 予定なし
}
```

| 論点 | 仕様 |
|---|---|
| 「過ぎた」の基準 | `at > from`（秒未満は切り捨てて比較する） |
| 探索上限 | 400 日。除外日だらけで見つからない場合は「予定なし」 |
| 存在しない時刻 | `ZonedDateTime.of` が自動補正する（日本では発生しないが実装は依存しない） |
| ワンショット | 次の 1 回のみ返す。過ぎたら `null` → アラームを自動 OFF にする |
| グループ OFF | ステップ 1 で弾かれるため常に `null` |

---

## 13.3 ConflictDetector

```kotlin
fun detect(target: DateRule, allRules: List<DateRule>, overrides: List<DateOverride>)
    : List<LocalDate>
```

| 入力の種類 | 検出対象 |
|---|---|
| 追加する規則が INCLUDE（予約日） | 同一スコープ + 上位スコープの EXCLUDE と重なる日 |
| 追加する規則が EXCLUDE（除外日） | 同一スコープ + 下位スコープの INCLUDE と重なる日 |

- 既に `DateOverride` で解決済みの日は**衝突として報告しない**
- 上位スコープの除外日を追加した場合、影響を受けるアラーム数も併せて返す（S-05 の表示用）
- 毎年繰り返す規則は、**探索範囲（今後 400 日）内**で展開して突き合わせる

---

## 13.4 BulkAlarmGenerator

```kotlin
fun generateByCount(start: LocalTime, intervalMinutes: Int, count: Int): List<LocalTime>
fun generateByEndTime(start: LocalTime, end: LocalTime, intervalMinutes: Int): List<LocalTime>
```

| 論点 | 仕様 |
|---|---|
| 日跨ぎ | `23:50 + 5分 × 5` → `23:50, 23:55, 00:00, 00:05, 00:10`。翌日分は UI で「翌日」と表示 |
| 終了時刻の扱い | `generateByEndTime` は**終了時刻を含む**（6:00〜6:30 / 5 分 → 7 件） |
| 上限 | 50 件（T-013）。超える指定はエラーを返し、UI で作成不可にする |
| 間隔 | 1 以上。0 以下はエラー |
| 個数 | 1 以上 50 以下 |
| 重複 | 既存アラームと同時刻になるものを返し、UI で警告する（作成自体は可能） |

---

## 13.5 HolidayCalendar

```kotlin
fun isHoliday(date: LocalDate): Boolean {
  customizations[date]?.let {
    return it.kind == ADD          // REMOVE なら false、ADD なら true
  }
  return builtIn.contains(date)
}
```

| 論点 | 仕様 |
|---|---|
| 優先順位 | ユーザー編集 > 内蔵データ |
| 収録範囲外の日付 | `false` を返す。S-09 で「〜2029年まで収録」と明示し、誤解を防ぐ |
| 振替休日 | 内蔵データに展開済みで収録する |

---

## 13.6 テストケース一覧

### TC-D: RingDecider（最重点）

| ID | 前提 | 期待結果 |
|---|---|---|
| D-01 | 個別 OFF | `Skip(ALARM_DISABLED)` |
| D-02 | グループ OFF / 個別 ON | `Skip(GROUP_DISABLED)` |
| D-03 | グループ ON / 個別 OFF | `Skip(ALARM_DISABLED)` |
| D-04 | グループ OFF → ON に戻す | 個別 OFF だったアラームは OFF のまま（D-011） |
| D-05 | グループ未所属 + 個別 ON | グループ判定を行わず通過 |
| D-06 | `validFrom` の前日 | `Skip(OUT_OF_VALID_PERIOD)` |
| D-07 | `validFrom` 当日 | 鳴る（両端を含む） |
| D-08 | `validUntil` 当日 / 翌日 | 当日は鳴る / 翌日は `Skip` |
| D-09 | `DailySkip` に該当 | `Skip(DAILY_SKIPPED)` |
| D-10 | 除外日あり + `DateOverride(RING)` | **鳴る**（手動解決が勝つ） |
| D-11 | 予約日あり + `DateOverride(SILENT)` | `Skip(OVERRIDE_SILENT)` |
| D-12 | ALARM と APP に別々の `DateOverride` | ALARM 側が勝つ |
| D-13 | APP スコープの除外日 | `Skip(EXCLUDED_APP)` |
| D-14 | GROUP スコープの除外日 | `Skip(EXCLUDED_GROUP)` |
| D-15 | ALARM スコープの除外日 | `Skip(EXCLUDED_ALARM)` |
| D-16 | 期間除外の開始日 / 終了日 | 両端とも除外される |
| D-17 | 期間除外の開始前日 / 終了翌日 | 除外されない |
| D-18 | `yearlyRepeat` の除外日（翌年の同月日） | 除外される |
| D-19 | `yearlyRepeat` で 2/29 指定、平年の判定 | **要仕様決定**（→ 13.7 L-1） |
| D-20 | 予約日あり（曜日は非該当） | 鳴る（曜日条件を上書き） |
| D-21 | 予約日 + 祝日（祝日 OFF 設定） | **鳴る**（明示指定 > 自動ルール） |
| D-22 | 除外日 + 予約日が同日 / 未解決 / 既定 SILENT | `Skip` |
| D-23 | 同上 / 既定 RING に設定 | 鳴る |
| D-24 | 祝日 + 祝日 OFF 設定 | `Skip(HOLIDAY)` |
| D-25 | 祝日 + 祝日 OFF 未設定 | 鳴る |
| D-26 | 内蔵祝日 + `HolidayCustomization(REMOVE)` | 鳴る |
| D-27 | 平日 + `HolidayCustomization(ADD)` + 祝日 OFF | `Skip(HOLIDAY)` |
| D-28 | 曜日マスクに該当 / 非該当 | 鳴る / `Skip(NOT_SCHEDULED_DAY)` |
| D-29 | `repeatDaysMask == 0`（ワンショット）の次の該当日 | 鳴る |
| D-30 | ワンショットで既に鳴り終わった日 | `Skip` |
| D-31 | グループの祝日設定を個別アラームが継承 | グループ設定に従う |

### TC-N: NextRingCalculator

| ID | 前提 | 期待結果 |
|---|---|---|
| N-01 | 今日 7:00 設定 / 現在 6:00 | 今日 7:00 |
| N-02 | 今日 7:00 設定 / 現在 7:00 ちょうど | **翌該当日**（`at > from`） |
| N-03 | 今日 7:00 設定 / 現在 8:00 | 次の該当曜日 |
| N-04 | 月〜金設定 / 現在 土曜 | 翌週月曜 |
| N-05 | 除外日が 10 日連続 | 11 日目 |
| N-06 | 12/30 に設定、年跨ぎ | 翌年 1/x を正しく返す |
| N-07 | うるう年 2/29 の予約日 | 2/29 を正しく返す |
| N-08 | すべての日が除外されている | `null`（400 日探索後） |
| N-09 | グループ OFF | `null` |
| N-10 | ワンショットで過去日のみ | `null` → アラームを自動 OFF |
| N-11 | タイムゾーンを +9 → +1 に変更 | 壁時計時刻（7:00）が維持される |

### TC-C: ConflictDetector

| ID | 前提 | 期待結果 |
|---|---|---|
| C-01 | 予約日を追加、同スコープに除外日あり | その日を衝突として返す |
| C-02 | 予約日を追加、APP スコープに除外日あり | 衝突として返す |
| C-03 | 除外日（期間 5 日）を追加、うち 2 日に予約日 | 2 日分を返す |
| C-04 | 既に `DateOverride` で解決済みの日 | 衝突として返さない |
| C-05 | 衝突なし | 空リスト |
| C-06 | 毎年繰り返す除外日 × 来年の予約日 | 探索範囲内なら衝突として返す |

### TC-B: BulkAlarmGenerator

| ID | 前提 | 期待結果 |
|---|---|---|
| B-01 | 6:00 / 5 分 / 6 個 | 6:00, 6:05, 6:10, 6:15, 6:20, 6:25 |
| B-02 | 23:50 / 5 分 / 5 個 | 23:50, 23:55, 0:00, 0:05, 0:10 |
| B-03 | 6:00〜6:30 / 5 分（終了時刻指定） | 7 件（終了時刻を含む） |
| B-04 | 間隔 0 / 負数 | エラー |
| B-05 | 個数 0 / 51 | エラー（上限 50） |
| B-06 | 個数 1 | 1 件（起点のみ） |
| B-07 | 既存アラームと同時刻を含む | 重複リストを返す |

### TC-H: HolidayCalendar

| ID | 前提 | 期待結果 |
|---|---|---|
| H-01 | 内蔵データの祝日 | `true` |
| H-02 | 内蔵データの振替休日 | `true` |
| H-03 | `REMOVE` された内蔵祝日 | `false` |
| H-04 | `ADD` されたカスタム休日 | `true` |
| H-05 | 収録範囲外の日付 | `false` |

### TC-S: AlarmScheduler（Robolectric）

| ID | 前提 | 期待結果 |
|---|---|---|
| S-01 | 複数アラームあり | 最も早い 1 件だけが登録される |
| S-02 | 全アラームが「予定なし」 | 登録がキャンセルされる |
| S-03 | 同時刻に 3 件 | 1 件の登録で 3 件すべてが処理対象になる |
| S-04 | `reschedule()` を並行に 2 回呼ぶ | 直列化され、後の結果が残る |
| S-05 | 再計算後の `cached_next_ring_at` | 全アラームで更新されている |

### TC-M: Room

| ID | 内容 |
|---|---|
| M-01 | グループ削除時、配下アラームの `group_id` が NULL になる |
| M-02 | グループ削除時、そのグループの `date_rule` / `date_override` も削除される |
| M-03 | アラーム削除時、関連する `date_rule` / `daily_skip` も削除される |
| M-04 | 各バージョン間のマイグレーション（v2 以降を作るときに追加） |

---

## 13.7 実装前に決める必要が残っている論点

| # | 論点 | 案 |
|---|---|---|
| L-1 | `yearlyRepeat` で 2/29 を指定した場合、平年はどう扱うか | A: 2/28 に繰り上げる / B: 平年は該当なし（推奨・単純） |
| L-2 | 同時刻に複数アラームがある場合の音の選び方 | 所属グループの `sort_order` が小さい方 / 先に作られた方 |
| L-3 | ワンショットアラームを鳴り終わった後、削除するか OFF にして残すか | OFF にして残す方が復元しやすい（推奨） |
| L-4 | `DailySkip` の「今日」の基準 | 鳴動した瞬間のローカル日付（深夜 2 時なら当日扱い / D-018 と整合） |
