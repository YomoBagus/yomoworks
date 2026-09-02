# nandemo-kaitori1.com 表示速度改善 実装指示書

> **この文書の読み手へ**
> あなたは本作業を担当する Claude Code です。
> この文書だけで作業できるよう、背景・実測値・変更コード・検証手順をすべて記載しています。
> **必ず「作業ルール」を読んでから STEP 0 に進んでください。**

---

## 0. 前提情報

### 対象サイト

| 項目 | 内容 |
|---|---|
| URL | `https://nandemo-kaitori1.com/` |
| 業種 | 不用品買取・出張買取（福岡市） |
| **構成** | **ルート直下は静的HTMLサイト**（WordPress ではない） |
| WordPress | `/wp/` 配下のブログのみ（今回の作業対象外） |
| サーバー | **エックスサーバー 共用**（`sv12206.xserver.jp` / 103.3.1.47） |
| CDN | なし |
| 既知ページ | `/`, `/company.html`, `/q&a.html` |

### 重要な制約

- **WP Rocket は使用しない。** トップページは静的HTMLのため構造上適用できず、かつ利用しない方針で確定済み。
- **有料プラグインは使わない。** サーバー標準機能とコード修正のみで対応する。
- **デザイン・レイアウト・文言・画像の見た目は一切変更しない。** 変えるのは「読み込み方」だけ。

### 目標

**PageSpeed Insights モバイル 90点以上**（現在 57点）

---

## 1. 現状の実測値

2026/09/01 計測 / Lighthouse 13.4.1 / Moto G Power エミュレーション / 低速4Gスロットリング

### ラボスコア（モバイル）＝ 改善対象

| 指標 | 現在値 | 合格ライン | 配点 | 状態 |
|---|---|---|---:|---|
| **LCP** | **8.5 秒** | 2.5 秒 | 25% | 🔴 最大の課題 |
| **TBT** | **520 ms** | 200 ms | 30% | 🟠 |
| **CLS** | **0** | 0.1 | 25% | 🟢 **満点。絶対に壊さないこと** |
| FCP | 2.2 秒 | 1.8 秒 | 10% | 🟠 |
| Speed Index | 4.5 秒 | 3.4 秒 | 10% | 🟠 |
| | | | | **総合 57 点** |

その他のスコア: ユーザー補助 96 / おすすめの方法 92 / SEO 100

### フィールドデータ（実ユーザー・CrUX 28日間）＝ すでに合格

LCP 1.8秒 / INP 90ms / CLS 0.01 / FCP 1.7秒 / **TTFB 1秒（オレンジ）**

> 実ユーザーの体感は既に良好。今回はラボスコアの改善が目的。

---

## 2. 原因分析（実測にもとづく）

### 転送量の内訳

| 内訳 | サイズ | 比率 |
|---|---|---|
| **Google タグ群（GTM / GA4 / 広告）** | **490.2 KiB** | **約 60%** |
| 自社リソース（HTML・CSS・画像） | 254 KiB | 約 31% |
| Swiper モジュール群 | 約 80 KiB | 約 9% |
| **合計** | **約 830 KiB** | 低速4G（≒200KB/秒）で**転送だけに4秒超** |

### 原因① Google タグ群が 490 KiB（TBT と LCP の主因）

| タグ | 転送サイズ | 未使用 |
|---|---|---|
| `gtm.js?id=GTM-T42S9Q8` | 159.1 KiB | 66.8 KiB |
| `gtag/destination?id=AW-362…`（Google広告） | 154.7 KiB | 65.8 KiB |
| `gtag/js?id=G-BC61XYHR8P`（GA4） | 176.4 KiB | 63.7 KiB |

長時間タスク5件を検出。帯域とメインスレッドの両方を占有している。

### 原因② メイン画像の優先度が低い（LCP の主因）

LCP要素は `images/fv0.jpg`（**50.3 KB** / 750×810 を 390×421 で表示）。
**画像は十分軽いのに 8.5 秒かかっている。**

現在のマークアップ:

```html
<img src="images/fv0.jpg"            alt="高価買取" width="750"  height="810" class="sp">
<img src="images/mainimg_210919.jpg" alt="高価買取" width="2000" height="871" class="pc" loading="lazy">
```

CSS（`style.css`）:

