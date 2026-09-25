# Material Design UI 改善実装ガイド

## 1. この文書の目的

Focus Flow のUIを、Material Design 3の考え方に沿った実装へ段階的に修正するための作業指示書です。

対象はUIの見た目だけではなく、次の項目です。

- Materialコンポーネントの使い分け
- 色・タイポグラフィ・余白・形状・標高のトークン化
- 48px以上の操作領域
- ホバー・フォーカス・押下・無効状態
- キーボード操作とスクリーンリーダー対応
- ライト／ダークテーマとレスポンシブ表示

機能仕様、タイマー計算、localStorage、通知音、通知機能は変更しないことを前提にします。

## 2. 現状の結論

現在の実装は、Material Components WebのクラスとMaterial Iconsを使っています。しかし、画面の大部分は `styles.css` の独自CSSで構成されています。

したがって、現在の評価は次のとおりです。

> Material Designを参考にした独自UI。Material Design 3のUIルールに厳密準拠したUIではない。

### 2.1 既にできていること

- Material Iconsを読み込んでいる
- MDCのButton、Card、Text Field、Slider、Icon Button、Rippleを使用している
- ライト／ダークテーマを切り替えられる
- 960px、650pxでレスポンシブレイアウトを切り替えている
- `:focus-visible`、disabled状態、`prefers-reduced-motion`を実装している
- ボタンや入力項目に多くの日本語ラベル、`aria-label`を設定している

### 2.2 優先して直す問題

| 優先度 | 問題 | 主な箇所 | 対応方針 |
|---|---|---|---|
| P0 | 操作領域が48px未満 | `styles.css:120, 206, 208, 210, 217, 234, 244` | 視覚上のアイコンサイズと操作領域を分離し、全操作対象を48px以上にする |
| P0 | タイマーの状態が支援技術に伝わらない | `index.html:55-58`, `app.js:278-284` | timer/progressの意味、値、区間変更をARIAで表現する |
| P0 | テーマ切替の状態が伝わらない | `index.html:41-44`, `app.js:90-93` | `aria-pressed`を追加する |
| P1 | 独自トークンとMDCトークンが混在 | `styles.css:3-38` | Material 3の色・文字・形状・余白トークンに統一する |
| P1 | 余白・サイズに任意値が多い | `styles.css:94-254` | 4px基準のスペーストークンへ置き換える |
| P1 | 独自コンポーネントがMaterialコンポーネント外 | `index.html:146-199`, `index.html:224` | Select、Snackbar、Progressの扱いを統一する |
| P1 | Material 2系MDCとMaterial 3表現が混在 | `index.html:24, 27` | まず依存バージョンを固定し、別作業でMaterial Webへの移行を検討する |
| P2 | タイポグラフィが個別指定中心 | `styles.css:139-163`, `styles.css:179-193` | M3のタイプスケールに割り当てる |
| P2 | 影・角丸が独自値 | `styles.css:133-159`, `styles.css:255` | 標高・形状トークンに整理する |

## 3. 実装方針

### 3.1 Materialの対象範囲を決める

今回の修正では、タイマーの円形表示やブランドマークのようなプロダクト固有の表現は残します。ただし、操作部品はMaterialの標準パターンに寄せます。

- 残してよい独自表現：タイマーの円形進捗、ブランドマーク、アプリ固有の装飾
- Material化する対象：ボタン、アイコンボタン、入力欄、スライダー、リスト、Select、Snackbar、カード、状態表示
- 独自表現にも必要なもの：色コントラスト、フォーカス表示、キーボード操作、ARIA、reduced motion

### 3.2 依存ライブラリの扱い

現状は `https://unpkg.com/material-components-web@latest/` を利用しています。`@latest` は再現性がなく、MDC Webの公式リポジトリもアーカイブ済みです。

短期修正では既存MDCを維持しても構いませんが、次を必須とします。

1. CDNのバージョンを固定する
2. READMEとこの文書で、旧MDCを使用していることを明記する
3. 「現行Material 3準拠」を正式な要件にする場合は、`@material/web`への移行を別タスクとして実施する

依存ライブラリの移行とUI修正を同じ変更に混ぜる場合は、機能回帰が増えるため、先にP0・P1のアクセシビリティ修正を完了させます。

