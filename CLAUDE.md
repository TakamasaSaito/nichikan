# にちかん — 開発仕様書 & CLAUDE.md

> Claude Code でゼロから再実装するための完全仕様書。
> 現行 `index.html`（1634行）を解析して生成。

---

## 0. プロジェクト概要

| 項目 | 内容 |
|------|------|
| アプリ名 | にちかん（日程調整アプリ） |
| キャッチコピー | 日程調整を、かしこく。 |
| ターゲット環境 | **iOS Safari（iPhone）最優先**、Android Chrome も動けばよい |
| 出力形式 | **単一ファイル HTML**（`index.html`）。CSS・JS はインライン |
| バックエンド | Firebase Realtime Database（REST SDK） |
| ホスティング | GitHub Pages（`takamasasaito.github.io/nichikan/`） |

---

## 1. iOS Safari 開発制約（必須・絶対守ること）

```
❌ 禁止
- template literals（バッククォート `...`）
- async / await
- <script type="module">（Firebaseのimport以外）
- confirm() / alert()
- 8桁16進カラー（#rrggbbaa）
- querySelector() 内でシングルクォートをネスト（Safari サイレントクラッシュ）
- 同一関数の重複定義

✅ 使用すること
- var（letも可だが varが安全）
- 文字列連結（+ 演算子）
- Promise チェーン（.then/.catch）
- rgba() 関数
- カスタムモーダル（confirm/alert の代替）
- document.getElementById() 経由のイベント登録
```

### JS変更後の検証コマンド（必須）
```bash
# scriptタグ内のJSを抽出して構文チェック
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const scripts = html.match(/<script[^>]*>([\s\S]*?)<\/script>/g);
const last = scripts[scripts.length-1].replace(/<\/?script[^>]*>/g,'');
try { new Function(last); console.log('OK'); } catch(e) { console.log('ERROR:', e.message); }
"
```

---

## 2. デザインシステム

### フォント（Google Fonts CDN）
```
DM Serif Display  → ロゴ・大見出し・数値強調
JetBrains Mono    → ID表示・ラベル・バッジ・数値
Noto Sans JP      → 本文・ボタン・フォーム
```

### カラートークン（CSS変数）

#### ライトモード
```css
--bg0: #F0EEE9;   /* ページ背景 */
--bg1: #F7F6F2;   /* ヘッダー */
--bg2: #FFFFFF;   /* カード */
--bg3: #F4F2EE;   /* 入力フィールド */
--bg4: #FFFFFF;   /* トースト */
--border: rgba(0,0,0,0.08);
--border-gold: rgba(180,140,40,0.2);
--text: #0F0E0C;
--text2: #6B6560;
--text3: #9E9890;
--gold: #b8922e;
--gold2: #d4aa50;
--ok: #1a9e75;         /* ○ 緑 */
--maybe: #c47a0a;      /* △ 橙 */
--no: #c93044;         /* × 赤 */
--ok-bg: rgba(26,158,117,.08);
--maybe-bg: rgba(196,122,10,.08);
--no-bg: rgba(201,48,68,.08);
```

#### ダークモード（@media prefers-color-scheme: dark）
```css
--bg0: #080a0f;
--bg1: #0d1017;
--bg2: #141820;
--bg3: #1c2130;
--bg4: #232b3a;
--border: rgba(255,255,255,0.06);
--border-gold: rgba(201,168,76,0.18);
--text: #eceae4;
--text2: #8a8070;
--text3: #5a5248;
--gold: #c9a84c;
--gold2: #e8c96a;
--ok: #2dd4a0;
--maybe: #f5a623;
--no: #f05c6e;
--ok-bg: rgba(45,212,160,.1);
--maybe-bg: rgba(245,166,35,.1);
--no-bg: rgba(240,92,110,.1);
--shadow-card: 0 1px 0 rgba(201,168,76,.07), 0 4px 24px rgba(0,0,0,.45);
```

### その他トークン
```css
--shadow-card: 0 1px 3px rgba(0,0,0,0.06), 0 4px 16px rgba(0,0,0,0.05);
--radius: 14px;
--radius-sm: 8px;
```

---

## 3. 画面構成・フロー

```
[オンボーディング]（初回のみ、全画面オーバーレイ）
  ↓ 完了
[ホーム]（役割選択）
  ├── 主催者 →
  │     ├── [主催者メニュー]
  │     │     ├── [作成] → [作成完了]
  │     │     ├── [管理入口] → [管理画面]
  │     │     ├── [集計入口] → [集計]
  │     │     └── [イベント一覧] → [集計]
  └── 参加者 →
        ├── [参加者メニュー]
        │     ├── [参加（ID入力）] → [回答] → ホームへ
        │     └── [集計入口] → [集計]
```