```css
.sp { display: none !important; }                                          /* 778行目 */
@media screen and (max-width:800px){ .sp     { display: block !important; } }        /* 787行目・打ち消し済 */
@media screen and (max-width:800px){ img.sp  { display: inline-block !important; } } /* 790行目 */
```

**問題:**

1. ブラウザは画像を**低優先度**で取得開始する
2. 「画面内にある＝重要」と判断するにはレイアウト確定＝**CSS読み込み完了**が必要
3. その間、GTM の 490 KiB が**高優先度**で帯域を占有
4. 結果、50 KB の画像が後回しにされ続ける
5. さらに **PC用画像に `loading="lazy"` が付いており、PC版の LCP を自ら悪化させている**
6. `fv0.jpg` は `loading` 属性がないため、**PCでは非表示なのにダウンロードされている**（50KB の無駄）

### 原因③ Swiper（スライダー）の読み込みが非効率

```
nandemo-kaitori1.com (255ms)
├ swiper@14.2.0/swiper-bundle.min.css   ← unpkg.com     (780ms・レンダリングブロック)
├ css/style.css                          (484ms, 5.49 KiB)
└ js/top-performance.js                  (470ms, 1.16 KiB)
   └ swiper@11/swiper-bundle.min.mjs    ← cdn.jsdelivr.net（CSSと別バージョン）
      ├ shared/swiper-core.min.mjs (20.89 KiB)
      └ modules/*.min.mjs を20個以上、個別に取得（最大 882ms）
```

- **CSS は v14.2.0、JS は v11 とバージョン不一致**
- **外部CDNが2つ**（unpkg / jsdelivr）
- **バンドルされておらず20個以上の個別リクエスト**
- **重複JS 34 KiB**（`swiper-core` が二重）
- クリティカルパスの最大待ち時間 **1,358 ms**

---

## 3. 作業ルール（厳守）

1. **1 STEP ずつ実行する。** 完了したら必ず停止し、依頼者に報告して指示を待つ。**複数STEPをまとめて進めない。**
2. **各STEPの前に対象ファイルをバックアップする。**
3. **計測は依頼者が行う。** あなたは変更を適用し、「計測してください」と依頼して待つ。
4. **見た目を変更しない。** デザイン・レイアウト・文言・配色・画像の内容は一切触らない。
5. **CLS は現在 0（満点）。** 画像の `width` / `height` 属性を削除・変更しないこと。壊すと25点失う。
6. **判断が必要な箇所では必ず依頼者に確認する。** 勝手に決めない。
7. **効果が出なかった場合も、勝手に別の施策を追加しない。** 報告して指示を仰ぐ。

### やってはいけないこと

- ❌ 広告のコンバージョン計測タグを遅延・削除する（売上計測が狂う）
- ❌ 画像の `width` / `height` 属性を外す（CLS が悪化する）
- ❌ ファーストビュー画像に `loading="lazy"` を付ける
- ❌ エックスサーバー XPageSpeed の「CSS/JavaScript の遅延読み込み」を有効化する（表示崩れの報告が多い）
- ❌ `.sp` / `.pc` の CSS ルールを削除する（他の要素でも使われている可能性がある）
- ❌ HTML の構造・クラス名を必要以上に変更する

---

## STEP 0　環境確認とバックアップ

### 目的
作業環境を確認し、いつでも元に戻せる状態を作る。

### 手順

1. **作業対象ファイルの入手方法を依頼者に確認する**（FTP / SFTP / サーバーのファイルマネージャー / ローカルにコピー済み など）
2. 以下のファイルを取得し、`backup_YYYYMMDD/` に**原本を保存する**
   - `index.html`（および下層の全 `.html`）
   - `/css/style.css`
   - `/js/top-performance.js`
   - `.htaccess`
3. `index.html` の `<head>` 全体と `/js/top-performance.js` の中身を読み、内容を報告する
4. 依頼者に**ベースライン計測**を依頼する

### 依頼者への依頼文

> https://pagespeed.web.dev/ で `https://nandemo-kaitori1.com/` を計測し、
> **モバイルのスコアと、LCP・TBT・CLS・FCP・Speed Index の5つの数値**をお知らせください。

