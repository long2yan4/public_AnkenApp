
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
| kaishibi |	timestamp	| 案件全体の開始日（ユーザーが唯一編集可能な開始日）
| hasScheduleConflict	| boolean	| 案件全体のスケジュールに矛盾があるか否か（true: 矛盾あり / false: 矛盾なし）


#### phases コレクション

| フィールド名 | 型 | 内容 |
|--------------|----|------|
| id | string | フェーズID |
| ankenId | string | 紐づく案件ID |
| order | number | 並び順の基準。0始まりで管理され、削除・追加時に再計算される。 |
| orderDisplay | number | ユーザー向けに表示される順序（= order + 1）。UIに1始まりで表示。 |
| phaseName | string | フェーズ名 |
| startDate	| timestamp	 | フェーズの開始日|
| endDate	| timestamp	| フェーズの終了日 |
| duration	| number	| フェーズの期間（日数）。endDate - startDate + 1 で計算。負の値や0も許容。|
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

### 4.3 今後の開発手順（推奨実施順）2025/5/29

1. **PhasePageに常に3件フェーズが表示される状態にする**
   - 新規案件登録時でも、最低3件の空フェーズが自動生成されて表示されるようにする。
   - 空フェーズには初期状態として `phaseName: ''` などのプレースホルダ値を持たせる。

2. **endDate → duration の自動計算処理を実装**
   - 各 `PhaseBox` 内でユーザーが `endDate` を選択した時点で、
     - `duration = endDate - startDate + 1` を算出
     - Firestore の該当ドキュメントに即時保存
   - startDate は前のフェーズの `endDate + 1` または全体開始日から決定される。

3. **PhaseBoxを完成系に近づける**
   - UIを整理し、適切な状態管理（phaseDocのバインド）や警告表示などを加える。
   - TextField、DatePicker、削除ボタンなどのレイアウトを整備。

4. **案件ページのタブUI・タイトルなどの骨組みを作成**
   - FlutterFlowでタブUI、タイトル、日付などを配置。
   - PhasePage とのデータ連携ができるよう、`ankenid` 経由の参照を設計。

5. **一覧や履歴など案件ページの拡張機能に進む**
   - 過去のフェーズ表示、ログ記録、他ページとの遷移などの機能追加。

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
# 要件定義書（更新案）

## 2. アプリの基本設計

### 2.2 データ構造

#### anken コレクション

| フィールド名 | 型 | 内容 |
|---|---|---|
| id | string | Firestore自動ID |
| uid | string | 匿名ユーザーID |
| fullName | string | 案件正式名 |
| shortName | string | 案件略称 |
| status | string | active / archived / deleted |
| createdAt / updatedAt / deletedAt | timestamp | 各タイミング |
| **kaishibi** | **timestamp** | **案件全体の開始日（ユーザーが唯一編集可能な開始日）** |
| **hasScheduleConflict** | **boolean** | **案件全体のスケジュールに矛盾があるか否か（true: 矛盾あり / false: 矛盾なし）** |

#### phases コレクション

| フィールド名 | 型 | 内容 |
|---|---|---|
| id | string | フェーズID |
| ankenId | string | 紐づく案件ID |
| order | number | 並び順の数値（0から始まる整数） |
| orderDisplay | string | 表示用の並び順（例: "1" "2"） |
| phaseName | string | フェーズ名 |
| **startDate** | **timestamp** | **フェーズの開始日** |
| endDate | timestamp | フェーズの終了日 |
| **duration** | **number** | **フェーズの期間（日数）。endDate - startDate + 1 で計算。負の値や0も許容。** |
| note | string | 備考 |
| createdAt / updatedAt / deletedAt | timestamp | 各タイミング |

---

## 9. スケジュール関連機能

### 9.1 スケジュール表示と管理の基本原則

* フェーズのスケジュールは、`PhasePage`の`ListView`で表示される各`PhaseBox`で管理する。
* 各フェーズには`startDate`と`endDate`が存在し、それに基づいて`duration`（期間）が計算される。
* **「案件全体の開始日」は`anken`コレクションの`kaishibi`フィールドで管理され、これがユーザーが直接編集可能な唯一の開始日となる。**
* **各フェーズの`startDate` (`order: 0`を除く) は、直前のフェーズの`endDate`に1日加算された日付として自動的に計算・設定される（ユーザーは直接編集不可）。**
* **`duration`は`endDate - startDate + 1`（日数）で計算される。この計算結果は負の値や0も許容する。**
* ユーザーの入力（`endDate`の変更、`kaishibi`の変更）を優先し、システムは入力された日付を受け入れる。
* **論理的な矛盾（例: `duration`が負や0、または前後フェーズ間で日付のずれ）が生じた場合、システムは自動修正は行わず、ユーザーに「見直し対象」として明確に提示する。**
* 各操作（追加・削除・日程変更）の後、案件全体のスケジュールに矛盾があるか否かを判断し、`anken.hasScheduleConflict`フラグを更新する。
* `anken.hasScheduleConflict`フラグが`true`の場合、`PhasePage`の画面上部に全体的な警告メッセージを表示し、ユーザーに注意を促す。

