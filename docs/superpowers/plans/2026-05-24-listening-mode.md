# リスニングモード実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 音声を聞いてから英文を答え合わせするリスニングモードを既存の英語学習アプリに追加する

**Architecture:** 単一ファイル（`index.html`）の React + Babel Standalone 構成。既存の `MODES` オブジェクトに5つ目のモードを追加し、問題画面のレンダリング分岐と自動再生 `useEffect` を1つ追加する。既存4モードの挙動は一切変更しない。

**Tech Stack:** React 18 (UMD), Tailwind CSS (CDN), Web Speech API (`window.speechSynthesis`)

**Spec:** `docs/superpowers/specs/2026-05-24-listening-mode-design.md`

**注記:** このプロジェクトにはテストフレームワークが存在しない。spec で「テストはブラウザ手動検証」と定義済み。よって本計画は TDD ではなく、コード変更 → ブラウザ目視確認 → コミット の流れで進める。

---

## ファイル構成

- 修正: `index.html`（唯一の変更対象）
- 修正箇所3つ:
  1. `MODES` 定義（355-360行付近）
  2. 問題画面のレンダリング分岐（616-685行付近）
  3. 自動再生 `useEffect`（557-561行の直後に新規追加）

---

### Task 1: MODES に listening モードと autoPlay フィールドを追加

**Files:**
- Modify: `index.html:355-360`

- [ ] **Step 1: MODES オブジェクトを書き換える**

`index.html` の355-360行（`const MODES = { ... };`）を以下に置換:

```js
const MODES = {
  en2ja:     { label: '英 → 日',  q: 'en',    a: 'ja', voice: false, autoPlay: false },
  ja2en:     { label: '日 → 英',  q: 'ja',    a: 'en', voice: false, autoPlay: false },
  en2voice:  { label: '英 → 🔊', q: 'en',    a: 'en', voice: true,  autoPlay: false },
  ja2voice:  { label: '日 → 🔊', q: 'ja',    a: 'en', voice: true,  autoPlay: false },
  listening: { label: '🔊 → 英', q: 'audio', a: 'en', voice: true,  autoPlay: true  },
};
```

- [ ] **Step 2: ブラウザで開いてタブが5つ表示されることを確認**

```bash
open index.html
```

期待: 上部のタブ列に「英 → 日 / 日 → 英 / 英 → 🔊 / 日 → 🔊 / 🔊 → 英」の5つが表示される。
（この時点ではリスニングタブを押すと「question is undefined」的な表示崩れが起きてもよい。次タスクで直す）

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: add listening mode entry and autoPlay field to MODES"
```

---

### Task 2: 問題画面に音声専用レンダリング分岐を追加

**Files:**
- Modify: `index.html:616-685`

- [ ] **Step 1: questionText の算出ロジックを修正**

`index.html:616` の現在の行:
```js
const questionText = mode.q === 'en' ? current.en : current.ja;
```

を以下に置換:
```js
const isAudioQuestion = mode.q === 'audio';
const questionText = isAudioQuestion ? '' : (mode.q === 'en' ? current.en : current.ja);
```

- [ ] **Step 2: 問題テキストの表示ブロックを条件分岐に変える**

`index.html:680-685` あたりの現在の `<div className="text-center flex flex-col justify-center" ...>` 内の `<div className={qIsEn ? 'font-serif-en' : 'font-serif-ja'}>...</div>` ブロックを以下に置換:

```jsx
<div className="text-center flex flex-col justify-center" style={{ minHeight: 200 }}>
  {isAudioQuestion ? (
    <div className="flex flex-col items-center gap-6">
      <Volume2 size={96} className="ink-soft" />
      <button
        onClick={() => speak(current.en)}
        className="btn-primary rounded-full px-6 py-3 flex items-center gap-2 font-serif-ja text-sm"
      >
        <Volume2 size={18} />
        REPLAY
      </button>
    </div>
  ) : (
    <div className={qIsEn ? 'font-serif-en' : 'font-serif-ja'}>
      <p className="text-2xl md:text-3xl leading-snug ink">
        {questionText}
      </p>
    </div>
  )}