### 完了条件
- [ ] バックアップを取得済み
- [ ] `<head>` と `top-performance.js` の内容を把握・報告済み
- [ ] ベースライン計測値を記録済み

---

## STEP 1　サーバー設定【依頼者が実施】

### 目的
コードを触らずに TTFB とファイル配信を改善する。フィールドデータの TTFB が1秒（オレンジ）のため効果が見込める。

### 内容

あなたはサーバーパネルを操作できないため、**依頼者に以下を依頼して待つ。**

> エックスサーバーのサーバーパネルで、`nandemo-kaitori1.com` に対し以下を設定してください。
>
> | 設定 | 値 |
> |---|---|
> | Xアクセラレータ | **Ver.2 に設定** |
> | ブラウザキャッシュ設定 | **ON** |
> | XPageSpeed → HTML/CSS/JavaScript の圧縮 | **ON** |
> | XPageSpeed → **CSS の遅延読み込み** | **OFF のまま**（表示崩れの原因になります） |
> | XPageSpeed → **JavaScript の遅延読み込み** | **OFF のまま**（同上） |
>
> 設定後、トップページと下層ページ（`/company.html`、`/q&a.html`）の**表示崩れがないか**ご確認のうえ、
> 再度 PageSpeed を計測してお知らせください。

### 完了条件
- [ ] 設定完了の連絡を受けた
- [ ] 表示崩れがないことを確認済み
- [ ] 計測値を記録済み

---

## STEP 2　メイン画像の `<picture>` 化＋先読み指定＋lazy除去 ⭐最重要

### 目的
**LCP 8.5秒 → 2〜3秒台を狙う。** 本作業で最も効果が大きい。

### 対象ファイル
`index.html`（`<head>` と `<div id="mainimg">` 内）

### 変更① マークアップ

**変更前**（`<div id="mainimg">` 内）:

```html
<img src="images/fv0.jpg"            alt="高価買取" width="750"  height="810" class="sp">
<img src="images/mainimg_210919.jpg" alt="高価買取" width="2000" height="871" class="pc" loading="lazy">
```

**変更後**:

```html
<picture>
  <source media="(max-width:800px)" srcset="images/fv0.jpg" width="750" height="810">
  <img src="images/mainimg_210919.jpg" alt="高価買取"
       width="2000" height="871" fetchpriority="high" decoding="async">
</picture>
```

### 変更② `<head>` に先読み指定を追加

`<head>` 内、既存の `<link rel="stylesheet">` **より前**に追加する。

```html
<link rel="preload" as="image" href="images/fv0.jpg"
      media="(max-width:800px)" fetchpriority="high">
<link rel="preload" as="image" href="images/mainimg_210919.jpg"
      media="(min-width:801px)" fetchpriority="high">
```

### 注意点

- **ブレークポイントは `800px`。** 既存CSS（`@media screen and (max-width:800px)`）と一致させること。異なる値を使わない。
- `width` / `height` は `<source>` と `<img>` の**両方に必ず指定する**。省略すると CLS が悪化する。
- `class="sp"` / `class="pc"` は**この2つの img からのみ外す**。`.sp` / `.pc` の CSS ルール自体は削除しない（他要素で使用されている可能性がある）。
- `loading="lazy"` は**必ず削除する**。ファーストビュー画像に付けてはいけない。
- `images/` のパスが相対パスであることを確認する。下層ページで同じ画像を使っている場合はパスの階層に注意。

### 期待される効果
- 不要な方の画像が**一切ダウンロードされなくなる**（PCで50KBの無駄が消える）
- CSS の読み込み完了を待たずに画像取得が最優先で始まる
- PC版の LCP も改善（`lazy` 除去による）

### 検証

1. スマホ幅（375px 等）で `fv0.jpg` が表示されることを確認
2. PC幅（1200px 等）で `mainimg_210919.jpg` が表示されることを確認
3. DevTools のネットワークタブで、**表示していない方の画像が読み込まれていないこと**を確認
4. **見た目が変更前と完全に同一であること**を確認
5. 依頼者に計測を依頼

### 完了条件
- [ ] 両方の画面幅で正しい画像が表示される
- [ ] 不要な画像が読み込まれていない
- [ ] 見た目に変化がない
- [ ] **CLS が 0 のまま**であることを計測で確認
- [ ] LCP の改善を確認

---