### 9.9.1 `duration`の計算ロジック

`duration`は、以下のカスタム関数を使用して計算される。

**カスタム関数名**: `calculateDurationDays`
**引数**:
* `startDate` (DateTime型)
* `endDate` (DateTime型)
**戻り値**: `int` (日数)

```dart
// calculateDurationDays カスタム関数
// フェーズの期間を計算する。
// 終了日が開始日より前の場合は負の値、同じ場合は0を返す。
int calculateDurationDays(DateTime startDate, DateTime endDate) {
  if (startDate == null || endDate == null) {
    // 必要に応じてエラーハンドリングまたはデフォルト値を返す
    return 0; // nullの場合は期間を0とする
  }
  final Duration difference = endDate.difference(startDate);
  return difference.inDays + 1; // 日数を計算 (負になる可能性あり、0になる可能性あり)
}

### 9.1.1 画面構成

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
### 9.1.2 Page State一覧
| 変数名                   | 型                | 説明                                          |
| --------------------- | ---------------- | ------------------------------------------- |
| `SelectedAnkenId`     | `String`         | 現在編集中または表示対象の案件ID。フェーズ取得などの条件指定に使用。         |
| `phaseNewEndDate`     | `DateTime`       | フェーズの終了日変更時に一時的に保持する日付。                     |
| `selectedStartDate`   | `DateTime`       | 現在選択されている案件の開始日。                            |
| `deletedOrder`        | `Integer`        | 削除対象のフェーズのorder番号。削除後の順序再計算で使用。             |
| `phasesToDeleteCheck` | `List<Document>` | 削除判定用にFirestoreから取得したフェーズ一覧。                |
| `updatedPhaseList`    | `List<Document>` | フェーズ削除・追加・更新後の最新フェーズ一覧。UI表示や保存処理の基データとして利用。 |
| `orderValue`          | `Integer`        | フェーズ順序再計算のループ内カウンターとして利用。                   |

SelectedAnkenId: 案件に紐づくフェーズを取得・操作する際の条件IDとして使用。
phaseNewEndDate: フェーズ終了日の更新操作時に新しい日付を保持し、保存処理へ引き渡す。
selectedStartDate: 案件開始日の管理。UI表示や日付計算の基準となる。
deletedOrder: 削除対象フェーズのorderを保持し、削除後のorder再割り当てに使用。
phasesToDeleteCheck: フェーズ削除前に最低件数判定を行うための一覧データ。
updatedPhaseList: 削除や追加などフェーズリスト更新後に最新の状態を保持しUIに反映。
orderValue: フェーズ順序の再計算を行う際にループインデックスとして使用。

# 要件定義書（更新案）

## 2. アプリの基本設計

### 2.2 データ構造

#### anken コレクション

| フィールド名 | 型 | 内容 |
|---|---|---|
| id | string | Firestore自動ID |
| uid | string | 匿名ユーザーID |
| fullName | string | 案件正式名 |
| shortName | string | 案件略称 |
| status | string | active / archived / deleted |
| createdAt / updatedAt / deletedAt | timestamp | 各タイミング |
| **kaishibi** | **timestamp** | **案件全体の開始日（ユーザーが唯一編集可能な開始日）** |
| **hasScheduleConflict** | **boolean** | **案件全体のスケジュールに矛盾があるか否か（true: 矛盾あり / false: 矛盾なし）** |

#### phases コレクション

| フィールド名 | 型 | 内容 |
|---|---|---|
| id | string | フェーズID |
| ankenId | string | 紐づく案件ID |
| order | number | 並び順の数値（0から始まる整数） |
| orderDisplay | string | 表示用の並び順（例: "1" "2"） |
| phaseName | string | フェーズ名 |
| **startDate** | **timestamp** | **フェーズの開始日** |
| endDate | timestamp | フェーズの終了日 |
| **duration** | **number** | **フェーズの期間（日数）。endDate - startDate + 1 で計算。負の値や0も許容。** |
| note | string | 備考 |
| createdAt / updatedAt / deletedAt | timestamp | 各タイミング |

---

## 9. スケジュール関連機能

### 9.1 スケジュール表示と管理の基本原則

* フェーズのスケジュールは、`PhasePage`の`ListView`で表示される各`PhaseBox`で管理する。
* 各フェーズには`startDate`と`endDate`が存在し、それに基づいて`duration`（期間）が計算される。
* **「案件全体の開始日」は`anken`コレクションの`kaishibi`フィールドで管理され、これがユーザーが直接編集可能な唯一の開始日となる。**
* **各フェーズの`startDate` (`order: 0`を除く) は、直前のフェーズの`endDate`に1日加算された日付として自動的に計算・設定される（ユーザーは直接編集不可）。**
* **`duration`は`endDate - startDate + 1`（日数）で計算される。この計算結果は負の値や0も許容する。**
* ユーザーの入力（`endDate`の変更、`kaishibi`の変更）を優先し、システムは入力された日付を受け入れる。
* **論理的な矛盾（例: `duration`が負や0、または前後フェーズ間で日付のずれ）が生じた場合、システムは自動修正は行わず、ユーザーに「見直し対象」として明確に提示する。**
* 各操作（追加・削除・日程変更）の後、案件全体のスケジュールに矛盾があるか否かを判断し、`anken.hasScheduleConflict`フラグを更新する。
* `anken.hasScheduleConflict`フラグが`true`の場合、`PhasePage`の画面上部に全体的な警告メッセージを表示し、ユーザーに注意を促す。

### 9.9.1 `duration`の計算ロジック

`duration`は、以下のカスタム関数を使用して計算される。

**カスタム関数名**: `calculateDurationDays`
**引数**:
* `startDate` (DateTime型)
* `endDate` (DateTime型)
**戻り値**: `int` (日数)

```dart
// calculateDurationDays カスタム関数
// フェーズの期間を計算する。
// 終了日が開始日より前の場合は負の値、同じ場合は0を返す。
int calculateDurationDays(DateTime startDate, DateTime endDate) {
  if (startDate == null || endDate == null) {
    // 必要に応じてエラーハンドリングまたはデフォルト値を返す
    return 0; // nullの場合は期間を0とする
  }
  final Duration difference = endDate.difference(startDate);
  return difference.inDays + 1; // 日数を計算 (負になる可能性あり、0になる可能性あり)
}