```

**注意:** `{showAnswer && (...)}` ブロック（687行以降）は触らない。閉じタグの整合性に注意 — 上記置換は外側の `<div className="text-center ...">` 開始タグから、内側の `</div>` までを置換し、`{showAnswer && (...)}` ブロックは続けてそのまま残す。

- [ ] **Step 3: ブラウザで動作確認**

```bash
open index.html
```

期待: 「🔊 → 英」タブを押すと、画面中央に大きなスピーカーアイコンと「REPLAY」ボタンが表示される。英文・日本語は非表示。REPLAY を押すと音声が流れる。ANSWER ボタンを押すと英文+「もう一度聞く」ボタンが表示される。

- [ ] **Step 4: 既存4モードが壊れていないことを確認**

「英→日」「日→英」「英→🔊」「日→🔊」を順に押し、それぞれ従来通り問題テキストが表示されることを目視確認。

- [ ] **Step 5: コミット**

```bash
git add index.html
git commit -m "feat: render speaker icon and REPLAY button for audio question"
```

---

### Task 3: 問題切替時の自動再生 useEffect を追加

**Files:**
- Modify: `index.html:557-561`

- [ ] **Step 1: 既存の useEffect の直後に新しい useEffect を追加**

`index.html:557-561` の現在の useEffect:
```js
useEffect(() => {
  if (showAnswer && mode.voice && current) {
    speak(current.en);
  }
}, [showAnswer, modeId, pos]);
```

の**直後**（562行目あたり）に以下を追加:
```js

useEffect(() => {
  if (!showAnswer && mode.autoPlay && current) {
    speak(current.en);
  }
}, [pos, modeId, showAnswer]);
```

- [ ] **Step 2: ブラウザで自動再生を確認**

```bash
open index.html
```

期待動作:
1. 「🔊 → 英」タブを押す → 自動で英語音声が1回流れる
2. Next ボタン（→）を押す → 新しい問題の音声が自動で流れる
3. Prev ボタン（←）を押す → 前の問題の音声が自動で流れる
4. ANSWER 押下後に Next → 新しい問題の音声が自動で流れる（showAnswer が false に戻り autoPlay が発火）
5. REPLAY ボタンは何度でも押下可能で毎回再生される

**他モードでの確認**: 「英→日」「日→英」タブを押しても自動再生されないこと（既存挙動が壊れていない）。

- [ ] **Step 3: コミット**

```bash
git add index.html
git commit -m "feat: auto-play audio on question change in listening mode"
```

---

### Task 4: 全体動作確認（最終手動検証）

**Files:** なし（検証のみ）

- [ ] **Step 1: spec の検証項目7つを順に確認**

`open index.html` でブラウザを開き、以下を順に検証:

1. **リスニングタブを選択** → 英語音声が自動で1回流れる
2. **問題画面に大きな🔊アイコンとREPLAYボタンのみが表示** される（英文・日本語は非表示、文字数ヒント等もなし）
3. **REPLAYボタンを複数回押す** → 毎回再生される
4. **ANSWERボタン** → 英文が表示される、音声ボタン（「もう一度聞く」）も表示される
5. **Next/Prev で問題切替** → 新しい問題が自動再生される
6. **既存4タブ**（英→日 / 日→英 / 英→🔊 / 日→🔊）の挙動が**一切変わっていない**
7. **Shuffle/Reset** が動作する（リスニングモード中に Shuffle/Reset しても問題なく動く）

- [ ] **Step 2: キーボードショートカット動作確認**

リスニングモード中に以下を確認:
- `→` キー: 次の問題に進み、自動再生される
- `←` キー: 前の問題に戻り、自動再生される
- `Space` または `Enter`: 1回目は答え表示、2回目以降は「もう一度聞く」と同じ動作（既存ロジックそのまま）

期待: いずれも `mode.voice === true` のため既存ロジックで正しく動作する。

- [ ] **Step 3: 検証が全項目パスしたら追加コミット不要**

すべて期待通りなら本タスクは完了。失敗があれば該当タスクに戻って修正する。

---

## 自己レビュー結果

- **Spec coverage:** spec の全節（アーキテクチャ、データ構造、画面仕様、自動再生ロジック、エラーハンドリング、テスト）をTask 1-4でカバー済み
- **Placeholder scan:** TBD/TODO/省略表現なし
- **Type consistency:** `isAudioQuestion`, `mode.autoPlay`, `mode.q === 'audio'` の使用が全タスクで一貫
- **Code completeness:** すべてのコード変更ステップに完全なコードブロックを記載
