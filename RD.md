
# 要件定義書（再構成版）

## 1. はじめに（目的と概要）

### 1.1 アプリの目的

このアプリは、案件（プロジェクト）のフェーズごとのスケジュール・納期設定を行う補助ツールです。高機能なガントチャートではなく、簡単に納期を管理したいユーザー（個人事業主や個人のイベント管理者など）を対象としています。

### 1.2 想定ユースケース

- 案件の納期を決めたい
- Googleカレンダーと手動連携したい
- シンプルにフェーズ管理したい

### 1.3 システム環境

- 対応OS: iOS (PWA形式)
- 開発環境: FlutterFlow（無料版）、Firebase Firestore（無料版）

---

## 2. アプリの基本設計

### 2.1 機能一覧

- 案件登録・管理
- フェーズの追加・削除・更新
- CSVエクスポート（Googleカレンダー用）
- アーカイブ / ごみ箱管理

### 2.2 データ構造

#### anken コレクション

| フィールド名 | 型 | 内容 |
|--------------|----|------|
| id | string | Firestore自動ID |
| uid | string | 匿名ユーザーID |
| fullName | string | 案件正式名 |
| shortName | string | 案件略称 |
| status | string | active / archived / deleted |
| createdAt / updatedAt / deletedAt | timestamp | 各タイミング |

#### phases コレクション

| フィールド名 | 型 | 内容 |
|--------------|----|------|
| id | string | フェーズID |
| ankenId | string | 紐づく案件ID |
| order | number | 表示順 |
| phaseName | string | フェーズ名 |
| startDate / endDate | date | 日付 |
| duration | number | 日数 |
| createdAt / updatedAt / deletedAt | timestamp | 各タイミング |

---

## 3. 画面設計とUIフロー

### 3.1 画面一覧

- SplashPage（起動後すぐにHomePageへ）
- HomePage（案件一覧とタブ表示）
- CreateAnkenDialog（案件登録用）
- PhasePage（フェーズ管理）

### 3.2 ユーザーフロー

- 起動 → SplashPage → HomePage → 新規登録または既存案件選択 → PhasePage
- フェーズ追加・修正 → CSV出力

---

## 4. 開発手順（FlutterFlow 実装ガイド）

### 4.1 プロジェクト初期設定

- FlutterFlowで新規プロジェクト作成
- Firestoreコレクション（anken, phases）設定

### 4.2 UI構成と動作設定

- 案件一覧リスト（チェックボックス付き）
- AppBar、フェーズリストのリピート構造
- 各アクションの実装（onEndDateChanged, onDeleteなど）
- 一括アーカイブ・削除ボタン追加

---

## 5. CSVエクスポートとGoogleカレンダー連携

- CSVフォーマット：
  - Subject: 短縮名_フェーズ名
  - Start/End Date: 終了日翌日まで
  - All Day: TRUE
  - Description: 案件正式名
  - Private: TRUE
- GoogleカレンダーへのURLスキーム：
  - 新規作成: `googlechrome://calendar.google.com/calendar/u/0/r/settings/createcalendar?pli=1`
  - インポート: `googlechrome://calendar.google.com/calendar/u/0/r/settings/export?pli=1`

---

## 6. データ更新ルールと注意点

- startDateは編集不可（前のフェーズのendDate+1日で自動設定）
- durationはendDate変更時に再計算
- 保存ボタン不要：各編集時に即Firestore反映
- 削除は論理削除（deletedAt設定）

---

## 7. テストと最終調整

- UI/UXの調整（色、位置、操作性）
- 操作ガイドや案内表示の追加
- セキュリティ：匿名ユーザーのデータ制限確認

---

## 8. 開発管理と改訂履歴

- 開発のリスク対策（複雑化の防止、構造の単純化）
- ChatGPTとのやりとり要点まとめ
- GitHubでの改訂履歴管理

---


---

## 9. PhasePage の詳細仕様

### 9.1 画面構成

| UI要素 | ウィジェット | 説明 |
|--------|--------------|------|
| 案件名表示 | Text | `anken.fullName` を表示 |
| 開始日 | DatePicker | `anken.startDate`。初回のみ編集可（フェーズ1のstartDateと同期） |
| フェーズリスト | Column + Repeated Widget | `phases` の order 昇順に表示 |
| フェーズ名 | TextField | `phaseName` を編集可能。変更時に保存アクション |
| 終了日 | DatePicker | `endDate` を編集可能。変更時に `duration` 再計算・保存 |
| 期間（日数） | Text | `startDate` 〜 `endDate` から自動計算 |
| フェーズ削除 | IconButton | 該当フェーズを論理削除 |
| フェーズ追加ボタン | ＋（FloatingActionButton等） | 現在のフェーズの次に追加。startDateは直前の `endDate +1日` |
| CSV出力 | Button | 現在の案件のフェーズ一覧をCSVで出力 |

