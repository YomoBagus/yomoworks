# nandemo-kaitori1.com PageSpeed 90点化 実行計画

対象: https://nandemo-kaitori1.com/ （不用品買取・出張買取 NANDEMO KAITORI / 福岡）
手持ちツール: WP Rocket
作成日: 2026-09-01
更新日: 2026-09-01（サイト構成が確定したため改訂）

---

> ## ✔ 確定事項
>
> クライアント確認により、**トップページは静的HTML（WordPress ではない）** と確定しました。
>
> - **WP Rocket はトップページに適用されません。** 対応は **§4（ケースA）** で進めます。
> - **§5（ケースB）は不適用**となりました。`/wp/` 配下のブログに WP Rocket を使う場合の参考として残しています。

---

## 0. この調査の前提と限界（先に読んでください）

本作業環境はネットワーク egress が許可リスト方式で制限されており、
**対象サイトへ直接アクセスできませんでした**（`nandemo-kaitori1.com:443` はプロキシが 403 で拒否）。
PageSpeed Insights API も匿名枠の日次クォータが枯渇していて実測できていません。

```
[agent-proxy] nandemo-kaitori1.com:443 — connect_rejected (organization policy)
PSI API      — HTTP 429 Quota exceeded (anonymous shared project)
```

したがって本書は **実測スコアに基づくものではなく**、DNS・逆引き・検索インデックスから
判明した事実と、そこから導かれる構成推定に基づく計画です。
§1 の検証（所要5分）を最初に実施し、結果によって §4 / §5 のどちらを採るか分岐してください。

### 調査で「確定」した事実

| 項目 | 値 | 取得方法 |
|---|---|---|
| IPアドレス | `103.3.1.47` | DNS Aレコード |
| ホスティング | **エックスサーバー** 共用 `sv12206.xserver.jp` | 逆引き PTR |
| CDN | **なし**（単一IP、Cloudflare等の前段なし） | Aレコード |
| IPv6 | なし | AAAA なし |
| 既知URL | `/`, `/company.html`, `/q&a.html`, `/wp/?p=146` | 検索インデックス |

### 確定した構成

**ルート直下は WordPress ではなく、手書きの静的HTMLサイト。WordPress は `/wp/` 配下のブログのみ。**
（当初は下記4点からの推定でしたが、**クライアント確認により確定**しました。）

根拠:

1. `/company.html`, `/q&a.html` と **`.html` 拡張子**。WordPress の標準パーマリンクではまず出ない形。
2. とくに **`q&a.html` はファイル名に生の `&` を含む**。WordPress はスラッグをサニタイズするため
   この URL は生成不可能。＝ 人力で置かれた静的ファイル。
3. ブログ記事が **`/wp/?p=146`**。つまり WordPress は
   - サブディレクトリ `/wp/` にインストールされ、
   - パーマリンクが「基本」（`?p=` 形式）のまま
   という、**既存の静的サイトに後からブログだけ足した**典型パターン。
4. 記事の日付が 2021-06-10 で、インデックスされている記事が1件のみ。ブログはほぼ運用されていない。

---

## 1. 構成の確定（完了）

**ここが本件の最大の分岐点でした。結果は「静的HTML」で確定です。**
以下のコマンドは、改修中に再確認したい場合の参考として残します。

```bash
# ① レスポンスヘッダ（サーバ・圧縮・キャッシュ）
curl -sI https://nandemo-kaitori1.com/ | grep -iE 'server|x-powered-by|cache-control|content-encoding|link'

# ② ルートが WordPress かどうかの判定（最重要）
curl -s https://nandemo-kaitori1.com/ | grep -oE 'wp-content|wp-includes|wp-json|wp-emoji' | sort -u

# ③ 読み込んでいるCSS/JS/画像の一覧
curl -s https://nandemo-kaitori1.com/ | grep -oE '(href|src)="[^"]+\.(css|js|jpg|jpeg|png|webp|svg)[^"]*"' | sort -u

# ④ 転送量と応答時間
curl -s -o /dev/null -w 'TTFB:%{time_starttransfer}s TOTAL:%{time_total}s SIZE:%{size_download}B ENC:%{content_type}\n' \
  https://nandemo-kaitori1.com/
```

### 判定（確定結果：静的HTML）