### 9.9.2 startDateの決定ロジック
各フェーズのstartDateは、以下のルールで決定される（ユーザーは直接編集不可）。

order: 0のフェーズのstartDate:
常にanken.kaishibiと同じ値となる。
anken.kaishibiが変更された場合は、このフェーズのstartDateも自動的に更新される。
order: 1以降のフェーズのstartDate:
常に「一つ前のorderのフェーズのendDateに1日加算した日付」となる。
前のフェーズのendDateが変更された場合や、フェーズの追加・削除によりorderが変動した場合、自動的に再計算・更新される。
### 9.9.3 スケジュール矛盾の検知とUI表示
以下の条件で、スケジュールに論理的な矛盾があると判断し、ユーザーに見直しを促す。

個別のフェーズにおける矛盾（PhaseBoxレベルの警告）

durationが負の値、または0である場合。
表示条件: phaseDoc.duration <= 0
表示内容: 「期間が不正です（〇日）。終了日を見直してください。」
視覚的フィードバック: duration表示のテキストを赤文字にする、背景色を薄い赤にするなど。
order: 0以外のフェーズにおいて、自身のstartDateが「前のフェーズのendDate + 1日」と一致しない場合。
表示条件: phaseDoc.order > 0 && phaseDoc.startDate != (previousPhaseEndDate.add(Duration(days: 1))) (ここでpreviousPhaseEndDateは、ListViewのループで前のフェーズから渡されるパラメータ)
表示内容: 「日程にずれがあります。見直しが必要です。」
視覚的フィードバック: フェーズ名のテキスト色を変える、アイコンを表示するなど。
(order: 0のフェーズにおいて、startDateがanken.kaishibiと一致しない場合も同様の警告を表示する。)
案件全体の矛盾（PhasePage全体の警告）

anken.hasScheduleConflictフィールドがtrueの場合に、PhasePageの最上部などに警告バナーを表示する。
表示内容: 「日程に矛盾があります。見直しが必要です。」
視覚的フィードバック: 赤い背景のバナー、警告アイコンなど。
このhasScheduleConflictフラグは、以下の「9.9.4 hasScheduleConflictフラグの更新ロジック」で定義される。
### 9.9.4 hasScheduleConflictフラグの更新ロジック
anken.hasScheduleConflictフラグは、以下のカスタム関数を呼び出すことで更新される。