### 画面ID一覧（`id="screen-{name}"`）

| screen-name | 説明 |
|-------------|------|
| `home` | ロール選択（主催者/参加者） |
| `host` | 主催者メニュー |
| `guest` | 参加者メニュー |
| `create` | イベント作成フォーム |
| `created` | 作成完了・ID表示 |
| `join` | 参加（ID入力・名前入力） |
| `answer` | 回答（○△×） |
| `result-entry` | 集計入口（ID入力） |
| `result` | 集計画面（タブ切替） |
| `event-list` | 作成済みイベント一覧 |
| `admin-entry` | 管理入口（ID入力） |
| `admin` | 管理画面（優先度設定） |

---

## 4. 機能仕様

### 4-1. オンボーディング（初回起動）

- 全画面オーバーレイ、4スライド構成
- スライド0：スプラッシュ（ロゴアニメーション）→ 2秒後自動遷移
- スライド1：「イベントを作成する」Step説明
- スライド2：「IDを共有して回答してもらう」Step説明
- スライド3：「最適日を自動で提案」→「はじめる」ボタン
- スワイプ対応（touchstart/touchend、50px閾値）
- スキップボタン（スライド1・2に表示）
- 完了後 `localStorage.setItem('nichikan_ob_done_prod', '1')` で記録
- 現行実装では毎回表示（prod環境では毎回スプラッシュ）

### 4-2. ナビゲーション

```javascript
// 画面切り替え
function go(name) {
  // 全 .screen の active を外す
  // #screen-{name} に active を付ける
  // window.scrollTo(0, 0)
}
```

- `[data-screen="xxx"]` 属性を持つ要素クリックで `go(xxx)` 呼び出し
- `event-list` 画面への遷移時は `renderEventList()` も呼ぶ

### 4-3. トースト通知

```javascript
function toast(msg) {
  // #toast の textContent を設定
  // .show クラスを付けて 2500ms 後に外す
  // position: fixed, bottom: 30px, transform でスライドイン
}
```

### 4-4. イベント作成

**入力フィールド**
- イベント名（必須、最大40文字）
- メモ（任意）
- 回答締め切り日（任意、type="date"）
- 参加者名リスト（任意、最大20文字/人、タグUI）
- 候補日（カレンダー選択）

**カレンダー**
- 月ナビ（前月/次月ボタン）
- 日曜：赤、土曜：青（#4fa3e8）
- 過去日：グレー・クリック不可
- 選択済み：ゴールド背景
- 選択日はタグとして下部に表示（×で削除）

**作成処理**
```javascript
// IDは6文字（ABCDEFGHJKLMNPQRSTUVWXYZ23456789 から重複なし）
function genId() {
  const c = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
  // 紛らわしい文字（I, O, 0, 1）は除外
}

// 保存データ構造
{
  id: "XXXXXX",         // 6文字
  name: "イベント名",
  memo: "メモ",
  dates: ["2025-07-01", ...],  // YYYY-MM-DD
  members: ["田中", "佐藤", ...],
  deadline: "2025-06-30",  // or ""
  createdAt: Date.now(),
  priorities: {},    // { "田中": "required" | "important" | "normal" }
  answers: {}        // { "田中": { name, answers: {date: "ok"|"maybe"|"no"}, comment, answeredAt } }
}
```

### 4-5. 参加・回答

**ID入力時の動作**
- 6文字になった時点で自動的にイベントを取得
- メンバー登録がある場合：名前チップを表示（回答済みは ✓ マーク）
- メンバーなし：直接テキスト入力
- チップ選択 → `#join-name` に名前をセット

**回答UI**
- 各候補日に対して ○ △ × ボタン
- 選択状態：`.active-ok` / `.active-maybe` / `.active-no` クラス
- 既存回答がある場合は復元表示（`applyExistingAnswer`）
- 回答者名バッジ：新規は「新規回答」緑バッジ、編集は「編集中」橙バッジ
- コメント入力（任意、textarea）
- 送信：全日付回答必須チェック

**回答データのキー**
```javascript
const key = name.replace(/[.#$[\]]/g, '_');  // Firebase のキー制約対応
```

### 4-6. 集計画面

**タブ構成**

| タブID | 内容 | 表示条件 |
|--------|------|---------|
| `table` | 集計表 + ヒートマップ | 全員 |
| `chart` | 棒グラフ2種 | 全員 |
| `best` | 最適日提案 + シェア | 全員 |
| `comments` | コメント一覧 | 全員 |
| `admin-inline` | ⚙️管理（優先度・確定） | 主催者のみ |

