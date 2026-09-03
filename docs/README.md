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

### 設計（フェーズ B）

| # | ファイル | 内容 |
|---|---|---|
| 10 | [10-screen-design.md](10-screen-design.md) | 画面一覧・遷移・ワイヤーフレーム・操作仕様 |
| 11 | [11-data-design.md](11-data-design.md) | Room のテーブル定義・インデックス・マイグレーション方針 |
| 12 | [12-architecture.md](12-architecture.md) | レイヤ構成・主要クラス・鳴動シーケンス・テスト構成 |
| 13 | [13-logic-spec.md](13-logic-spec.md) | **判定ロジックの確定仕様とテストケース一覧（実装完了の定義）** |
| 14 | [14-play-console-setup.md](14-play-console-setup.md) | **Play Console 申請の進め方（12人×14日の要件と分担）** |

## 進め方

**要件 → 設計 → 実装**の順で進める（D-012）。配布は Google Play 公開（D-007）。

1. ~~配布方針・グループ仕様の決定~~ → 完了（D-007 〜 D-012）
2. ~~祝日データ・衝突時の扱い・Direct Boot・MVP 範囲の決定~~ → 完了（D-013 〜 D-017）
3. ~~深夜アラーム・繰り返し・イヤホン対応の決定~~ → 完了（D-018 〜 D-020）
4. **設計フェーズ（画面設計 / データ設計 / アーキテクチャ / ロジック仕様）← いまここ**
5. 実装着手

環境情報（実機 / Play アカウント / アプリ名）は設計と並行して確認する。

## 現在の状況

- **フェーズ A（要件確定）完了。フェーズ B（設計）を実施中**
- 設計書 4 本（画面 / データ / アーキテクチャ / ロジック）を作成済み
- 確定事項 21 件（D-001 〜 D-021） / 仮決定 14 件 / 未決 7 件
- 残る未決は「手元の実機」「Play アカウント」「アプリ名」など環境・段取りの情報のみ

## ステータス表記

- **確定** … 合意済み。覆すときは決定ログに理由を残す
- **仮** … こちらの推奨案。異論がなければ確定に昇格
- **要判断** … ユーザーの判断待ち
