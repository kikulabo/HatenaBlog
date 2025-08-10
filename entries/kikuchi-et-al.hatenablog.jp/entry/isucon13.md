---
Title: ISUCON13 に New Relic を導入して挑んできた
Category:
- NewRelic
Date: 2023-12-02T00:00:00+09:00
URL: https://kikuchi-et-al.hatenablog.jp/entry/isucon13
EditURL: https://blog.hatena.ne.jp/kikuchi_et_al/kikuchi-et-al.hatenablog.jp/atom/entry/6801883189061902451
---

[f:id:kikuchi_et_al:20231126182034p:plain]

※ [New Relic Advent Calendar 2023](https://qiita.com/advent-calendar/2023/newrelic) に投稿した記事です。

こんいす〜！今年も ISUCON に参加してきました！コンテスト中に New Relic を使ってみたので導入してみた感想とか苦労したことを書いてみます。

[https://x.com/isucon_official/status/1728222880922825166:embed]

# ISUCON とは

ISUCON は「**I**ikanjini **S**peed **U**p **Con**test（いい感じに スピードアップ コンテスト）」の略で、お題となるWebサービスを決められたルールの中で高速化を図るチューニングコンテストです。

今年のお題は「ISUPipe」というライブ配信サービスの高速化がテーマになっており、 ISUPipe の投げ銭機能により、送金・寄付された金額がスコアになるといったものでした。

# チーム情報

チーム名: あしたから本気出す

言語: Pyhton

最終スコア: 6,885

# 事前に準備したこと

去年初参加した ISUCON12 では New Relic の導入がうまくいかず 2 時間半ぐらいかかったので、今年は色んな過去問を解き New Relic の導入パターンを学んでおきました。

インフラエージェントはコマンド一発でインストールが可能なのですが、 APM に関してはアプリケーションがコンテナ上で動いているのか pyenv で動いているのかによって微妙に導入方法が変わってくるので、本番で New Relic を使おうという人は事前に導入方法を確認しておくと良いと思います。

# ISUCON13 Python の環境で APM を導入する方法

今年は virtualenv 上で動いていたので、最終的には以下のような手順で APM を導入することが出来ました。

1.仮想環境をアクティベート

```
. .local/share/virtualenvs/python-RZQ_zDKY/bin/activate
```

2.仮想環境上で newrelic ライブラリをインストール

```
pip install newrelic
```

3.newrelic.ini を作成

4.isupipe-python.service を以下のように変更

```
[Unit]
Description=isupipe-python
After=syslog.target
After=mysql.service
Requires=mysql.service

[Service]
WorkingDirectory=/home/isucon/webapp/python
EnvironmentFile=/home/isucon/env.sh
# Environment を追加★
Environment=NEW_RELIC_CONFIG_FILE=/home/isucon/newrelic.ini

User=isucon
Group=isucon
# ExecStart を変更★
# ExecStart=/home/isucon/.x pipenv run python app.py
ExecStart=/home/isucon/.x pipenv run /home/isucon/.local/share/virtualenvs/python-RZQ_zDKY/bin/newrelic-admin run-program python app.py
ExecStop=/bin/kill -s QUIT $MAINPID

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

# 苦労した点

という感じにさらっと書きましたが実際には本番で New Relic の導入に手こずりました。

pipenv が使われてそうだったので `pipenv install newrelic` を実行してインストールしたものの newrelic-admin が見つからないといったエラーが出てしまいました。

その後色々考えたところ手順1で書いたような仮想環境をアクティベートした後に newrelic をインストールしたところ上手くいきました。

あとは Environment に記述した内容が最初絶対パスになっていなかったため newrelic.ini が見つからないというエラーも出てしまい、なんやかんやあって今年は APM の導入に 1 時間ぐらいかかってしまいました。

導入に時間かかっては元も子もないので来年ももし New Relic を導入するならもっと早く APM を入れられるようになりたい…！

# New Relic を使って分析してみた

ベンチマーカーを回して負荷を確認した所 mysql の使用率が高いことは分かっていました。

New Relic の Database の結果を見ると `MySQL users select` の時間が掛かっていることが分かりました。

[f:id:kikuchi_et_al:20231126235928p:plain]

Transactions のページで Throughput 順にソートすると get_icon_handler が多く叩かれていることが判明しました。

[f:id:kikuchi_et_al:20231127000131p:plain]

コードを読んでみると users テーブルから ユーザの ID を取得した後に icons テーブルからユーザ ID に紐付く画像のバイナリーデータを取得していました。

そこで DB に登録されたバイナリーデータを参照するのではなく、ディスク上に保存された画像ファイルを参照するようにすれば users テーブル（と icons テーブル）が叩かれずに済むので負荷が減るのではと考えました。

上記のような修正を行った結果 Most time consuming における users テーブルのランキングを下げることができ、スコアを上昇させることが出来ました！

[f:id:kikuchi_et_al:20231127003937p:plain]

# 感想

今年もなんやかんや導入に手こずってしまいましたが去年よりは早く APM を使える状態に出来て良かったなと思いました。

ISUCON ではしばしばアクセスログの分析には alp 、スロークエリログの分析には pt-query-digest 等が用いられることが多いですが、これは CLI を使って分析する手法なので New Relic よりもこっちの方が手軽に導入することが出来ます。

一方 New Relic はベンチーマーカーを回した結果を GUI 上で可視化することが出来るのでアプリケーション全体を俯瞰して見やすいというのが強みなのかなと考えています。

とは言ったものの今メトリクスを見返してみるともっと他に優先するべき変更点等があったなという気もしていて、まだまだ観察力が足りないなと痛感しました。

今年も ISUCON を楽しんで参加することが出来ました。運営の皆様本当にありがとうございます。

また来年も参加したいと考えているので、次こそは1万点を目標に頑張っていきたいと思います。