カスタム関数名: checkPhaseConflicts
引数:

phasesList: List&lt;DocumentSnapshot> (現在の案件に属する、order昇順でソートされたフェーズのリスト)
ankenKaishibi: DateTime (案件全体の開始日 - anken.kaishibiから取得) 戻り値: boolean (全体として矛盾がある場合はtrue、なければfalse)
checkPhaseConflictsカスタム関数ロジック:



// checkPhaseConflicts(phasesList, ankenKaishibi)
// 案件全体のフェーズスケジュールに矛盾があるか否かを判断する
bool checkPhaseConflicts(List<DocumentSnapshot> phasesList, DateTime ankenKaishibi) {
    if (phasesList == null || phasesList.isEmpty) {
        return false; // フェーズがない場合は矛盾なし
    }

    // 最初のフェーズの期待される開始日として anken.kaishibi を使用
    DateTime expectedStartDateForCurrentPhase = ankenKaishibi;

    for (int i = 0; i < phasesList.length; i++) {
        DocumentSnapshot currentPhaseDoc = phasesList[i];
        Map<String, dynamic> currentPhaseData = currentPhaseDoc.data() as Map<String, dynamic>;

        DateTime currentActualStartDate = currentPhaseData['startDate']?.toDate();
        DateTime currentActualEndDate = currentPhaseData['endDate']?.toDate();
        int? currentCalculatedDuration = currentPhaseData['duration'] as int?; // Nullableに対応

        // 1. durationが負またはゼロであるかチェック
        if (currentCalculatedDuration == null || currentCalculatedDuration <= 0) {
            return true; // 矛盾あり
        }

        // 2. startDateが期待される日付と一致しないかチェック
        if (currentActualStartDate == null) {
             return true; // startDateがnullなら矛盾あり
        }

        if (currentActualStartDate != expectedStartDateForCurrentPhase) {
            return true; // 期待されるstartDateと異なる場合、矛盾あり
        }

        // 次のループのために、次のフェーズの期待される開始日を更新
        if (currentActualEndDate == null) {
            return true; // 現在のフェーズのendDateがnullなら矛盾あり
        }
        expectedStartDateForCurrentPhase = currentActualEndDate.add(Duration(days: 1));
    }

    return false; // 全てのチェックをパスすれば矛盾なし
}

このcheckPhaseConflictsカスタム関数は、以下の各イベント後のアクションフローの最後で呼び出され、ankenドキュメントのhasScheduleConflictフラグを更新する。

anken.kaishibiが変更された時
任意のフェーズのendDateが変更された時
PhaseBoxが追加/削除された時
### 9.9.5 各操作における更新ロジックのフロー（変更点）
a. anken.kaishibi変更時
UIからの入力受付: 案件全体の開始日を編集するDatePickerでユーザーが日付を選択。
Firestoreへの更新:
ankenドキュメントのkaishibiを更新。
phases.order: 0のドキュメントのstartDateをanken.kaishibiと同じ値に更新。
phases.order: 0のendDateは変更しない。
phases.order: 0のdurationをcalculateDurationDays(新しいstartDate, 既存のendDate)で再計算し、更新。
hasScheduleConflictフラグの更新:
checkPhaseConflictsカスタム関数を呼び出し、anken.hasScheduleConflictを更新。
UI更新: PhasePageの警告バナーの表示/非表示を更新。PhaseBoxレベルの警告も更新される。
b. PhaseBoxのendDate変更時
UIからの入力受付: PhaseBox内のDatePickerでユーザーが日付を選択（制限なし）。
Firestoreへの更新:
該当フェーズドキュメントのendDateをユーザーが選択した新しい日付に更新。
durationをcalculateDurationDays(既存のstartDate, 新しいendDate)で再計算し、更新。
hasScheduleConflictフラグの更新:
checkPhaseConflictsカスタム関数を呼び出し、anken.hasScheduleConflictを更新。
UI更新: PhasePageの警告バナーの表示/非表示を更新。PhaseBoxレベルの警告も更新される。
c. PhaseBox削除時
フェーズの削除: 該当フェーズを論理削除。
orderの繰り上げとstartDateの自動再計算（自動調整）:
削除されたフェーズより後の全てのフェーズのorderとorderDisplayを繰り上げる。
orderが繰り上がったフェーズのstartDateは、その新しいorder位置における「前のフェーズのendDate + 1日」として自動的に再計算し、Firestoreに保存する。（order: 0になるフェーズはanken.kaishibiに合わせる）。
durationも新しいstartDateと既存のendDateに基づいて再計算し、保存。
hasScheduleConflictフラグの更新:
checkPhaseConflictsカスタム関数を呼び出し、anken.hasScheduleConflictを更新。
UI更新: PhasePageの警告バナーの表示/非表示を更新。PhaseBoxレベルの警告も更新される。
d. PhaseBox追加時
新しいフェーズの挿入:
新しいフェーズドキュメントをFirestoreに作成し、適切なorderを設定。
startDateは、直前のフェーズのendDate + 1日としてシステムが自動計算して設定。（order: 0として挿入する場合はanken.kaishibiに合わせる）。
endDateはユーザーにデフォルト値を提示するか、後から入力させる。
durationも計算し、保存。
既存フェーズのorder繰り下げとstartDateの自動再計算（自動調整）:
挿入されたフェーズ以降の既存フェーズのorderとorderDisplayを繰り下げる。
orderが繰り下がったフェーズのstartDateは、その新しいorder位置における「新しい前のフェーズのendDate + 1日」として自動的に再計算し、Firestoreに保存する。
durationも新しいstartDateと既存のendDateに基づいて再計算し、保存。
hasScheduleConflictフラグの更新:
checkPhaseConflictsカスタム関数を呼び出し、anken.hasScheduleConflictを更新。
UI更新: PhasePageの警告バナーの表示/非表示を更新。PhaseBoxレベルの警告も更新される。



