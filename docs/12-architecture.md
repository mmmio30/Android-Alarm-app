# 12. アーキテクチャ設計

| 項目 | 内容 |
|---|---|
| フェーズ | B（設計） |
| ステータス | ドラフト |
| 最終更新 | 2026-09-03 |

---

## 12.1 レイヤ構成

```
┌──────────────────────────────────────────────────────────┐
│  UI 層 (Compose)                                          │
│    Screen ── ViewModel ── UiState(StateFlow)              │
│    Android に依存する。ロジックは持たない                  │
├──────────────────────────────────────────────────────────┤
│  Domain 層  ★ Android 非依存・純粋 Kotlin                 │
│    model/     : Alarm, AlarmGroup, DateRule, ...          │
│    schedule/  : RingDecider, NextRingCalculator,          │
│                 ConflictDetector, HolidayCalendar         │
│    usecase/   : ToggleGroup, BulkCreateAlarms, ...        │
│    ← ここに全ての判断ロジックを閉じ込める（NFR-6.1）       │
├──────────────────────────────────────────────────────────┤
│  Data 層                                                  │
│    Room(DAO) / DataStore / アセット(祝日JSON)             │
│    Repository が Domain のモデルへ変換して渡す             │
├──────────────────────────────────────────────────────────┤
│  Platform 層  ★ Android のアラーム機構との接続            │
│    AlarmScheduler / AlarmReceiver / BootReceiver /        │
│    RingService / AlarmPlayer / NotificationHelper         │
└──────────────────────────────────────────────────────────┘
```

**最重要の設計原則**:
「鳴るか鳴らないか」の判断は **Domain 層の純粋関数だけ**が行う。
`Context` も `Calendar` も `System.currentTimeMillis()` も使わない。
時刻は `java.time.Clock` を注入する（NFR-6.2）。
→ アラームアプリの正しさをユニットテストだけで検証できる状態を作る。

---

## 12.2 パッケージ構成

単一モジュール + パッケージ分割（T-012）。

```
com.<domain>.alarm
├── AlarmApplication.kt
├── di/                        Hilt モジュール
├── domain/
│   ├── model/                 Alarm, AlarmGroup, DateRule, DateOverride,
│   │                          AlarmSettings, RingPlan, SkipReason
│   ├── schedule/
│   │   ├── RingDecider.kt         willRing(alarm, date): RingDecision
│   │   ├── NextRingCalculator.kt  次の鳴動時刻の探索
│   │   ├── ConflictDetector.kt    除外日 × 予約日の衝突検出
│   │   └── HolidayCalendar.kt     内蔵祝日 + ユーザー編集の合成
│   └── usecase/
├── data/
│   ├── db/                    AlarmDatabase, Entity, Dao
│   ├── prefs/                 SettingsDataStore
│   ├── asset/                 BuiltInHolidaySource
│   └── repository/            AlarmRepository, DateRuleRepository, ...
├── alarm/                     ★ Platform 層
│   ├── AlarmScheduler.kt
│   ├── AlarmReceiver.kt
│   ├── BootReceiver.kt
│   ├── RingService.kt
│   ├── AlarmPlayer.kt
│   ├── VibrationController.kt
│   └── NotificationHelper.kt
├── work/                      DailyMaintenanceWorker
└── ui/
    ├── navigation/            ボトムナビ 3 タブ + 遷移
    ├── theme/
    ├── home/ alarmedit/ groupedit/ bulkcreate/
    ├── calendar/ schedule/ holiday/
    ├── ring/                  S-07 鳴動画面
    ├── settings/ permission/
    └── component/             共通 Composable
```

---

## 12.3 主要クラスと責務

### Domain 層

| クラス | 責務 | 依存 |
|---|---|---|
| `RingDecider` | `willRing(alarm, date): RingDecision` を判定する。**鳴らない理由も返す**（S-06 で表示するため） | なし（純粋） |
| `NextRingCalculator` | 指定日以降で最初に鳴る日時を求める。探索上限 400 日 | `RingDecider` |
| `ConflictDetector` | 除外日と予約日の衝突を検出する（双方向 / FR-4.5.1） | なし |
| `HolidayCalendar` | 内蔵祝日 + `HolidayCustomization` を合成し `isHoliday(date)` を返す | なし |
| `BulkAlarmGenerator` | 起点 + 間隔 + 個数から時刻リストを生成（日跨ぎ対応） | なし |