## 4. 具体的な修正手順

### Step 1：デザイントークンを定義する

`styles.css`の先頭にある個別値を、次のような役割別トークンへ整理します。色の具体値はMaterial Theme BuilderまたはMaterial Color Utilitiesで生成し、ライト／ダークの両方でコントラストを確認します。

```css
:root {
  /* Color roles */
  --md-sys-color-primary: ...;
  --md-sys-color-on-primary: ...;
  --md-sys-color-primary-container: ...;
  --md-sys-color-on-primary-container: ...;
  --md-sys-color-surface: ...;
  --md-sys-color-surface-container: ...;
  --md-sys-color-on-surface: ...;
  --md-sys-color-on-surface-variant: ...;
  --md-sys-color-outline: ...;
  --md-sys-color-outline-variant: ...;
  --md-sys-color-error: ...;
  --md-sys-color-on-error: ...;

  /* Spacing: 4px grid */
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;
  --space-8: 32px;
  --space-12: 48px;

  /* Shape */
  --md-sys-shape-corner-small: 8px;
  --md-sys-shape-corner-medium: 12px;
  --md-sys-shape-corner-large: 16px;
  --md-sys-shape-corner-extra-large: 28px;

  /* Typography */
  --md-sys-typescale-body-medium-size: 14px;
  --md-sys-typescale-body-medium-line-height: 20px;
  --md-sys-typescale-label-large-size: 14px;
  --md-sys-typescale-label-large-line-height: 20px;
}
```

既存の `--primary` などをすぐに全削除する必要はありません。まず既存変数を新トークンへ参照させ、コンポーネントごとに段階的に置換します。

### Step 2：操作領域を48px以上にする

Materialのアイコンは24px程度の視覚サイズでも構いませんが、クリック可能な領域は48px以上にします。MDC Icon Buttonを小さく見せたい場合も、ボタン本体またはラッパーに48pxの領域を残します。

最低限、次のルールを適用します。

```css
.interactive,
.mdc-button,
.mdc-icon-button,
.notification-button,
.quick-options button,
.add-interval,
.text-button {
  min-width: 48px;
  min-height: 48px;
}

.mdc-icon-button .material-icons {
  width: 24px;
  height: 24px;
  font-size: 24px;
}
```

次の既存指定は、見た目を小さくするための指定としては残さず、操作領域を壊さない形へ変更します。

- `.theme-toggle`: 40px → 48px以上
- `.icon-button`: 29px → 48px以上
- `.icon-button.move-up/.move-down`: 31px → 48px以上
- `.add-interval`: 45px → 48px以上
- `.quick-options button`: 39px → 48px以上
- `.button-small`: 41px → 48px以上
- `.notification-button`: 34px → 48px以上

横幅が狭い区間リストでは、アイコンを小さくするのではなく、各ボタンを縦に並べる、または48pxのタップ領域が重ならないレイアウトに変更します。

### Step 3：テーマ切替の状態をアクセシブルにする

`index.html`に初期状態を設定し、`app.js`の`applyTheme()`で状態も更新します。

```html
<button
  class="theme-toggle mdc-icon-button"
  id="themeToggle"
  type="button"
  aria-label="ダークモードに切り替える"
  aria-pressed="false">
```

```js
elements.themeToggle.setAttribute('aria-pressed', String(isDark));
```

アイコンとラベルだけでなく、現在の状態を機械的に取得できるようにすることが目的です。

### Step 4：タイマーを意味のあるUIとして公開する

タイマーの見た目を表す`timer-orb`と、支援技術へ伝えるタイマー情報を分けます。

推奨構成：

```html
<div
  class="timer-orb"
  id="timerOrb"
  role="progressbar"
  aria-label="現在の区間の進捗"
  aria-valuemin="0"
  aria-valuemax="100"
  aria-valuenow="0">
  ...
</div>

<span id="timeDisplay" role="timer" aria-live="off">25:00</span>
<span id="timerAnnouncement" class="sr-only" aria-live="polite"></span>
```

`renderTimer()`で次を更新します。

- `aria-valuenow`: 0〜100の進捗率
- `aria-valuetext`: 「残り25分」などの自然文
- 区間開始、一時停止、再開、完了時だけ`timerAnnouncement`へ通知

