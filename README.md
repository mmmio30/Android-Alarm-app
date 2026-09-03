# Android-Alarm-app

「大量のアラームを、束ねて・まとめて作り、まとめて操作できる」Android アラームアプリ。

**現在の状況: 要件定義 完了 → 設計フェーズ 実施中。** 実装はまだ開始していない。

## コアコンセプト

寝坊防止のために 5 分おきなど複数アラームを設定する人向けに、
**アラームの管理コストをゼロに近づける**ことだけを差別化軸とする。

1. **グループ** — 複数アラームを束ねて 1 操作で一括 ON/OFF
2. **一括作成** — 「6:00 から 5 分おきに 6 個」を 1 操作で作成
3. **カレンダー先行入力** — 予約日・除外日・祝日を事前にまとめて登録

さらに **「グループ内で 1 つ停止したら、その日の残りは自動スキップ」** を備え、
5 分おきアラームの最大の不満（起きた後も鳴り続ける）を解決する。

音・バイブ・スヌーズ・祝日自動 OFF などは既存アプリ同等を目標とする。

## ドキュメント

| 分類 | 内容 |
|---|---|
| 要件定義 | `docs/01` 〜 `docs/09` |
| 設計 | `docs/10` 〜 `docs/13` |

索引は [`docs/README.md`](docs/README.md)。
判断待ちの項目は [`docs/06-open-questions.md`](docs/06-open-questions.md)、
確定事項は [`docs/07-decision-log.md`](docs/07-decision-log.md) に集約している。

## 技術スタック（予定）

Kotlin / Jetpack Compose / Material 3 / Room / DataStore / Hilt /
AlarmManager (`setAlarmClock`) / WorkManager