### 9.2 データ処理・ロジック（修正前）

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
### 9.6.1 PhasePage パラメータ定義（最新版）

## 概要
`PhasePage` は、「案件（Anken）」に関連するフェーズ情報（Phase）を一覧・編集するためのページである。新規登録時と既存案件表示時で処理を分岐する必要があるため、以下のようにパラメータ設計がなされている。

---

## 📌 パラメータ一覧

| パラメータ名 | 型 | 必須 | 用途 | 備考 |
|--------------|----|------|------|------|
| `ankenDoc` | Document Reference（Ankens コレクション） | ✔️ | 対象となる案件のドキュメント参照 | 新規・既存どちらの場合でも共通で渡す |
| `isNew` | Boolean | ✔️ | このページが「新規作成フロー」からの遷移かどうかを判定する | true の場合は初期フェーズを自動生成 |

---

## ✅ パラメータの使用目的

- `ankenDoc`  
  - 関連するフェーズ（phases コレクション）を取得するためのフィルタキー（ankenID）として使用。
  - 表示する案件名や開始日などの情報を表示するためにも使用。

- `isNew`  
  - `true` の場合：`onLoad` アクションで初期フェーズ（3件）を自動生成する。endDate には currentDate + 1日, +2日, +3日 などを設定。
  - `false` の場合：既存の phase データを読み込むのみで、登録処理は行わない。

---

## 🧩 想定される遷移元とパラメータの渡し方

### 1. 新規登録時（実装済）
- 遷移元：AnkenDialog
- 渡すパラメータ：
  - `ankenDoc`：Anken コレクションに作成された直後の Document Reference
  - `isNew`：`true`

### 2. 既存案件からの遷移（未実装）
- 遷移元（予定）：AnkenList（案件一覧ページなど）
- 渡すパラメータ：
  - `ankenDoc`：選択された案件の Document Reference
  - `isNew`：`false`

---

## 💡 備考・今後の拡張

- `isNew` フラグは将来的に `mode`（例：`view` / `edit` / `new`）として拡張することも可能。
- `ankenDoc.id`（String）は必要に応じて取得できるため、PhaseBox などでは Document Reference を渡す設計が推奨される。


### 9.6.2 PhaseBox（カスタムウィジェット）設計指針

- **パラメータ一覧**：

| パラメータ名 | 型 | 説明 |
|--------------|----|------|
| phaseRef | Document Reference | このフェーズの Firestore ドキュメント参照。削除・更新処理に使用。 |
| phaseDoc | Document | Firestore の `phases` コレクションの 1 ドキュメント（全情報を保持） |
| phaseId | String | 該当フェーズID（Firestoreドキュメント参照用） |
| phaseName | String | フェーズ名（双方向バインディング） |
| endDate | DateTime | 終了日（編集可能） |
| duration | Integer | 期間（日数） |
| onDelete | Action | 削除ボタン押下時の処理 |
| onEndDateChanged | Action | 終了日更新時の処理（newEndDateを引数） |
| onPhaseNameChanged | Action | フェーズ名変更時の処理 |
| orderDisplayName	| String	| ユーザー向け表示用の順序名（1始まり、例：第1フェーズ、第2フェーズ）| 

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
### 9.9 各種ロジックの要点と注意点