---

### 9.2 データ処理・ロジック

- `startDate` は常に自動計算：
  - フェーズ1：`anken.startDate`
  - フェーズ2以降：直前の `endDate + 1日`

- `duration` の計算：
  - `endDate - startDate + 1`
  - `onEndDateChanged` 発火時に自動再計算

- `phaseName`、`endDate` の編集はリアルタイムで `Firestore` に反映
  - 保存ボタンなし、入力変更アクションで即反映（UX簡素化）

- 論理削除：
  - 削除時は `deletedAt` を現在時刻に、UI上から非表示に

- 再描画：
  - フェーズの編集・追加・削除ごとに `phases` 配列全体を再取得 or State更新

---

### 9.3 Firebase Firestore連携

- `phases` コレクションは `order` または `startDate` 昇順でクエリ
- 各フェーズに必要なフィールド：
  - `id`, `ankenId`, `order`, `phaseName`, `startDate`, `endDate`, `duration`

---

### 9.4 注意点とベストプラクティス

- `startDate` の手動編集を禁止（整合性維持）
- 新規フェーズ追加時は「挿入位置の前フェーズの `endDate` を参照」
- 編集/削除により `startDate` や `duration` を自動的に再計算・更新
- `phases` の UI は常に最新状態に同期されるように構成

---


---

### 9.5 一時データとアクションのパラメータ設計

#### `newEndDate` の役割と処理の流れ

- **用途**：
  - DatePickerで選択された新しい終了日を一時的に保持する。
  - UI操作直後のローカル変数として利用され、最終的に `endDate` として保存。

- **典型的な使用パターン（FlutterFlow）**：
  1. `DatePicker` の `onChanged` で `newEndDate` に保存
  2. カスタムアクション `onEndDateChanged(newEndDate)` を呼び出し
  3. `duration` を `startDate` から再計算
  4. `Firestore` の該当フェーズドキュメントに `endDate` と `duration` を保存

- **newEndDate の注意点**：
  - Firestoreには保存しない（UI一時処理専用）
  - StateまたはApp変数で管理
  - 他のフェーズの `startDate` 再計算トリガーとなる

---

### 9.6 PhaseBox（カスタムウィジェット）設計指針

- **パラメータ一覧**：

| パラメータ名 | 型 | 説明 |
|--------------|----|------|
| phaseId | String | 該当フェーズID（Firestoreドキュメント参照用） |
| phaseName | String | フェーズ名（双方向バインディング） |
| endDate | DateTime | 終了日（編集可能） |
| duration | Integer | 期間（日数） |
| onDelete | Action | 削除ボタン押下時の処理 |
| onEndDateChanged | Action | 終了日更新時の処理（newEndDateを引数） |
| onPhaseNameChanged | Action | フェーズ名変更時の処理 |

- **パラメータ設計のポイント**：
  - `onEndDateChanged(newEndDate)` は、更新後に即再計算と保存を行う
  - `onDelete` は `deletedAt` をセットし、リストから非表示とする
  - TextFieldの `onChanged` に `onPhaseNameChanged` を紐付けて即反映

---

### 9.7 再描画と同期処理の要点

- **フェーズの編集／追加／削除のたびに以下を実行**：
  - `phases` リストの再構築（State更新 or 再クエリ）
  - `startDate` 自動再設定（前フェーズ `endDate +1`）
  - `duration` 再計算と再保存

- **ルールの要点**：
  - 開始日は手動変更不可（自動で更新）
  - `order` が前後関係のキー（昇順表示）
  - フェーズ挿入位置は `order` を動的に調整 or 挿入後に並べ替え

---

### 9.8 よくあるミスと防止策

| ミス | 防止策 |
|------|--------|
| startDateが上書き可能になっている | UIで `disabled: true`、または編集ロジックを排除 |
| duration未更新 | endDate変更時に `duration` 自動再計算処理を設置 |
| フェーズ順がバラバラ | Firestore取得時に必ず `order` または `startDate` 昇順クエリ |
| newEndDateが保存される | 保存時には `newEndDate` → `endDate` に代入する処理を明示 |

---