- **② が空 → ルートは静的HTML。** ← **今回はこちら。**
  → **WP Rocket はトップページに対して一切効きません。** **§4（ケースA）で対応します。**
- ~~② に `wp-content` 等が出る → ルートも WordPress。§5（ケースB）へ。~~ → 不該当

> ### なぜこれが重要か
> WP Rocket は WordPress の PHP 実行時にフック（出力バッファ）して HTML を書き換えるプラグインです。
> `index.html` のような静的ファイルは **Apache/nginx が直接返すため PHP を通らず、WP Rocket の
> 処理対象になりません**。キャッシュも、CSS/JS最適化も、LazyLoad も、Delay JS も、
> 静的HTMLページには適用されません。
>
> つまり本件では、**「WP Rocket を入れてトップページを90点にする」は原理的に不可能**で、
> 効くのは `/wp/` 配下のブログ記事だけです。
> WP Rocket の設定をいくら詰めても、トップページのスコアは1点も動きません。

---

## 2. 「90点以上」の中身を分解する

PageSpeed Insights の Performance スコアは Lighthouse の5指標の加重平均です（v10以降）。

| 指標 | 配点 | 90点達成に必要な水準（モバイル） |
|---|---:|---|
| **TBT** (Total Blocking Time) | **30%** | ≤ 200ms（実質 150ms 以下が安全） |
| **LCP** (Largest Contentful Paint) | **25%** | ≤ 2.5s（実質 2.0s 前後） |
| **CLS** (Cumulative Layout Shift) | **25%** | ≤ 0.1（実質 0.05 以下） |
| FCP (First Contentful Paint) | 10% | ≤ 1.8s |
| SI (Speed Index) | 10% | ≤ 3.4s |

**TBT と CLS で合計55%** を占める点が重要です。画像を軽くするだけ（LCP対策）では 90 に届きません。
JavaScript の実行時間（TBT）とレイアウトシフト（CLS）を潰す必要があります。

### モバイル計測の過酷さ

PSI のモバイルは以下の条件でエミュレートされます。実機の高速回線とは別物です。

- CPU: 約4倍のスロットリング（Moto G Power 相当）
- 回線: 低速4G（下り 1.6Mbps / RTT 150ms）

→ **JS の総量がほぼそのまま TBT に効きます。** jQuery + スライダープラグイン + 計測タグの構成は、
それだけで TBT 500〜1500ms に達し、この時点で 90 点は不可能になります。

### 現実的な目標設定（重要）

- **デスクトップ 90+ は十分現実的。** 画像とキャッシュだけでほぼ到達します。
- **モバイル 90+ は本気の作業が必要。** 画像・フォント・JS の3点を全部やって初めて届きます。
- スコアは実行ごとに **±5点程度ブレます**。「90点」は **5回計測の中央値** で判定してください。
- SEO に実際に効くのは PSI 上部の **フィールドデータ（CrUX / 実ユーザー）** であり、
  90点というラボスコアはあくまで改善の指標です。ここは分けて考えてください。

---

## 3. パフォーマンス・バジェット（モバイル 90+ の目安）

改修時はこの数値を上限として設計してください。

| 項目 | 上限 |
|---|---|
| TTFB | < 500ms |
| HTML（圧縮後） | < 30KB |
| CSS 合計 | < 60KB（うちクリティカル部をインライン化 ≤ 14KB） |
| JS 合計（圧縮後） | < 100KB |
| LCP画像（ヒーロー画像） | **< 80KB** |
| Webフォント | **0（システムフォント推奨）** または サブセット < 100KB |
| ファーストビュー総転送量 | < 400KB |

---

## 4. ケースA：ルートが静的HTML【確定・本件はこちら】

WP Rocket は使えません。代わりに **①静的HTMLの直接改修 + ②エックスサーバー側の設定** で攻めます。
静的サイトはテーマ・プラグインの肥大がない分、**WordPress より 90点到達は本来ラクです。**

### 4-1. 画像（最優先・効果最大）

買取サイトは商品写真・実績写真が多く、ここが最大のボトルネックである可能性が最も高いです。

1. **実表示サイズまでリサイズ。** スマホ幅375pxの枠に 2000px超の写真を入れているケースが頻出。
   表示幅の最大2倍（Retina想定）まで縮小する。
