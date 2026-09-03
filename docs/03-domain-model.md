# 03. ドメインモデルと鳴動判定ロジック

| 項目 | 内容 |
|---|---|
| ステータス | ドラフト |
| 最終更新 | 2026-09-03 |

## 3.1 エンティティ一覧

```
AppSettings (1)
  └─ DateRule[]            … アプリ全体の除外日 / 休止日

AlarmGroup (N)
  ├─ GroupSettings         … 音・バイブ・スヌーズ等の共通設定
  ├─ DateRule[]            … グループの除外日 / 予約日
  └─ Alarm (N)
        ├─ AlarmSettings?  … null なら「グループから継承」
        └─ DateRule[]      … アラーム個別の除外日 / 予約日

RingSession                … 鳴動中の一時状態（永続化する）
```

## 3.2 主要フィールド（案）

### AlarmGroup

| フィールド | 型 | 説明 |
|---|---|---|
| id | Long | 主キー |
| name | String | グループ名 |
| colorToken | String | 表示色 |
| isEnabled | Boolean | **マスタースイッチ**。false なら配下すべて鳴らない |
| skipRestOnDismiss | Boolean | 1 つ解除したらその日の残りをスキップ（FR-2.4.1） |
| holidayBehavior | Enum | NONE / SKIP_JP_HOLIDAY |
| settings | GroupSettings | 継承元となる共通設定 |
| sortOrder | Int | 並び順 |

### Alarm

| フィールド | 型 | 説明 |
|---|---|---|
| id | Long | 主キー |
| groupId | Long? | 所属グループ。null = 未所属 |
| label | String | ラベル |
| hour / minute | Int | 時刻 |
| repeatDays | Set\<DayOfWeek\> | 空 = 繰り返しなし（ワンショット） |
| isEnabled | Boolean | **個別スイッチ** |
| validFrom / validUntil | LocalDate? | 有効期間（FR-4.2.4） |
| overrideSettings | AlarmSettings? | null = グループ継承 |
| sortOrder | Int | グループ内の並び順（エスカレーション用にも使う） |

### AlarmSettings / GroupSettings（同一構造）

| フィールド | 型 | 既定値 |
|---|---|---|
| soundUri | String? | 端末既定のアラーム音 |
| volumePercent | Int | 100 |
| fadeInSeconds | Int | 0（= 無効） |
| vibrate | Boolean | true |
| snoozeMinutes | Int | 5 |
| snoozeMaxCount | Int | 3 |
| autoStopMinutes | Int | 5 |
| autoStopAction | Enum | DISMISS / SNOOZE |
| dismissChallenge | Enum | NONE / MATH / SHAKE / TYPING |

### DateRule（除外日 / 予約日の共通表現）

| フィールド | 型 | 説明 |
|---|---|---|
| id | Long | 主キー |
| scope | Enum | APP / GROUP / ALARM |
| ownerId | Long? | scope に対応する ID |
| kind | Enum | **EXCLUDE**（鳴らさない）/ **INCLUDE**（この日は鳴らす） |
| startDate | LocalDate | 開始日 |
| endDate | LocalDate | 終了日（単日なら startDate と同じ） |
| yearlyRepeat | Boolean | 毎年繰り返すか |
| note | String? | 「帰省」「有給」などのメモ |

> **設計意図**: 除外日と予約日を 1 テーブルで表現することで、
> カレンダー UI・一覧 UI・判定ロジックをすべて共通化できる。

### RingSession（鳴動中の状態）

| フィールド | 型 | 説明 |
|---|---|---|
| alarmId | Long | 鳴っているアラーム |
| firedAt | Instant | 鳴動開始時刻 |
| snoozeCount | Int | 現在のスヌーズ回数 |
| state | Enum | RINGING / SNOOZING / DISMISSED |

### DailySkip（その日の一時スキップ）

| フィールド | 型 | 説明 |
|---|---|---|
| date | LocalDate | 対象日 |
| groupId | Long? | グループ単位のスキップ（FR-2.4.1 発動時） |
| alarmId | Long? | アラーム単位のスキップ（「今日はスキップ」操作） |

---

## 3.3 実効 ON 判定アルゴリズム（★仕様の中核）

「アラーム A は 日付 D に鳴るか？」を判定する。**上から順に評価し、最初に該当したもので確定**する。

```
fun willRing(alarm: Alarm, date: LocalDate): Boolean {

  // --- ステップ 1: スイッチ ---
  if (!alarm.isEnabled) return false
  if (alarm.group != null && !alarm.group.isEnabled) return false      // FR-2.2.2 (AND判定)

  // --- ステップ 2: 有効期間 ---
  if (alarm.validFrom  != null && date < alarm.validFrom)  return false
  if (alarm.validUntil != null && date > alarm.validUntil) return false

  // --- ステップ 3: その日限りのスキップ ---
  if (dailySkipExists(alarm, date)) return false                        // FR-2.4.1 / 「今日はスキップ」

  // --- ステップ 4: 除外指定（3階層の OR。最も強い） ---
  if (matchesExcludeRule(scope = APP,   date)) return false
  if (matchesExcludeRule(scope = GROUP, date)) return false
  if (matchesExcludeRule(scope = ALARM, date)) return false

  // --- ステップ 5: 祝日 ---
  if (effectiveHolidayBehavior(alarm) == SKIP_JP_HOLIDAY
      && holidayCalendar.isHoliday(date)) return false

  // --- ステップ 6: 予約日（INCLUDE）は曜日条件を上書きして鳴らす ---
  if (matchesIncludeRule(alarm, date)) return true

  // --- ステップ 7: 通常の繰り返し条件 ---
  return if (alarm.repeatDays.isEmpty()) {
           isNextOneShotDate(alarm, date)      // 繰り返しなし: 直近1回のみ
         } else {
           date.dayOfWeek in alarm.repeatDays
         }
}
```