## STEP 3　Google タグの読み込みタイミング変更 ⭐効果大

### 目的
**TBT 520ms → 200ms以下を狙う。** 帯域も空くため LCP にも効く。

### ⚠️ 着手前に必ず依頼者へ確認すること

> **確認事項**
>
> 1. Google広告のコンバージョン計測タグ（`AW-362…`）は、GTM 経由で発火していますか？ 直接設置ですか？
> 2. コンバージョン計測タグは**遅延対象から外します**。それでよろしいですか？
> 3. アクセス解析（GA4）は「操作 or 読み込み完了後3秒」で発火させます。
>    この場合、**読み込み後3秒以内に何も操作せず離脱した訪問者は計測されません。**
>    数％程度セッション数が減る可能性がありますが、進めてよろしいですか？

**上記の回答を得るまで、コードを変更しないこと。**

### 対象ファイル
`index.html` の `<head>` 内 GTM スニペット

### 変更内容

**変更前**（GTM 標準スニペット）:

```html
<!-- Google Tag Manager -->
<script>(function(w,d,s,l,i){ ... })(window,document,'script','dataLayer','GTM-T42S9Q8');</script>
<!-- End Google Tag Manager -->
```

**変更後**:

```html
<!-- Google Tag Manager (遅延読み込み版) -->
<script>
(function () {
  var fired = false;
  function loadGTM() {
    if (fired) return;
    fired = true;
    (function (w, d, s, l, i) {
      w[l] = w[l] || [];
      w[l].push({ 'gtm.start': new Date().getTime(), event: 'gtm.js' });
      var f = d.getElementsByTagName(s)[0],
          j = d.createElement(s),
          dl = l != 'dataLayer' ? '&l=' + l : '';
      j.async = true;
      j.src = 'https://www.googletagmanager.com/gtm.js?id=' + i + dl;
      f.parentNode.insertBefore(j, f);
    })(window, document, 'script', 'dataLayer', 'GTM-T42S9Q8');
  }
  ['scroll', 'pointerdown', 'keydown', 'touchstart', 'mousemove'].forEach(function (e) {
    window.addEventListener(e, loadGTM, { once: true, passive: true });
  });
  window.addEventListener('load', function () { setTimeout(loadGTM, 3000); });
})();
</script>
<!-- End Google Tag Manager -->
```

### 注意点

- **`<noscript>` の GTM タグ（`<body>` 直後）はそのまま残す。**
- **コンテナID `GTM-T42S9Q8` を書き換えないこと。**
- 3秒のフォールバックにより、無操作の訪問者も読み込み完了3秒後には計測される。
- **フォームのサンキューページなど、コンバージョン計測が必要なページではこの変更を適用しない。**該当ページは元のスニペットのままにする。

### 検証

1. ページを開き、何も操作せず GA4 のリアルタイムを確認 →**3秒後に計測されること**
2. スクロールした場合、**即座に計測されること**
3. DevTools ネットワークタブで、初期読み込み時に `gtm.js` が**読み込まれていないこと**を確認
4. 依頼者に計測を依頼

### 完了条件
- [ ] 事前確認3項目の回答を得た
- [ ] GA4 リアルタイムで計測を確認
- [ ] コンバージョン計測ページには適用していない
- [ ] TBT の改善を確認

---

## STEP 4　JavaScript の遅延読み込み（defer 化）

### 目的
表示をブロックしている JavaScript を、表示後に実行させる。FCP の改善。

### 対象ファイル
`index.html`（`<script>` タグ）

### 手順

1. `<head>` 内および `<body>` 内の `<script src="...">` をすべて洗い出し、一覧を報告する
2. 各スクリプトに `defer` 属性を付与する

```html
<!-- 変更前 -->
<script src="js/top-performance.js"></script>

<!-- 変更後 -->
<script src="js/top-performance.js" defer></script>
```

### 注意点

- **インラインスクリプト（`src` なし）には `defer` は効かない。** 対象外。
- `document.write()` を使っているスクリプトに `defer` を付けると壊れる。事前に中身を確認する。
- `top-performance.js` は Swiper を動的 import している。`defer` 化後に**スライダーが動くことを必ず確認する。**
- STEP 3 で作成した GTM 遅延スクリプトはインラインなので対象外。