`RingDecision` は真偽値ではなく理由付きの型にする。

```kotlin
sealed interface RingDecision {
  data class Ring(val time: LocalTime) : RingDecision
  data class Skip(val reason: SkipReason) : RingDecision
}

enum class SkipReason {
  ALARM_DISABLED, GROUP_DISABLED, OUT_OF_VALID_PERIOD,
  DAILY_SKIPPED, OVERRIDE_SILENT, EXCLUDED_APP, EXCLUDED_GROUP,
  EXCLUDED_ALARM, HOLIDAY, NOT_SCHEDULED_DAY, UNRESOLVED_CONFLICT
}
```

**設計判断**: 理由を返す設計にしないと S-06（「鳴りません（祝日: 秋分の日）」）が実装できない。
`Boolean` を返す関数を後から拡張するのは面倒なので、最初から理由付きにする。

### Platform 層

| クラス | 責務 |
|---|---|
| `AlarmScheduler` | **再計算とアラーム登録の唯一の入口**。`reschedule()` だけを公開する |
| `AudioHardeningGuard` | Android 17 の音声制限（4.4b）に適合しているかを検査し、無音失敗を検出する |
| `AlarmReceiver` | `AlarmManager` からの発火を受ける `BroadcastReceiver`。すぐ `RingService` を起動する |
| `BootReceiver` | `BOOT_COMPLETED` / `MY_PACKAGE_REPLACED` / `TIMEZONE_CHANGED` / `TIME_SET` / `DATE_CHANGED` を受けて `reschedule()` |
| `RingService` | フォアグラウンドサービス。音・バイブ・自動停止タイマーを保持する |
| `AlarmPlayer` | `MediaPlayer` のラッパ。`USAGE_ALARM`、フェードイン、フォールバック |
| `NotificationHelper` | N-01〜N-03 の通知とチャネル管理 |
| `DailyMaintenanceWorker` | 日次の再計算・孤児レコード掃除・過去データ整理 |

---

## 12.4 AlarmScheduler — 再計算の単一入口

```kotlin
class AlarmScheduler @Inject constructor(
  private val repo: AlarmRepository,
  private val nextRingCalculator: NextRingCalculator,
  private val alarmManager: AlarmManager,
  private val clock: Clock,
) {
  private val mutex = Mutex()          // 再計算は必ず直列化する

  suspend fun reschedule() = mutex.withLock {
    // 1. 全アラームの次の鳴動時刻を計算
    val plans = repo.loadAll().mapNotNull { nextRingCalculator.next(it, clock.today()) }

    // 2. キャッシュを更新（11.5）
    repo.updateCachedNextRingAt(plans)

    // 3. 最も早い時刻を 1 件だけ登録（T-007）
    val earliest = plans.minByOrNull { it.at } ?: run {
      alarmManager.cancel(pendingIntent()); return@withLock
    }
    alarmManager.setAlarmClock(                       // T-008: Doze を貫通する
      AlarmClockInfo(earliest.at.toEpochMilli(), showIntent()),
      pendingIntent()
    )
  }
}
```

### 再計算を直列化する理由

「アラーム編集」「発火後の再計算」「日次メンテナンス」が同時に走ると、
古い計算結果で `AlarmManager` を上書きしてしまう可能性がある。
`Mutex` で直列化し、**常に最後の計算結果が残る**ようにする。

### 同時刻に複数アラームがある場合

`setAlarmClock` は 1 件しか登録できない。発火時に「その時刻に該当する全アラーム」を
改めて問い合わせ、まとめて処理する。

```
6:05 に 3 個のアラームが該当 → 1 回の発火 → 3 個をまとめて 1 つの鳴動として扱う
（音は最も優先度の高い 1 件のものを使う。UI には「ほか 2 件」と表示）
```

**[未確定 A-1]** 同時刻の複数アラームをどう扱うか（1 つにまとめる / 順に鳴らす）は要検討。
一括作成では通常同時刻にならないため、MVP は「まとめて 1 件として扱う」で進める。