**集計表**
- 日付ごとに ○/△/× の人数表示
- ヒートマップバー（緑:○比率 / 橙:△比率 / 赤背景:残り）
- BEST行：最高スコアかつ必須NGなし → ゴールド背景
- スコア列は主催者のみ表示
- スコア行タップ → スコア内訳ポップアップ（主催者のみ）

**スコア計算ロジック**
```
通常(normal): ○=+2点, △=+1点, ×=0点
重要(important): ○=+6点(×3), △=+3点(×3), ×=0
必須(required): ×があった日 → reqFail=true → 最適日から除外
最適日 = reqFailなし かつ score最大の日
```

**グラフ（Chart.js 4.4.0）**
- 棒グラフ1：○人数（最適日はゴールド、通常は薄ゴールド、NG日はグレー）
- 棒グラフ2：○△×積み上げ（緑/橙/赤）

**最適日**
- OPTIMAL DATE として大きく表示
- 主催者：SCORE + ○△×人数
- 参加者：○△×人数のみ（スコア非表示）
- 日程確定ボタン（主催者のみ）
- テキストコピーボタン

**日程確定**
- 主催者のみ表示（⚙️管理タブ or result-tab-best）
- 候補日ボタンをタップ → Firebase保存 → 確定バナー表示
- 取り消しボタンあり

**未回答状況カード**
- メンバー登録がある場合のみ表示
- ステータスバー（登録N人 / 回答済みN人 / 未回答N人）
- メンバーチップ（回答済み：緑 ✓、未回答：赤 ⏳）

**テキストコピー**
```
【イベント名】日程調整結果
メモ
回答者：N人

M/D（曜）　○N △N ×N  ⭐おすすめ
...
おすすめ：M/D（曜）（○N人）
✅ 確定日：M/D（曜）
```

### 4-7. 優先度設定

**設定値**

| 値 | 表示 | 効果 |
|----|------|------|
| `required` | 🔴 必須 | ×の日を候補から除外 |
| `important` | 🟡 重要 | スコア3倍 |
| `normal` | ⚪ 一般 | 通常カウント |

- **回答前のメンバーも設定可能**（今回修正済み）
- 未回答者は「未回答」バッジ付きで表示
- 主催者のみ参照・設定可能（参加者には表示されない）
- 管理画面（`screen-admin`）と集計画面内⚙️タブの両方に存在

**保存対象**
- 回答済み + 未回答メンバー全員のpriorityを保存
- キー：`name.replace(/[.#$[\]]/g, '_')`

### 4-8. 管理画面

- IDを入力して管理画面へ
- 回答状況（メンバーチップ）
- 優先度設定（セレクトボックス）
- 「優先度を保存する」ボタン
- イベントリセットボタン（`answers: {}` にして保存）

### 4-9. イベント一覧

- `localStorage` 内の全イベントを作成日時降順表示
- 各アイテム：イベント名、候補日件数、回答数、ID
- タップ → 集計画面へ
- 「全データを削除」ボタン

### 4-10. スコア内訳ポップアップ

- 集計表のスコアをタップで下からスライドインするシート
- 各回答者の名前・優先度バッジ・回答（○△×）・得点を表示
- 合計得点と計算式を表示
- 必須メンバーが×の場合は警告表示
- バックドロップタップで閉じる

### 4-11. ウォークスルー

- 全画面オーバーレイ（4ステップ）
- 各ステップで対象要素をゴールドのoutlineでハイライト（パルスアニメーション）
- 「次へ」「スキップ」ボタン
- 完了でホームに戻る

---

## 5. Firebase 連携

```javascript
// Firebase SDK（ESM import、<script type="module"> で読み込み）
import('https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js')
import('https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js')

// 設定値
const firebaseConfig = {
  apiKey: "AIzaSyBi8T1eiSrFt4Cvqg8Cc1P6nmob-hpOSrs",
  authDomain: "nichikan-7788c.firebaseapp.com",
  databaseURL: "https://nichikan-7788c-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId: "nichikan-7788c",
  storageBucket: "nichikan-7788c.firebasestorage.app",
  messagingSenderId: "332427860675",
  appId: "1:332427860675:web:d6ba6e6226ca9c4246e076"
};

// パス: events/{id}

// 取得
async function getEvent(id) {
  const snap = await get(ref(db, 'events/' + id));
  return snap.exists() ? snap.val() : null;
}

// 保存
async function saveEvent(data) {
  await set(ref(db, 'events/' + data.id), data);
}
```