2. **WebP化。** JPEG比で 25〜35% 削減。
   - 手作業なら [Squoosh](https://squoosh.app/) / `cwebp -q 80`
   - または エックスサーバーの XPageSpeed の WebP変換（§4-5）
   - `<picture>` で WebP + JPEG フォールバックを組む
3. **ヒーロー画像（LCP要素）を最適化。**
   ```html
   <link rel="preload" as="image" href="/img/hero.webp" fetchpriority="high">
   ...
   <img src="/img/hero.webp" width="750" height="500"
        fetchpriority="high" decoding="async" alt="出張買取">
   ```
   **ヒーロー画像には `loading="lazy"` を付けない**（LCPが遅延します。よくある事故）。
4. **ファーストビュー外の画像は遅延読み込み。**
   ```html
   <img src="..." width="400" height="300" loading="lazy" decoding="async" alt="...">
   ```
5. **すべての `<img>` に `width` / `height` を明記。** → CLS（配点25%）に直結。

### 4-2. Webフォント（日本語サイト特有の地雷）

**Google Fonts の Noto Sans JP などをフルセットで読み込んでいたら、それだけで致命傷です**
（日本語フォントは 2〜5MB規模）。

- **推奨：システムフォントスタックに置き換え（転送量 0）**
  ```css
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans",
                 "Hiragino Kaku Gothic ProN", "Noto Sans JP", "Yu Gothic",
                 Meiryo, sans-serif;
  }
  ```
  買取サイトでフォントの個性が売上を左右することはまずありません。費用対効果が最も高い判断です。
- どうしても使うなら: サブセット化（使用文字だけ）＋ セルフホスト ＋ `font-display: swap` ＋ `preload`。
- **アイコンフォント（Font Awesome 等）は全廃を推奨。** 使っている数個のアイコンを
  インライン SVG に置換すれば、CSS + フォントファイルの数百KBが消えます。

### 4-3. CSS

- **クリティカルCSSを `<head>` にインライン化**（ファーストビュー分、目安14KB以内）。
- 残りは遅延読み込み:
  ```html
  <link rel="stylesheet" href="/css/main.css" media="print" onload="this.media='all'">
  <noscript><link rel="stylesheet" href="/css/main.css"></noscript>
  ```
- Bootstrap や自作CSSをフルで読んでいる場合、実使用は通常10%以下。
  未使用CSSの削除で数十〜百数十KB削減できます（Chrome DevTools の Coverage タブで確認）。

### 4-4. JavaScript（TBT対策・配点30%）

1. **`<head>` 内の同期 `<script>` を全廃。** すべて `defer` を付けて `</body>` 直前へ。
   ```html
   <script src="/js/main.js" defer></script>
   ```
2. **jQuery とスライダープラグインの見直し。** これが TBT の主犯であることが多いです。
   - スライダーは CSS の `scroll-snap` かごく小さなバニラJSに置換できないか検討。
   - jQuery 依存が浅ければ素の DOM API に書き換え（jQuery本体で 30KB + 実行時間）。
3. **サードパーティタグを遅延実行。** GA4 / GTM / 広告タグ / チャットウィジェット / LINE等は、
   初回のユーザー操作（scroll / click / touch）まで、または `load` イベント後まで遅らせる。
   ```html
   <script>
   // 初回操作またはload後に計測タグを注入する例
   (function () {
     var loaded = false;
     function boot() {
       if (loaded) return; loaded = true;
       var s = document.createElement('script');
       s.src = 'https://www.googletagmanager.com/gtag/js?id=G-XXXXXXX';
       s.async = true;
       document.head.appendChild(s);
     }
     ['scroll','pointerdown','keydown','touchstart'].forEach(function (e) {
       window.addEventListener(e, boot, { once: true, passive: true });
     });
     window.addEventListener('load', function () { setTimeout(boot, 3000); });
   })();
   </script>
   ```
   > 注意: 計測タグを遅らせると **アクセス解析の計測数がわずかに減ります**。
   > スコアと計測精度のトレードオフです。事業判断として合意を取ってから実施してください。

### 4-5. エックスサーバー側の設定

サーバーパネルの設定は **静的HTMLにも効きます**（WP Rocket と違い PHP を経由しないため）。
ここがケースAでの「プラグイン相当」の役割を果たします。

| 設定 | 推奨 | 備考 |
|---|---|---|
| **Xアクセラレータ Ver.2** | **ON** | 静的ファイル配信とPHPの高速化。まず有効化 |
| **ブラウザキャッシュ設定** | **ON** | 静的アセットに `Cache-Control` が付与される |
| **サーバーキャッシュ設定** | ON（静的サイトなら安全） | ※ケースB（WordPress）では OFF。§5-4参照 |
| **XPageSpeed** | **部分的に ON** | 下記の注意を必ず読むこと |

**XPageSpeed の使い分け（重要）**
Google の `mod_pagespeed` ベースの機能で、Apache レベルで動くため静的HTMLにも適用されます。ただし:

- ✅ 有効化してよい: **HTML / CSS / JavaScript の圧縮（minify）**、**画像の最適化・WebP変換**
- ❌ 避けるべき: **CSSの遅延読み込み / JavaScriptの遅延読み込み**
  → レイアウト崩れ・JS破損の報告が多い機能です。この2つは §4-3 / §4-4 のように
    自分のコード側で明示的に制御するほうが確実で、デバッグも可能です。
- 有効化後は **必ずトップページ・各下層ページの表示崩れを目視確認**してください。

**.htaccess で圧縮とキャッシュを明示**（サーバーパネル設定と併用可）:

```apache
# Brotli / gzip 圧縮
<IfModule mod_brotli.c>
  AddOutputFilterByType BROTLI_COMPRESS text/html text/css text/javascript application/javascript application/json image/svg+xml
</IfModule>
<IfModule mod_deflate.c>
  AddOutputFilterByType DEFLATE text/html text/css text/javascript application/javascript application/json image/svg+xml
</IfModule>

# ブラウザキャッシュ
<IfModule mod_expires.c>
  ExpiresActive On
  ExpiresByType image/webp "access plus 1 year"
  ExpiresByType image/jpeg "access plus 1 year"
  ExpiresByType image/png  "access plus 1 year"
  ExpiresByType image/svg+xml "access plus 1 year"
  ExpiresByType text/css   "access plus 1 month"
  ExpiresByType application/javascript "access plus 1 month"
  ExpiresByType text/html  "access plus 10 minutes"
</IfModule>
```

> 変更前に必ず `.htaccess` のバックアップを取ってください。記述ミスで 500 エラーになります。

### 4-6. ケースAでの想定コストと効果

| 施策 | 工数目安 | スコア寄与 |
|---|---|---|
| 画像リサイズ + WebP化 + lazy/fetchpriority | 4〜8h | **大**（LCP / SI） |
| Webフォント撤去 or サブセット | 1〜2h | **大**（LCP / FCP / SI） |
| JS の defer化 + サードパーティ遅延 | 3〜6h | **大**（TBT） |
| `width`/`height` 付与 | 1〜2h | **大**（CLS） |
| クリティカルCSSインライン化 | 3〜5h | 中（FCP / LCP） |
| Xserver 設定 + .htaccess | 1h | 中（TTFB / 転送量） |

**合計 13〜24時間程度で、モバイル 90+ は十分射程内です。**

---

## 5. ケースB：WordPress の場合の WP Rocket 設定【本件では不適用・参考】

> **本件のトップページは静的HTMLのため、この章は適用されません。**
> `/wp/` 配下のブログに WP Rocket を適用する場合、および将来 WordPress 化した場合の
> 参考資料として残します。

上から順に、**1項目ずつ有効化 → 表示確認 → 計測** を繰り返してください。
まとめて有効化すると事故の切り分けができません。

### 5-1. WP Rocket 推奨設定

**キャッシュ**
- ✅ モバイル端末用のキャッシュを有効化 → **ON**
- ✅ モバイル端末に専用のキャッシュファイルを使用 → **ON**（モバイル最適化の前提）
- キャッシュの有効期限: 10時間程度

**ファイルの最適化**
- ✅ CSSファイルを縮小化 → ON
- ✅ **使用していないCSSを削除（Remove Unused CSS）** → **ON**
  ページ単位で未使用CSSを除去。最大70%削減の報告があり、**単体で最も効く機能**。
  ただし最も壊れやすくもあるため、有効化後に全ページの表示を確認。
  崩れたら該当セレクタを「セーフリスト」に追加して解決します。
  （※「CSSを非同期で読み込む」との併用は不可。Remove Unused CSS を優先）
- ✅ JavaScriptファイルを縮小化 → ON
- ⚠️ JavaScriptファイルを連結 → **OFF 推奨**（HTTP/2 環境では効果が薄く、破損リスクのみ増える）
- ✅ JavaScriptの遅延読み込み（Load JS deferred） → ON
- ✅ **JavaScriptの実行を遅延（Delay JS Execution）** → **ON**
  **TBT（配点30%）に最も効く機能。** ユーザー操作まで JS 実行を止めます。
  ただし **ファーストビューに必要なものは除外リストへ**:
  - メインスライダー / カルーセル
  - モバイルハンバーガーメニュー
  - 電話発信・LINE ボタンなど CTA 系のスクリプト

  → 除外を怠ると「スコアは上がったが**電話ボタンが押せない**」という、
     買取サイトにとって最悪の事故になります。§6 の確認手順を必ず実施してください。

**メディア**
- ✅ 画像の遅延読み込み（LazyLoad） → ON
- ✅ **重要な画像を最適化する（Optimize Critical Images）** → **ON**
  LCP画像を自動検出し、**プリロード + `fetchpriority="high"` 付与 + LazyLoad から自動除外**を
  まとめて行います。モバイル版とデスクトップ版を個別に処理してくれるため、手動調整より確実です。
- ✅ iframe / 動画の遅延読み込み → ON
- ✅ YouTube iframe をプレビュー画像に置換 → ON（動画があれば効果大）
- ✅ **不足している画像サイズを追加** → **ON**（CLS対策・配点25%）

**プリロード**
- ✅ キャッシュを事前読み込みする → ON
- ✅ フォントの先読み → 使用中のWebフォントを指定
- ✅ DNSリクエストを事前に読み込む → 外部ドメイン（計測タグ等）を登録
- ⚠️ リンクの先読み（Preload Links） → ON でよいが、スコアには影響しません（体感速度のみ改善）

**データベース / ハートビート**
- スコアには影響しません。運用の軽量化として任意で。

### 5-2. WP Rocket だけでは埋まらない穴

**WP Rocket は画像ファイル自体を圧縮しません。** ここを誤解している人が非常に多いです。
LCP を落とすには、別途どれかが必要です。

- **Imagify**（WP Rocket と同じ開発元、連携がスムーズ）
- ShortPixel / EWWW Image Optimizer
- または エックスサーバーの XPageSpeed の WebP変換

同様に、WP Rocket では以下も解決しません:
- **テーマ本体の重さ**（読み込むCSS/JSの絶対量）
- **プラグインの過多**（各プラグインが独自のCSS/JSを吐く）
- **TTFB**（サーバー応答速度そのもの）
- **日本語Webフォントの重量**

→ §4-2（フォント）と §4-4（サードパーティ）はケースBでも同様に実施が必要です。

### 5-3. 使っていないプラグインの棚卸し

WordPress で 90点に届かない典型原因は「プラグインが各自CSS/JSを全ページで読み込んでいる」ことです。
Contact Form 7、スライダー系、ページビルダー系が代表格。
Query Monitor などで実際の読み込みを確認し、不要なものは停止、
必要なものは該当ページ以外で `wp_dequeue_script()` してください。

### 5-4. ⚠️ エックスサーバー設定との競合（ケースB特有）

WP Rocket と Xserver の機能は **役割が重複するため、両方ONにすると事故ります。**

| Xserver 設定 | WP Rocket 併用時 | 理由 |
|---|---|---|
| **サーバーキャッシュ設定** | **OFF 必須** | 二重キャッシュとなり、更新が反映されない／古いページが配信される |
| **XPageSpeed の CSS/JS 遅延読み込み** | **OFF 必須** | WP Rocket の同等機能と衝突して JS が壊れる |
| XPageSpeed の minify | OFF 推奨 | WP Rocket 側でやるので不要。二重処理は避ける |
| **Xアクセラレータ Ver.2** | **ON でよい** | 静的ファイル配信とPHP高速化。競合しません |
| ブラウザキャッシュ設定 | ON でよい | 競合しません |

---

## 6. リリース前チェックリスト（買取サイト特有・必須）

スコアより **コンバージョン導線が壊れていないこと** が優先です。
実機（iPhone Safari / Android Chrome）で必ず確認してください。

- [ ] **電話発信ボタン**（`tel:` リンク）が実機でタップして発信できる
- [ ] **LINE 追加 / 問い合わせボタン**が正しく遷移する
- [ ] **お問い合わせフォームが送信でき、自動返信メールが届く**
- [ ] 追従フッターCTA（ある場合）が表示され、タップできる
- [ ] メインスライダー / カルーセルが動く
- [ ] スマホのハンバーガーメニューが開閉する
- [ ] `/q&a.html` `/company.html` など全下層ページの表示崩れがない
- [ ] Google Analytics でリアルタイム計測が記録される（タグ遅延後も）
- [ ] 画像がすべて表示される（WebP化・LazyLoad後の抜けがないか）

---

## 7. 計測プロトコル

1. 計測は **PageSpeed Insights**（https://pagespeed.web.dev/）で、
   URL は `https://nandemo-kaitori1.com/` を指定。
2. **モバイルとデスクトップを別々に記録。** 90点の難所はモバイル。
3. **5回計測して中央値を採用。** 単発の数値で判断しない（±5点ブレます）。
4. 1施策ごとに計測し、**どの施策が何点効いたかを記録**する。
   まとめて実施すると、効かなかった施策や壊した施策を特定できません。
5. 施策前に **現状のベースラインを必ず記録**（スコア / LCP / TBT / CLS の4値）。

### 記録テンプレート

| # | 施策 | 実施日 | Mobile | LCP | TBT | CLS | Desktop | 備考 |
|---|---|---|---:|---:|---:|---:|---:|---|
| 0 | ベースライン（施策前） | | | | | | | |
| 1 | | | | | | | | |

---

## 8. 推奨実施順（ケースA想定）

| Step | 内容 | 理由 |
|---|---|---|
| **0** | ~~構成判定~~（完了）+ ベースライン計測 | 構成は静的HTMLで確定。計測は効果測定の基準として必須 |
| 1 | Xserver 設定（Xアクセラレータ / ブラウザキャッシュ）+ `.htaccess` | 低リスク・即効・コード変更不要 |
| 2 | 画像の棚卸し → リサイズ / WebP / `width`・`height` / lazy / fetchpriority | LCP・CLS・SI に最大の寄与 |
| 3 | Webフォント撤去（システムフォント化）+ アイコンフォント → SVG | 日本語サイト最大の地雷。効果が大きい |
| 4 | JS の `defer` 化 + サードパーティタグの遅延実行 | TBT（配点30%）の本丸 |
| 5 | クリティカルCSSインライン化 + 残りを遅延 | FCP / LCP の仕上げ |
| 6 | §6 チェックリストで導線確認 → 再計測 | 事故防止 |

**Step 1〜4 の時点でモバイル 80前後、Step 5 まで完了で 90+ が現実的なラインです。**

---

## 9. 補足：スコア以外に気づいた点

- **`/q&a.html` の URL に生の `&` が含まれています。** URLエンコードの問題を起こしやすく、
  SEO・共有・計測の面でも不利です。`/qa.html` 等へのリネーム + 301リダイレクトを推奨します
  （速度とは別件ですが、改修のついでに直す価値があります）。
- **CDN が未導入です。** 福岡ローカルの集客が主なら国内単一サーバーでも実害は小さいですが、
  Cloudflare の無料プランを前段に置くと、画像配信・TTFB・HTTP/3 で底上げできます。
  ただし Xserver の各種キャッシュとの兼ね合いを検証してから導入してください。
- **`/wp/` のブログがほぼ更新されていない**ようです（インデックス上は2021年の1記事のみ）。
  運用しないのであれば、放置された WordPress は**セキュリティリスク**になります。
  更新するか、閉じるかを別途判断してください。

---

## 10. まとめ

1. **確定**: ルートは静的HTMLです。**WP Rocket はトップページのスコアを1点も動かしません。**
   §4（ケースA）で対応します。WP Rocket は `/wp/` のブログにのみ有効です。
2. 90点の鍵は **TBT(30%) + CLS(25%) = 55%**。画像だけでは届きません。
3. 静的HTMLでも **Xserver のサーバー設定 + コード改修**で 90+ は到達可能。
   むしろプラグイン肥大がない分、WordPress より有利です。
4. **モバイル 90+ は 13〜24時間程度の作業**が現実的な見積もりです。
   「設定を数個ONにするだけ」では届きません。
5. スコアより **電話・LINE・フォームの導線が生きていること**が優先。§6 を必ず通してください。