### 検証
1. **スライドショーが正常に動作すること**
2. ハンバーガーメニューが開閉すること
3. その他の動的な要素が動作すること
4. 依頼者に計測を依頼

### 完了条件
- [ ] スクリプト一覧を報告済み
- [ ] スライダー・メニューが正常動作
- [ ] 計測値を記録

---

## STEP 5　CSS の圧縮

### 目的
CSS ファイルの容量削減。

> **注意:** `style.css` は 5.5 KiB、Swiper CSS は 3.7 KiB（圧縮済み）。
> **効果はごくわずか（1 KiB 程度）と見込まれる。**過度な期待をしないこと。
> STEP 1 の XPageSpeed で既に圧縮されている場合は、この STEP をスキップしてよいか依頼者に確認する。

### 手順

1. STEP 1 の XPageSpeed による圧縮が効いているか確認する
2. 効いていない場合のみ、`css/style.css` を手動で圧縮する
3. **元ファイルは `style.css.bak` として必ず残す**

### 検証
- [ ] 全ページで表示崩れがないこと（圧縮ミスでCSSが壊れやすいので入念に）

---

## STEP 6　画像の WebP 化

### 目的
画像容量の削減。

> **注意:** PSI の「適切なサイズ」指摘は **4.3 KiB のみ**。画像は既に最適化されている。
> LCP画像 `fv0.jpg` は 50.3 KB → WebP化で約30 KB（**20 KB削減 ≒ 低速4Gで0.1秒**）。
> **効果は限定的。**

### 手順

1. `images/` 内の画像を WebP に変換する（**元ファイルは削除せず残す**）
2. STEP 2 で作成した `<picture>` に WebP の `<source>` を**先頭に**追加する

```html
<picture>
  <source media="(max-width:800px)" type="image/webp" srcset="images/fv0.webp"            width="750"  height="810">
  <source media="(max-width:800px)"                   srcset="images/fv0.jpg"             width="750"  height="810">
  <source                            type="image/webp" srcset="images/mainimg_210919.webp" width="2000" height="871">
  <img src="images/mainimg_210919.jpg" alt="高価買取"
       width="2000" height="871" fetchpriority="high" decoding="async">
</picture>
```

3. `<head>` の preload も WebP を指すよう更新する

```html
<link rel="preload" as="image" href="images/fv0.webp" type="image/webp"
      media="(max-width:800px)" fetchpriority="high">
```

### 注意点
- **`<source>` の順序が重要。** WebP を先、JPEG を後に置く。逆にすると WebP が使われない。
- **`width` / `height` を全ての `<source>` に付ける**（CLS維持）。
- 変換時の画質は 80 程度。**画質劣化が見た目でわからないことを確認する。**

### 検証
- [ ] WebP対応ブラウザで WebP が読み込まれること
- [ ] 画質の劣化が見た目でわからないこと
- [ ] CLS が 0 のままであること

---

## STEP 7　フォント読み込みの見直し

### 目的
Webフォントの読み込み最適化。

> **注意:** PSI にフォント関連の指摘は**1件も出ていない。**
> Webフォントを使用していない可能性が高く、**その場合この STEP は不要。**

### 手順

1. **まず調査する。** `index.html` と `style.css` に以下がないか確認する
   - `fonts.googleapis.com` / `fonts.gstatic.com` への `<link>`
   - `@font-face` 宣言
   - `@import url(...)` によるフォント読み込み
2. **見つからなければ、この STEP は不要と報告して次へ進む**
3. 見つかった場合のみ、以下を検討して依頼者に提案する
   - `font-display: swap` の付与
   - `<link rel="preload" as="font" crossorigin>` の追加
   - 日本語フォントの場合は、**端末標準フォントへの変更を提案する**（見た目が変わるため、必ず依頼者の判断を仰ぐ）

### 完了条件
- [ ] 調査結果を報告済み
- [ ] （該当時のみ）依頼者の判断を得て対応済み

---

## STEP 8　最終確認と計測

### 手順

1. **全ページの表示確認**（PC・スマートフォンの実機）
   - `/`、`/company.html`、`/q&a.html`