**注意**
- Firebase はページロードをブロックしないよう `Promise.all([import(...)])` で非同期ロード
- `_db` が null の間に操作しようとしたら `toast('通信準備中…')` を表示
- <script type="module"> は Firebase import 専用。通常ロジックは通常の `<script>` に書く

---

## 6. localStorage（開発・デモ用途）

現行コードには `loadDB()` / `saveDB()` があるが、**実際のデータはFirebaseに保存**。
localStorageはイベント一覧表示用のキャッシュとして使用。

```javascript
function loadDB() {
  try { return JSON.parse(localStorage.getItem('nichikan_db') || '{}'); } catch(e) { return {}; }
}
function saveDB(db) {
  localStorage.setItem('nichikan_db', JSON.stringify(db));
}
```

---

## 7. URLパラメータ対応

```javascript
// ?id=XXXXXX で直接参加画面に遷移
const urlId = new URLSearchParams(location.search).get('id');
if (urlId) {
  $('join-id').value = urlId.toUpperCase();
  state.role = 'guest';
  go('guest');
  setTimeout(function() { joinEvent(); }, 100);
}
```

---

## 8. state オブジェクト

```javascript
var state = {
  eventId: null,        // 現在操作中のイベントID
  eventData: null,      // 現在操作中のイベントデータ
  joinName: null,       // 参加者名
  answerChoices: {},    // { "2025-07-01": "ok" | "maybe" | "no" }
  calYear: new Date().getFullYear(),
  calMonth: new Date().getMonth(),
  selectedDates: [],    // 作成時の選択候補日
  members: [],          // 作成時の参加者リスト
  chart: null,          // Chart.js インスタンス（○棒グラフ）
  chartRatio: null,     // Chart.js インスタンス（積み上げ棒）
  resultSummary: null,  // テキストコピー用キャッシュ
  role: null            // 'host' | 'guest'
};
```

---

## 9. 日付フォーマット関数

```javascript
function fmt(s) {
  // "2025-07-01" → "7/1（火）"
  var parts = s.split('-').map(Number);
  var y = parts[0], m = parts[1], d = parts[2];
  var dw = ['日', '月', '火', '水', '木', '金', '土'];
  return m + '/' + d + '（' + dw[new Date(y, m-1, d).getDay()] + '）';
}
```

---

## 10. 外部依存ライブラリ

| ライブラリ | バージョン | 用途 | 読み込み方法 |
|-----------|-----------|------|------------|
| Chart.js | 4.4.0 | グラフ描画 | cdnjs CDN（`<script src>`） |
| Firebase App | 10.12.0 | Firebase初期化 | ESM dynamic import |
| Firebase Database | 10.12.0 | RTDB操作 | ESM dynamic import |
| Google Fonts | - | フォント | `<link rel="preconnect/stylesheet">` |

---

## 11. デモデータ

デバッグ用に2イベントのデモデータをlocalStorageに投入する関数 `loadDemoData()` が存在。
- DEMO01：チーム納会（全員回答済み、優先度設定あり）
- DEMO02：プロジェクトキックオフ（一部未回答）

---

## 12. Claude Code への実装指示

### ファイル構成（単一ファイル）
```
index.html  ← これ1ファイルのみ
```

### 実装順序（推奨）

1. **HTML骨格** — 全画面の`<div id="screen-xxx">`をHTMLで定義、CSS変数・フォント読み込み
2. **ナビゲーション** — `go()`, `toast()`, back-btn, data-screen
3. **カレンダー** — `renderCal()`, 日選択, member タグ
4. **イベント作成** — `createEvent()`, `genId()`, Firebase saveEvent
5. **参加・回答** — `joinEvent()`, `buildAnswerScreen()`, `submitAnswer()`
6. **集計** — `renderResult()`, スコア計算, Chart.js グラフ
7. **優先度設定** — `renderAdmin()`, `saveAdmin()`, `renderInlineAdmin()`
8. **オンボーディング** — スプラッシュ4スライド、スワイプ
9. **ウォークスルー** — 4ステップ、ハイライトアニメーション
10. **スコアポップアップ** — 内訳シート
11. **仕上げ** — URLパラメータ対応, テキストコピー, デモデータ

### 各ステップで守ること
- JS実装後は必ず `node --check` で構文確認
- イベントリスナーは `getElementById` + `.addEventListener` で登録（inline onclickは使わない）
- テンプレートリテラルを使わずに文字列連結で生成
- 関数の重複定義がないかチェック
- `<script type="module">` はFirebase import のみに限定。通常の `<script>` と明確に分離