#### 9.9.1 `duration` の計算ロジック

- 対象式:  
  `duration = endDate - startDate + 1（日数）`
- startDate は以下のように決定：
  - 第1フェーズ：案件の全体開始日
  - 第2フェーズ以降：前フェーズの `endDate + 1`

- 保存タイミング：
  - DatePicker で `endDate` 選択後、即 `Update Document` でFirestoreに保存。

#### 9.9.2 `order` の扱いと再計算

- Phaseの並び順表示用に使用。
- 初期作成時は昇順（0, 1, 2…）で付与。
- 追加・削除・並び替えのたびに、orderの自動再計算を行う。

#### 9.9.3 `endDate` 入力時のエラー対策

| ケース | 制約または対策内容 |
|--------|-------------------|
| 全体の開始日より前の選択 | DatePickerのminDate制限、または警告表示 |
| 前フェーズより前の日付 | エラー表示 or 選択不可に |
| 後フェーズより後の日付 | duration整合が取れなくなるため無効化または警告 |

#### 9.9.4 フェーズ追加・削除に伴う処理

- **追加時**：
  - `order` を再割り振り。
  - 挿入位置に応じて周辺フェーズの `startDate`, `duration` 再計算。
- **削除時**：
  - 残りフェーズの `order` 再計算。
  - 該当フェーズの削除により、他フェーズの `duration` に影響が出る場合は連動再計算。

#### 9.9.5 全体 `startDate` の変更時対応

- 案件の全体開始日を更新した場合は、すべてのフェーズの `startDate`, `duration` を再計算。
- 特殊処理として、startDate更新のトリガーに応じた一括更新関数が必要。

---

### 9.10 補足方針・運用メモ

- 基本的に **ユーザーによる「保存ボタン」は設置せず、即時保存**の方式を採用。
- 一時的に整合性が崩れるのは許容しつつ、次の操作で修正される想定。
- Cloud Function等の導入は今後検討。
- PhaseBoxはListViewで繰り返し表示され、各 `phaseDoc` を個別に参照できている。

---
### 10 PhasePage 遷移と phaseBox 制御に関する設計メモ

## 1. PhasePage 遷移パターン

| パターン             | 遷移元            | 渡されるデータ                | 遷移先       | 実装状況   |
|----------------------|-------------------|-------------------------------|--------------|------------|
| 新規登録時           | AnkenDialog       | ankenDoc（Document Reference）| PhasePage    | ✅ 実装済み |
| 既存案件表示（予定） | AnkenList（仮）   | ankenDoc（Document Reference）| PhasePage    | ⛔ 未実装   |

- 現在、PhasePage に遷移してくるのは **新規作成後のみ**。
- 既存案件からの遷移（AnkenList → PhasePage）は **今後の実装予定**。

---

## 2. PhasePage 内の挙動制御（新規 or 既存の判定）

| 条件                     | 判定   | 用途例                    |
|--------------------------|--------|---------------------------|
| `phaseQuery.length == 0` | 新規   | 初期3件フェーズの自動作成 |
| `phaseQuery.length > 0`  | 既存   | 既存フェーズの表示        |

- 現時点では `isNew` のようなフラグは使わず、**phase クエリ結果の有無で判断**。

---

## 3. phaseBox の制御方針

- `phaseBox` のパラメータとして、各フェーズの Document Reference を渡す設計。
- `endDate` は null チェックが必要（条件分岐による切り替え）。

| 状況            | 表示内容                     |
|-----------------|------------------------------|
| `endDate == null` | 新規フェーズ → プレースホルダやPicker初期値 |
| `endDate != null` | 既存フェーズ → Firestore の日付を表示 |

- 🔧 現在の `phaseBox` 表示エラー（赤エラー）は、**この条件分岐未処理の可能性が高い**。

---

## 4. 今後の実装タスク（ToDo）

- [ ] `AnkenList`（仮）画面の作成と、`PhasePage` への遷移処理実装
- [ ] `phaseQuery.length` を用いた新規／既存の分岐処理
- [ ] `endDate` における条件分岐処理（null or Firestore 値）
- [ ] `phaseBox` を既存データでも安定表示できるように調整

---
### 11 PageStateの構成（追加）