毎秒の値をそのまま`aria-live="polite"`で読み上げると過剰通知になるため、状態変化だけを読み上げます。

### Step 5：MaterialコンポーネントでないUIを整理する

次のUIは、Materialコンポーネントに置き換えるか、独自UIとしてアクセシビリティ要件を満たすかを決めます。

#### プリセット選択

対象：`index.html:158`

- Material Selectへ置き換える
- 置き換えない場合は、ネイティブ`select`としてキーボード操作とフォーカス表示を維持し、Materialコンポーネントであると表記しない

#### Toast

対象：`index.html:224`、`styles.css:255-256`

- Material Snackbarへ置き換える
- 独自Toastを残す場合は、`role="status"`、`aria-live="polite"`、十分なコントラスト、キーボードで内容を確認できることを検証する

#### 区間リスト

対象：`app.js:234-248`

- コンテナを`ul`、行を`li`にする
- 行内の入力欄・移動・削除ボタンに一意なラベルを付ける
- `mdc-list-item`の見た目だけを付けて、リストの意味を持たせない状態を避ける

### Step 6：タイポグラフィをタイプスケールへ割り当てる

個別の`font-size`を増やさず、役割ごとのクラスを作ります。

最低限、次の役割を定義します。

- `display`: タイマーの残り時間
- `headline`: カード見出し
- `title`: セクション見出し
- `body`: 説明文・状態文
- `label`: ボタン・入力ラベル・補助情報

日本語フォントとしてNoto Sans JPを使うこと自体は問題ありません。ただし、10px前後の本文を常用せず、読みやすさを確認します。

### Step 7：余白、角丸、影をトークンへ置き換える

次のような任意値を直接書かないようにします。

- `gap: 7px`, `9px`, `11px`, `13px`
- `border-radius: 42px 42px 24px 42px`
- コンポーネントごとに異なる大量の`box-shadow`

タイマーの円形表現など、ブランド上必要な装飾は例外として残して構いません。その場合も、通常のボタン・カード・入力欄には共通の形状と標高を使用します。

標準的なカードと操作部品は、次のように整理します。

- 小さな入力・ボタン：small shape
- 通常カード：medium shape
- 大きな主要カード：large / extra-large shape
- 主要ボタン：filledまたはelevatedのいずれか
- 補助操作：outlinedまたはtext
- 同一領域内で、主要アクションを複数のfilledボタンにしない

### Step 8：状態表現を統一する

すべての操作部品について、以下の状態を確認します。

- default
- hover
- focus-visible
- pressed / active
- disabled
- selected
- error

色の変化だけに依存せず、フォーカスリング、Ripple、背景のstate layer、アイコン、テキストでも状態が分かるようにします。

既存の`transform: translateY()`や`scale()`は、操作対象の位置が大きく動いて誤タップを誘発しないかを確認します。特に小型ボタンを視覚的に浮かせるアニメーションより、Materialのstate layerを優先します。

## 5. 作業順序

### フェーズA：P0の安全性・アクセシビリティ

1. すべての操作対象を48px以上にする
2. テーマ切替へ`aria-pressed`を追加する
3. タイマーにtimer/progressの意味を追加する
4. 区間開始・一時停止・完了を状態通知する
5. キーボードのTab、Enter、Space操作を確認する

### フェーズB：Materialトークンとコンポーネント

1. 色トークンを整理する
2. 余白を4px基準へ置き換える
3. タイポグラフィを役割別クラスへ置き換える
4. 標準カード・ボタン・入力欄の形状と影を統一する
5. Select、Snackbar、Listの扱いを決める

### フェーズC：依存関係と仕上げ

1. `material-components-web@latest`を固定バージョンへ変更する
2. Material Webへの移行要否を決定する
3. ライト／ダークの全状態でコントラストを確認する
4. レスポンシブとreduced motionを確認する
5. READMEの「Material 3 Expressiveを意識した」という表現を、実装状態に合わせて更新する

## 6. 完了条件

### 必須条件