---

## 12.5 鳴動シーケンス

```
AlarmManager
    │ (setAlarmClock で登録した時刻に発火)
    ▼
AlarmReceiver (BroadcastReceiver)
    │ ・goAsync() で短時間だけ処理を保持
    │ ・その時刻に該当するアラームを DB から特定
    │ ・startForegroundService(RingService)
    │   ※ 正確アラームの発火直後はバックグラウンド FGS 起動が許可される
    ▼
RingService (Foreground Service)
    │ ・startForeground(N-03 通知 + fullScreenIntent)
    │ ・AlarmPlayer.start()  … USAGE_ALARM / フェードイン
    │ ・VibrationController.start()
    │ ・自動停止タイマー開始（既定 5 分）
    │ ・RingSession を DB に保存（RINGING）
    ▼
RingActivity (全画面)
    │ ・setShowWhenLocked(true) / setTurnScreenOn(true)
    │ ・ロック画面の上に表示
    │
    ├─[停止]───────────────────────────────────┐
    │    ・RingSession を DISMISSED に          │
    │    ・group.skipRestOnDismiss なら          │
    │      DailySkip を作成 → N-01 通知（D-008）│
    │                                           │
    ├─[スヌーズ]────────────────────────────┐  │
    │    ・snoozeCount++                     │  │
    │    ・上限超過なら停止扱い               │  │
    │    ・N-02 通知を表示                    │  │
    │    ・snoozeMinutes 後を予約             │  │
    │                                        │  │
    └─[無操作で自動停止]──────────────────┐  │  │
         ・DailySkip は作らない（D-009）  │  │  │
         ・autoStopAction に従う          │  │  │
                                          ▼  ▼  ▼
                                    RingService.stopSelf()
                                          │
                                          ▼
                                 AlarmScheduler.reschedule()
```

### 鳴動中に次のアラーム時刻が来た場合（FR-5.9）

`RingService` は起動済みなので、新しい発火は既存サービスへ Intent で届く。
既存の鳴動を停止し、新しいアラームの鳴動へ差し替える。
スヌーズ待機中に定時アラームが来た場合は、**スヌーズを破棄する**（FR-6.6）。

---

## 12.6 再計算のトリガ一覧

| トリガ | 受け口 | 備考 |
|---|---|---|
| アラーム / グループ / 除外日の変更 | Repository の書き込み後に UseCase から呼ぶ | UI からは直接呼ばない |
| アラーム発火・鳴動終了 | `RingService` | |
| アプリ起動 | `AlarmApplication` / 最初の画面 | キャッシュの陳腐化対策（11.5） |
| 端末再起動 | `BootReceiver` (`BOOT_COMPLETED`) | Direct Boot は非対応（D-013） |
| アプリ更新 | `BootReceiver` (`MY_PACKAGE_REPLACED`) | |
| タイムゾーン / 時刻変更 | `BootReceiver` (`TIMEZONE_CHANGED` / `TIME_SET`) | |
| 日付変更 | `BootReceiver` (`DATE_CHANGED`) | |
| 日次セーフティネット | `DailyMaintenanceWorker` | 登録漏れの自己修復（NFR-1.5） |

**原則**: 再計算は**冪等**であること。何回呼んでも同じ結果になる。
「呼びすぎて壊れる」ことがないなら、迷ったら呼ぶ。

---

## 12.7 権限と状態の監視

`AlarmHealthChecker`（仮）が以下を集約し、S-01 のバナーと S-10 に供給する。

| チェック項目 | 判定方法 | 重大度 |
|---|---|---|
| 正確アラームの権限 | `AlarmManager.canScheduleExactAlarms()` | 致命的 |
| 通知の権限 | `POST_NOTIFICATIONS` の付与状態 | 致命的 |
| 全画面表示の権限 | `NotificationManager.canUseFullScreenIntent()` | 重大 |
| アラーム音量 | `AudioManager.getStreamVolume(STREAM_ALARM) == 0` | 重大 |
| おやすみモード | DND の設定でアラームがブロックされていないか | 重大 |
| 電池最適化 | `PowerManager.isIgnoringBatteryOptimizations()` | 警告 |
| 通知チャネル | チャネルが無効化されていないか | 重大 |

