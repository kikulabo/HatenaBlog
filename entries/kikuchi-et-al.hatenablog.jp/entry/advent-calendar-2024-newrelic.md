---
Title: 「New Relic Browser」で実現するフロントエンドのパフォーマンス改善
Category:
- NewRelic
Date: 2024-12-20T00:00:00+09:00
URL: https://kikuchi-et-al.hatenablog.jp/entry/advent-calendar-2024-newrelic
EditURL: https://blog.hatena.ne.jp/kikuchi_et_al/kikuchi-et-al.hatenablog.jp/atom/entry/6802418398312205623
---

これは [https://qiita.com/advent-calendar/2024/newrelic:title] [シリーズ2] の 20 日目のエントリーです。

今年の技術書典17で実践フロントエンドオブザーバビリティという本を出しました。

[https://techbookfest.org/product/dzpBwetTxzjucgabdEu5TX:embed:cite]

この本の最後の章ではフロントエンドのパフォーマンス計測を行うために、 New Relic Browser を使った計測方法を紹介しています。

本エントリーでは私が開発に関わっているハッカー飯という Web サービスに New Relic Browser を導入して得られた知見を紹介します。

[https://hackermeshi.com/:embed:cite]

# Core Web Vitals

本題に入る前にまずユーザーエクスペリエンスを評価するための指標である Core Web Vitals について説明します。

Core Web Vitals はユーザーがウェブページを訪問した際に感じる「快適さ」を数値で評価するもので、ウェブサイトのパフォーマンスを具体的に改善するための道しるべとなります。

## LCP（Largest Contentful Paint）

>LCP reports the render time of the largest image, text block, or video visible in the viewport, relative to when the user first navigated to the page.

参考: [https://web.dev/articles/lcp:title]

LCP はビューポート内で最大の要素(画像やテキストブロックなど)が表示されるまでの時間です。目標値は 2.5 秒以内でサーバー応答時間の短縮やリソースの最適化によって改善できます。

## INP（Interaction to Next Paint）

>INP is a metric that assesses a page's overall responsiveness to user interactions by observing the latency of all click, tap, and keyboard interactions that occur throughout the lifespan of a user's visit to a page. The final INP value is the longest interaction observed, ignoring outliers.

参考: [https://web.dev/articles/inp:title]

ユーザーのインタラクションに対するページの応答性能を評価したものです。例えばボタンをクリックしたときに画面が反応するまでの速さを測ります。

目標値としては 200 ミリ秒以内が良好とされており、安定してページの応答性が高ければ高いほどユーザーにとって快適な操作感を提供できます。

## CLS（Cumulative Layout Shift）

>CLS is a measure of the largest burst of layout shift scores for every unexpected layout shift that occurs during the entire lifecycle of a page.

参考: [https://web.dev/articles/cls:title]

CLSはページの読み込み中に発生する予期せぬレイアウトのズレの累積スコアを測定したものです。目標値 は 0.1 以下で、画像や広告スペースのサイズを明示的に指定すると改善できます。

# New Relic Browser をインストール

New Relic の管理画面で生成した JavaScript スニペットをフロントエンドのアプリケーション内に埋め込むことで測定が可能になります。

しかし、ハッカー飯には dev、stg、prd の 3 つの環境があるためそれぞれの環境毎に New Relic Browser をインストールする必要がありました。

この問題を解決するために各環境ごとに環境変数を使って設定を行いました。

ハッカー飯ではフロントエンドの開発に Vite を使用していたので .env ファイルを用いて環境変数を定義し、 ビルド時に適用するようにしました。

.env (環境毎に .env ファイルを用意する)

```
VITE_NEW_RELIC_APPLICATION_ID=5555555555
```

JavaScript スニペット

```
NREUM.loader_config = {
  accountID: "0000000",
  trustKey: "1111111",
  agentID: import.meta.env.VITE_NEW_RELIC_APPLICATION_ID,
  licenseKey: "*****",
  applicationID: import.meta.env.VITE_NEW_RELIC_APPLICATION_ID
};
```

同一アカウント内で複数環境の New Relic Browser をインストールしたい場合 agentID と applicationID を変えれば環境を分けることが出来るようです。

（New Relic が生成した JavaScript スニペットを見ると agentID と applicationID は同一のもので良いようです。）

# New Relic Browser を使ってみた

Summary を確認すると LCP と INP は良いスコアを出していることが分かりました。一方 CLS は good になっているものの LCP と INP に比べると改善の余地があることが分かりました。

[f:id:kikuchi_et_al:20241217040758p:plain]

CLS をドリルダウンしていくと `https://hackermeshi.com/profiles/***` というハッカー飯のプロフィールページで特に CLS のスコアが悪いことが分かりました。

[f:id:kikuchi_et_al:20241217040826p:plain]

該当ページにアクセスして実際にレイアウトシフトが発生しているかを確認しました。

この時、ただリロードするだけだと確認し辛かったので Chrome の開発者ツール内にある Network タブの「No throttling」を「3G」に変更してから検証を行いました。

[f:id:kikuchi_et_al:20241217040905p:plain]

これにより通信速度を遅くできるため、レイアウトシフトを目視で確認しやすくなります。

矢印で示した箇所を注目してください。初めは画面の真ん中らへんにあるのですが、その後ページが読み込まれるとレイアウトシフトが発生していることが分かりました。

### レイアウトシフト発生前

[f:id:kikuchi_et_al:20241217040952p:plain]

### レイアウトシフト発生後

[f:id:kikuchi_et_al:20241217041217p:plain]

このように New Relic Browser を使ったことでクライアントがアクセスした各ページ毎のパフォーマンスを可視化することができ、ユーザーエクスペリエンスを向上させるための改善点を見つけることができました。

もう一度 Summary の画面に注目するとフロントエンドでいくらかエラーが発生していることも分かりました。

[f:id:kikuchi_et_al:20241217041252p:plain]

エラーの詳細をクリックすると Session replay の機能でエラーが発生した瞬間の画面を確認することができます。

[f:id:kikuchi_et_al:20241217041313p:plain]

この機能を使うことでエラーが発生した状況を再現しやすくなり、エラーの原因を特定しやすくなりました。

# まとめ

本エントリーではハッカー飯に New Relic Browser を導入し、Core Web Vitals を基にパフォーマンスを評価した経験について紹介しました。

### Core Web Vitalsの重要性

LCP、INP、CLS といった指標がユーザーエクスペリエンスを評価するための具体的な基準となることを紹介しました。それぞれの目標値を意識し、改善の指針とすることでユーザにとって快適なWebサービスを提供できるようになります。

### New Relic Browserの導入手法

環境ごとのデータを分離するため Vite の .env ファイルを活用して設定を行いました。これにより複数の環境での計測データの管理ができるようになりました。

### パフォーマンスデータの可視化と改善

New Relic Browser によってハッカー飯の CLS スコアに問題があるページを特定し Chrome の開発者ツールを活用して実際のレイアウトシフトを確認できました。その結果、具体的な改善点を見つけることができました。

### エラーのトラブルシューティング

New Relic の Session Replay 機能によりエラー発生時の状況を再現しやすくなり、問題解決に役立つ情報を得ることができるようになりました。

また、これらの取り組みによってフロントエンドのパフォーマンス向上だけでなく、開発者の作業効率も向上させることができるようになったと考えています。

今回の知見が皆さんのプロジェクトにおけるパフォーマンス改善の参考になれば幸いです。