- [ ] クリック・タップ可能な要素の実効領域がすべて48px以上
- [ ] アイコンだけのボタンに日本語の`aria-label`がある
- [ ] テーマ切替に`aria-pressed`がある
- [ ] タイマーの残り時間、進捗、区間状態が支援技術へ伝わる
- [ ] キーボードだけで主要操作を完了できる
- [ ] すべての操作対象に明確な`focus-visible`表示がある
- [ ] disabled、selected、pressed、errorの状態が色だけに依存していない
- [ ] ライト／ダークの主要テキストがWCAG AA相当のコントラストを満たす
- [ ] 320px幅で横スクロールが発生しない
- [ ] `prefers-reduced-motion: reduce`で不要なアニメーションが停止する

### Material 3準拠を名乗る場合の追加条件

- [ ] Material 3の色・タイポグラフィ・形状・標高トークンを使用している
- [ ] 現行のMaterial Webまたは、使用するMDCの対応バージョンを明記している
- [ ] Materialコンポーネントで代替できるUIを独自CSSだけで再実装していない
- [ ] 独自のタイマー装飾は、標準コンポーネントと視覚的・操作的に競合していない
- [ ] 依存ライブラリのバージョンが固定され、再現可能に読み込める

## 7. 手動確認チェックリスト

### デスクトップ

- [ ] 1440px：タイマーカードと設定カードの階層が分かる
- [ ] 1024px：設定カードがタイマーを圧迫しない
- [ ] 768px：1カラムへ自然に切り替わる

### モバイル

- [ ] 650px以下：ボタンが横からはみ出さない
- [ ] 375px：クイック設定の各ボタンを誤タップしない
- [ ] 320px：区間行の入力欄と操作ボタンが重ならない

### キーボード・支援技術

- [ ] Tab移動順が画面の読み順と一致する
- [ ] フォーカス位置が常に視認できる
- [ ] Spaceで開始／一時停止できる
- [ ] Enterでボタンを実行できる
- [ ] テーマ切替後に状態が読み上げ可能である
- [ ] タイマーの毎秒更新が過剰に読み上げられない
- [ ] 通知音の音量を色やアイコンだけに頼らず確認できる

### 状態・テーマ

- [ ] ライトテーマのdefault / hover / focus / disabled / error
- [ ] ダークテーマのdefault / hover / focus / disabled / error
- [ ] 実行中、一時停止、完了、待機中
- [ ] プリセット0件、保存済み、読み込み中、読み込み不可
- [ ] ブラウザ通知の未許可、許可、拒否

## 8. 変更後に確認するコマンド

```powershell
# 独自の小さい操作領域が残っていないか確認
rg -n "width: (29|31|40)px|height: (29|31|40)px|min-height: (34|39|41|42|45)px" styles.css

# M3トークンの利用状況を確認
rg -n "md-sys-color|md-sys-typescale|md-sys-shape|mdc-theme" index.html styles.css app.js

# 状態・アクセシビリティ属性を確認
rg -n "aria-label|aria-pressed|aria-live|role=|aria-value|focus-visible" index.html styles.css app.js

# Material依存のバージョンが固定されているか確認
rg -n "material-components-web|@material/web|@latest" index.html README.md
```

上記の検索結果だけでは実効サイズや色コントラストは判定できません。ブラウザの開発者ツールで実測し、キーボード操作とスクリーンリーダーでも確認します。

## 9. 最終判定の書き方

修正完了後は、READMEまたはレビューコメントに次のどちらかを明記します。

### Material 3準拠を満たした場合

> Material 3のデザイントークン、標準コンポーネント、48px以上の操作領域、状態表現、アクセシビリティ要件を確認済み。独自表現はタイマー進捗とブランド装飾に限定している。

### 一部準拠に留まる場合

> Material Componentsを使用した独自テーマUI。Material Designの考え方を取り入れているが、Material 3の標準トークンまたは全コンポーネントには準拠していない。

## 10. 参考資料

- [Material Design 3 公式サイト](https://m3.material.io/)
- [Material Web Components: Material 3とデザイントークン](https://github.com/material-components/material-web/blob/main/docs/intro.md)
- [Material Web Components: Button仕様](https://github.com/material-components/material-web/blob/main/docs/components/button.md)
- [MDC Web: Icon Buttonと48×48pxのタッチターゲット](https://github.com/material-components/material-components-web/blob/master/packages/mdc-icon-button/README.md)
- [MDC Webリポジトリ](https://github.com/material-components/material-components-web)（アーカイブ済み。依存移行判断の確認用）