以下の3つのPageStateを削除処理対応のために新たに追加：

## 11.1 deletedOrder（int）

目的: ユーザーが削除ボタンを押したときに、そのPhaseBoxのorderを一時的に記録。

使用箇所:

onDeleteアクションの冒頭でセットされる。

Loop処理や更新処理で、対象のフェーズが削除対象かを判別するために利用。

## 11.2 phasesToDeleteCheck（List）

目的: 該当ankenIdに紐づくすべてのフェーズをQueryし、削除前に何件あるかを確認するために一時保持。

使用箇所:

削除処理の冒頭でFirestoreからフェーズを取得し、分岐条件（最低1件保持するため）に使用。

## 11.3 updatedPhaseList（List）

目的: 削除後にorderを詰め直すための対象フェーズ群を一時的に保持。

使用箇所:

deletedOrderより大きいorder値のフェーズを対象とし、Loop処理内でorder更新に使用。

### 12. PhaseBoxの削除処理

## 12.1.1 削除条件の判定
フェーズ数の確認: 対象案件 (ankenId) に紐づくフェーズ (phases コレクション) を order 昇順でクエリし、phasesToDeleteCheck に格納。

削除可否の判定: phasesToDeleteCheck.length > 1 の場合、削除可能とする。1件以下の場合、削除不可とし、アラートを表示。

## 12.1.2 削除処理の実行
対象フェーズの削除: 条件を満たす場合、選択されたフェーズドキュメント (phaseDoc) を削除。

## 12.1.3 フェーズ一覧の再取得
再クエリ: 削除後、対象案件のフェーズ一覧を再取得し、updatedPhaseList に格納。

## 12.1.4 フェーズ順序の再計算
ループ処理: updatedPhaseList をループし、各フェーズドキュメントの order を Loop Index に更新。

表示用順序の設定: 同時に、orderDisplay フィールドを Loop Index + 1 に設定し、ユーザーに1始まりの順序を表示。

## 12.1.5 アラート表示（削除不可時）
アラート内容:

タイトル: 削除できません

メッセージ: フェーズは最低1件必要です

アイコン: ❌ または ⚠️

## 12.2 表示用順序 (orderDisplay) の管理
保存場所: phases コレクション内の各フェーズドキュメントに orderDisplay フィールドを追加。

更新タイミング: フェーズの追加、削除、開始日程変更、終了日程変更時に orderDisplay を再計算し、Firestoreに保存。

## 12.3 UIコンポーネントへの表示値の受け渡し
パラメータ追加: phaseBox コンポーネントに orderDisplayName パラメータを追加。

表示内容: orderDisplay の値を orderDisplayName として渡し、ユーザーに表示。


## 備考

削除による表示エラーを防ぐため、フェーズは最低1件が必須となっている。
この方針（最低1件残す）により、フェーズ追加の自動処理（空なら追加など）は不要。

### 13. フェーズ（Phase）追加機能のアクションフロー詳細
## 13.1. 概要
このドキュメントは、案件詳細画面で新しいフェーズを追加する際の「＋」ボタンのアクションフローについて詳細を記述します。
この機能は、既存のフェーズの間に新しいフェーズを挿入し、挿入位置以降の既存フェーズの表示順序（orderおよびorderDisplayフィールド）を自動的に更新するものです。

## 13.2. 対象機能
機能: 案件詳細画面におけるフェーズの追加
トリガー: ListView内の各フェーズ（phaseBox）に配置された「＋」ボタンのタップ
## 13.3. データ構造（Firestore phases コレクション）
phasesコレクションには、以下の主要なフィールドが含まれることを想定します。

ankenId (String): 案件を識別するID。
order (Integer): フェーズの実際の順序を示す数値。昇順でソートされる。
orderDisplay (Integer): UI上での表示順序を示す数値（通常 order + 1）。
phaseName (String): フェーズの名称。
startDate (DateTime): フェーズの開始日。
endDate (DateTime): フェーズの終了日。
status (String): フェーズのステータス（例: "未着手", "進行中", "完了"）。
createdAt (DateTime): ドキュメント作成日時。
updatedAt (DateTime): ドキュメント最終更新日時。
## 13.4. アクションフロー詳細
「＋」ボタンのアクションフローは、以下の3つの主要なステップで構成されます。

# 設定の前提
ListView: phasesコレクションをorderフィールドで昇順にソートして表示。
「＋」ボタン: ListViewの各アイテム（phaseBox）内に配置。
phasesItem変数: 各phaseBoxには、そのフェーズのドキュメントデータがphasesItemという変数で利用可能。
案件ID: Page Parametersとして現在の案件IDが利用可能。