### 判定の優先順位（強い順）

| 順位 | 条件 | 結果 |
|---|---|---|
| 1 | スイッチ OFF（個別 or グループ） | 鳴らない |
| 2 | 有効期間外 | 鳴らない |
| 3 | 当日スキップ | 鳴らない |
| 4 | 除外日（APP / GROUP / ALARM のいずれか） | 鳴らない |
| 5 | 祝日（祝日 OFF 設定時） | 鳴らない |
| 6 | 予約日（INCLUDE） | **鳴る** |
| 7 | 曜日繰り返し条件 | 条件次第 |

**[要判断]** 「予約日（6）を除外日（4）より強くするか」は逆の設計もありうる。
現案は「除外の方が強い」。理由: 「有給を入れたので全部止めたい」が最も直感に近いため。

---

## 3.4 スケジューリング方式（★実装の中核）

### 課題
1 グループ 10 個 × 5 グループ = 50 アラーム、さらに予約日展開で数百件になりうる。
これらを全部 `AlarmManager` に登録するのは非現実的（システム制限・電池・再計算コスト）。

### 方針: 「次の 1 件だけ登録し、発火のたびに再計算する」

```
1. 全アラームについて「次の実効鳴動時刻」を計算（3.3 のロジックを未来方向に走査）
2. 最も早いものを 1 件だけ AlarmManager.setAlarmClock() で登録
   （同一時刻に複数ある場合は 1 つの発火でまとめて処理）
3. 発火時: 鳴動 → 完了後に 1. へ戻って再計算
```

**メリット**
- 登録数が常に 1 件 → システム制限・電池消費を回避
- `setAlarmClock()` を使うため Doze を貫通し、ステータスバーに次のアラームが出る
- 除外日を後から追加しても、再計算するだけで正しく反映される

### 再計算のトリガ

| トリガ | 実装 |
|---|---|
| アラーム / グループ / 除外日の変更 | Repository の変更を監視して即時再計算 |
| アラーム発火後 | 鳴動処理の完了時 |
| 端末再起動 | `BOOT_COMPLETED` / `LOCKED_BOOT_COMPLETED` |
| アプリ更新 | `MY_PACKAGE_REPLACED` |
| タイムゾーン / 時刻変更 | `TIMEZONE_CHANGED` / `TIME_SET` |
| **セーフティネット** | WorkManager で 1 日 1 回（深夜帯）に再計算・整合性チェック |

### 未来方向の走査上限
「次の鳴動時刻」を探すとき、除外日だらけで永遠に見つからない可能性があるため、
**探索上限を 400 日**とし、超えたら「予定なし」として扱う。

---

## 3.5 グループ「1 つ解除で残りスキップ」の状態遷移（FR-2.4.1）

```
[アラーム鳴動]
      │
      ├── ユーザーが「停止」
      │        │
      │        ├─ group.skipRestOnDismiss == true
      │        │      → DailySkip(date=今日, groupId=G) を作成
      │        │      → 通知「本日の残り N 件をスキップしました / 取り消す」
      │        │      → 再計算（残りは willRing() のステップ 3 で false になる）
      │        │
      │        └─ false → 何もしない（残りは通常どおり鳴る）
      │
      ├── ユーザーが「スヌーズ」
      │        → RingSession を SNOOZING にして snoozeMinutes 後に再登録
      │        → DailySkip は作らない（まだ起きていないため）
      │
      └── 自動停止（無操作で autoStopMinutes 経過）
               → DailySkip は作らない（起きていない可能性が高いため）★重要
```

**★重要な設計判断**: 「自動停止」ではスキップを発動させない。
無操作で鳴り止んだ = 寝ている、なので残りは鳴らし続けるべき。
「明示的に停止操作をした」ときのみ起床とみなす。

---

## 3.6 「日付」の扱いに関する注意

| 論点 | 方針 |
|---|---|
| 日付境界 | ローカル日付の 0:00 で区切る |
| 深夜アラーム（例 2:00） | 「その日」= 鳴る瞬間のローカル日付。ユーザーの体感（前日の夜）とズレる可能性を UI で補足 |
| タイムゾーン変更 | ローカル時刻を維持（7:00 設定なら移動先でも 7:00）。再計算をトリガ |
| サマータイム | 日本では非対応だが、`java.time` の `ZonedDateTime` を用いて存在しない時刻・重複時刻を正しく処理 |
| 保存形式 | 時刻は「時・分」の壁時計値で保存し、UTC エポックでは保存しない |
