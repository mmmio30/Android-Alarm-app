# ドキュメント一覧

Android アラームアプリの要件定義ドキュメント。

| # | ファイル | 内容 |
|---|---|---|
| 01 | [01-product-overview.md](01-product-overview.md) | 背景・課題・コンセプト・スコープ・用語定義 |
| 02 | [02-functional-requirements.md](02-functional-requirements.md) | 機能要件（FR-1 〜 FR-9） |
| 03 | [03-domain-model.md](03-domain-model.md) | データモデル・鳴動判定ロジック・スケジューリング方式 |
| 04 | [04-android-constraints.md](04-android-constraints.md) | Android 固有の制約（権限・Doze・OEM 省電力）と対応方針 |
| 05 | [05-nonfunctional-requirements.md](05-nonfunctional-requirements.md) | 非機能要件（信頼性・性能・データ・保守性） |
| 06 | [06-open-questions.md](06-open-questions.md) | **未決事項 / 要判断リスト（← まずここに回答がほしい）** |
| 07 | [07-decision-log.md](07-decision-log.md) | 決定ログ（確定事項と仮決定） |
| 08 | [08-scope-and-roadmap.md](08-scope-and-roadmap.md) | MVP 範囲とフェーズ分け |
| 09 | [09-play-release.md](09-play-release.md) | Google Play 公開に必要な作業と審査上の論点 |

## 進め方

**要件 → 設計 → 実装**の順で進める（D-012）。配布は Google Play 公開（D-007）。

1. ~~配布方針・グループ仕様の決定~~ → 完了（D-007 〜 D-012）
2. ~~祝日データ・衝突時の扱い・Direct Boot・MVP 範囲の決定~~ → 完了（D-013 〜 D-017）
3. `06-open-questions.md` の残り（実機 / Play アカウント / アプリ名 / minSdk など）に回答
4. 設計フェーズ（画面設計 / データ設計 / アーキテクチャ / ロジック仕様）
5. 実装着手

## 現在の状況

- フェーズ A（要件確定）はほぼ完了。MVP 範囲を確定済み
- 確定事項 17 件（D-001 〜 D-017） / 仮決定 14 件 / 未決 10 件
- 残る未決は主に「手元の実機」「Play アカウント」「アプリ名」など環境・段取りの情報

## ステータス表記

- **確定** … 合意済み。覆すときは決定ログに理由を残す
- **仮** … こちらの推奨案。異論がなければ確定に昇格
- **要判断** … ユーザーの判断待ち