# アクション1: 既存のフェーズの順序を繰り下げる
挿入位置以降の既存のフェーズのorderとorderDisplayを1ずつ増やし、新しいフェーズのためのスペースを確保します。

アクション名: Query Collection (例: Get Phases to Shift)
Action Type: Firestore → Query Collection
Collection: phases
Filters:
Filter 1: ankenId ==
Value Source: Page Parameters → 案件ID
Filter 2: order >
Value Source: Set from Variable → phasesItem → order
（「＋」ボタンがあるフェーズのorder値よりも大きい全てのフェーズを対象とします）
Order By:
order Descending (降順)
重要: 大きいorder値から順に更新することで、データの上書きや重複を回避し、順序の整合性を保ちます。
Output Variable Name: phasesToShift (例: Document (List))
このクエリ結果（更新対象のフェーズのリスト）がこの変数に格納されます。

# アクション2: 取得したフェーズをループしてorderとorderDisplayを更新
上記で取得したphasesToShiftリストの各フェーズに対して、orderとorderDisplayをIncrementします。

アクション名: Loop

Action Type: Control Flow → Loop

Loop Type: For Each

Loop Object: Query Collectionアクションの出力変数である phasesToShift を選択します。

Loop Item Name: currentPhaseDoc (例: Document)

ループの各イテレーションで処理される個々のフェーズドキュメントを表す変数名。
Loop Actions (「Loop」アクションの内部):

アクション名: Update Document
Action Type: Firestore → Update Document
Document Reference:
「Select Reference to Update」の入力欄をクリック。
Source: 「Loop Item」セクションの中から、currentPhaseDoc（あなたのループアイテム名）を選択します。
currentPhaseDocの下にある「Document Reference」または「Reference」を選択します。（例: currentPhaseDoc.reference）
Set Fields:
Field: order
Value Source: Increment
Increment Amount: 1
Field: orderDisplay
Value Source: Increment
Increment Amount: 1
Field: updatedAt
Value Source: Global Properties → Current Time

# アクション3: 新しいフェーズドキュメントを作成する
既存フェーズの繰り下げが完了した後、新しいフェーズを適切な順序で追加します。

アクション名: Create Document
Action Type: Firestore → Create Document
Collection: phases
Set Fields:
Field: ankenId
Value Source: Page Parameters → 案件ID
Field: order
Value Source: Inline Function
引数（Argument）: currentOrder (Type: Integer) として phasesItem → order を設定。
Expression: currentOrder + 1
Field: orderDisplay
Value Source: Inline Function
引数（Argument）: currentOrder (Type: Integer) として phasesItem → order を設定。
Expression: currentOrder + 2
Field: endDate
Value Source: Inline Function
引数（Argument）: previousPhaseEndDate (Type: DateTime, Nullable: True) として phasesItem → endDate を設定。
Expression: previousPhaseEndDate != null ? previousPhaseEndDate.add(Duration(days: 1)) : DateTime.now()
（null回避のため、previousPhaseEndDateがnullの場合は現在時刻をデフォルトとしています。必要に応じて調整してください。）
Field: updatedAt
Value Source: Global Properties → Current Time

## 13.5. 動作検証結果と期待される挙動
上記のフローにより、以下の挙動が期待されます。

ユーザーが「＋」ボタンをタップすると、
Query Collectionが実行され、現在の案件において、タップされた「＋」ボタンのフェーズよりorder値が大きい全てのフェーズドキュメントが、orderの降順で取得されます。
Loopアクションが、取得した各フェーズドキュメントに対してUpdate Documentを実行します。
各フェーズのorderとorderDisplayが、安全にIncrementされます。
ループ処理が完了した後、Create Documentアクションが実行されます。
新しいフェーズドキュメントが、Inline Functionによって計算された適切なorder（挿入元フェーズのorder + 1）とorderDisplay（order + 2）で作成されます。
endDateは、Inline Functionを通じて、挿入元フェーズのendDateに1日加算された日付、またはnull回避のデフォルト値が設定されます。
全ての書き込み操作がアトミックにFirestoreに適用されます。
ListViewはFirestoreの変更を検知し、自動的にUIを更新し、新しいフェーズが正しい位置に表示されます。


### 備考

- `PhasePage` では「常に `ankenDoc` をパラメータとして渡す」方針に統一すると、今後の拡張や分岐がしやすくなる。
- 編集フラグや表示モードなどを扱う場合は、追加のパラメータ設計も検討する。