**設計判断**: 「権限がある / ない」だけでなく **「実際に鳴るか」**を基準にする。
音量 0 とおやすみモードは権限ではないが、鳴らない原因としては最頻出（NFR-1.4）。

---

## 12.8 エラー処理とフォールバック

| 事象 | 挙動 |
|---|---|
| 音源ファイルが読めない | 端末既定のアラーム音にフォールバック（NFR-1.3）。**無音にしない** |
| **音声が無言で失敗した**（Android 17 の音声制限 / 4.4b） | 再生開始後に実際に鳴っているかを検証し、鳴っていなければバイブ + 通知で異常を伝える。`requestAudioFocus()` の戻り値も必ずチェックする |
| 既定音も読めない | 最終手段として端末のバイブのみで鳴動し、通知で異常を伝える |
| 正確アラーム権限がない | **代替登録はしない（D-026）**。Android 17 では音声再生自体が無言で失敗しうるため、「たぶん鳴る」状態を作らず、鳴らせないことを明確に警告する |
| DB 読み込み失敗 | 起動時に検出し、エラー画面を出す。無言で「アラーム 0 件」にしない |
| 発火時にアラームが既に削除済み | 何もせず再計算のみ行う |
| 再計算中に例外 | ログを残し、日次メンテナンスで再試行する |

**原則**: **無言で鳴らないことだけは避ける。** 鳴らせない場合は必ずユーザーに伝える。

---

## 12.9 スレッドと並行性

| 処理 | スレッド |
|---|---|
| DB アクセス | Room の suspend 関数（IO ディスパッチャ） |
| 再計算 | IO + `Mutex` で直列化 |
| `RingDecider` の判定 | 呼び出し元のスレッド（純粋関数・軽量） |
| 音声再生 | `MediaPlayer` の内部スレッド |
| UI | Main。`StateFlow` を `collectAsStateWithLifecycle` で購読 |

`BroadcastReceiver` 内の非同期処理は `goAsync()` + 短時間で完結させ、
長い処理は `RingService` 側で行う（Receiver は 10 秒程度で殺される）。

---

## 12.10 テスト構成（NFR-6.1 / 4.7）

| 種別 | 対象 | 割合 |
|---|---|---|
| ユニット（純粋 Kotlin） | `RingDecider` / `NextRingCalculator` / `ConflictDetector` / `HolidayCalendar` / `BulkAlarmGenerator` | **最重点** |
| ユニット（Robolectric） | `AlarmScheduler` の登録内容、通知の組み立て | 中 |
| 実機（3 台） | 鳴動の確実性、Android 17 の音声制限、OEM 省電力（→ `04` の 4.7b） | **必須** |
| 結合（androidTest） | Room の DAO とマイグレーション | 中 |
| 手動 | 実機での鳴動（再起動後 / DND / 電池最適化 ON / 低電力モード） | 必須 |

開発ビルド限定のデバッグ画面を用意する（4.7）。

- 「10 秒後に鳴らす」
- 「任意の日付を指定して `willRing()` の結果と理由を見る」
- 「現在 `AlarmManager` に登録されている時刻を表示する」
- 「音声の実際の再生状態と `AudioHardening` の判定結果を表示する」（4.4b の検証用）

---

## 12.11 未確定の論点

| # | 論点 | 備考 |
|---|---|---|
| A-1 | 同時刻に複数アラームがある場合の扱い | MVP は「まとめて 1 件」で進める |
| ~~A-2~~ | ~~フォアグラウンドサービスの型~~ | **解決: `mediaPlayback`（D-025 / 4.4b）** |
| A-3 | `RingService` と `RingActivity` の役割分担 | 音の保持は Service、表示は Activity で確定。停止操作の受け口をどちらにするか要検討 |
| ~~A-4~~ | ~~権限が無い状態での不正確なアラームへのフォールバック~~ | **解決: 実装しない（D-026）** |
| A-5 | 音声の無言失敗をどう検知するか | 再生開始後の実測、`AudioHardeningGuard` の実装方法。フェーズ C で実機検証しながら決める |