2. **コンバージョン導線の確認**（最重要）
   - [ ] 電話発信ボタン（`tel:` リンク）が実機でタップして発信できる
   - [ ] LINE ボタンが正しく遷移する
   - [ ] お問い合わせフォームが送信でき、**自動返信メールが届く**
   - [ ] スライドショーが動作する
   - [ ] ハンバーガーメニューが開閉する
3. **解析の疎通確認**
   - [ ] GA4 リアルタイムに記録される
   - [ ] Google広告のコンバージョン計測が動作している
4. **最終計測**（依頼者に依頼）

> PageSpeed を**5回計測し、その中央値**でご判定ください。
> 1回の結果では ±5点程度ぶれます。

### 完了条件
- [ ] 全チェック項目クリア
- [ ] モバイル 90点以上を達成

---

## STEP 9　Swiper の整理【90点に届かない場合のみ】

> **STEP 8 で 90点に到達した場合、この STEP は実施しない。**

### 目的
LCP・FCP・SI のさらなる改善。原因③への対応。

### 手順

1. **Swiper のバージョンを統一する**（現在 CSS: v14.2.0 / JS: v11 と不一致）
   - 依頼者に、どちらのバージョンに揃えるか確認する
2. **セルフホスト化**
   - `swiper-bundle.min.css` と `swiper-bundle.min.js` をダウンロードし、`/css/` `/js/` に配置
   - `unpkg.com` / `cdn.jsdelivr.net` への参照を自社パスに置き換える
3. **バンドル版1ファイルにまとめる**
   - 現在20個以上の ESM モジュールを個別取得している状態を解消する
   - `swiper-bundle.min.js`（全モジュール入り）1ファイルの読み込みに変更する
4. **Swiper CSS のインライン化を検討**
   - 3.7 KiB と小さいため、`<style>` で `<head>` にインライン化すればレンダリングブロックが解消する

### 注意点
- **バージョン変更により API が変わる可能性がある。** 変更後は必ずスライダーの動作を確認する。
- v11 と v14 では初期化オプションの互換性に差異がある可能性がある。動かない場合は依頼者に報告し、元のバージョンに揃える方針を相談する。

### 検証
- [ ] スライドショーが変更前とまったく同じ動きをすること
- [ ] 外部CDNへのリクエストが消えていること
- [ ] リクエスト数が大幅に減っていること

---

## 計測記録表

各STEP完了ごとに記入すること。

| STEP | 内容 | 実施日 | スコア | LCP | TBT | CLS | FCP | SI |
|---|---|---|---:|---:|---:|---:|---:|---:|
| 0 | ベースライン | | **57** | **8.5s** | **520ms** | **0** | **2.2s** | **4.5s** |
| 1 | サーバー設定 | | | | | | | |
| 2 | メイン画像 `<picture>` 化 | | | | | | | |
| 3 | タグ読み込み変更 | | | | | | | |
| 4 | JS の defer 化 | | | | | | | |
| 5 | CSS 圧縮 | | | | | | | |
| 6 | 画像 WebP 化 | | | | | | | |
| 7 | フォント見直し | | | | | | | |
| 8 | 最終計測 | | | | | | | |
| 9 | Swiper 整理（必要時） | | | | | | | |

---

## 到達見込み

| 段階 | 見込みスコア |
|---|---|
| STEP 1〜4 完了時 | **75〜85点** |
| STEP 5〜7 完了時 | **80〜88点** |
| STEP 9 まで完了時 | **90点以上** |

**STEP 2 と STEP 3 で大半が決まります。**STEP 5・6・7 は効果が小さいことが実測でわかっているため、
STEP 4 の時点で 90点に届いた場合は、依頼者に「以降は実施不要では」と提案してください。

---

## 参考：スコアの計算根拠

Lighthouse のパフォーマンススコアは5指標の加重平均です。

| 指標 | 配点 |
|---|---:|
| TBT（操作可能になるまでの待ち時間） | 30% |
| LCP（主要コンテンツの表示時間） | 25% |
| CLS（画面のガタつき） | 25% |
| FCP（最初の描画） | 10% |
| Speed Index（体感速度） | 10% |

**CLS は既に満点(0)のため、25%分は確保済みです。絶対に壊さないでください。**
仮に LCP 以外の4指標を全て満点にしても、LCP が 8.5秒のままなら**総合77点が上限**です。
だからこそ STEP 2 が最優先になります。
