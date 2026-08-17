# Resonite で Youtube 動画が上手く再生できない理由（詳細編）

前回の記事（[X](https://x.com/konto250/status/2089005526151803166)）では対処法だけを書きました。要点は以下です。

- 常用していないブラウザで YouTube にログインし、Resonite の「設定 → ビデオストリーミングサービス → ブラウザの Cookie を使用」をそのブラウザに切り替える
- Chromium 系（Chrome/Edge/Brave/Vivaldi）を指定した場合、Resonite 使用中はそのブラウザを完全に閉じる
- ライブが見られなくなるので、ライブを見るときだけ「NONE」に戻す

ここではログを読んで分かった範囲で、なぜそうなるのかを書きます。

実測で確認したことと、推測にとどまることを区別して書きます。また以下は2026年8月中旬時点の観測です。yt-dlp も YouTube も頻繁に変わるので、そのうち成立しなくなります。

---

## 1. Resonite は yt-dlp を外部プロセスとして呼んでいる

Resonite は YouTube の URL を渡されると、同梱の `yt-dlp.exe` を起動します。ログにコマンドラインが丸ごと出ます。

```
[yt-dlp] Command executed in 3250ms: ...\RuntimeData\yt-dlp.exe
  --js-runtimes undefined --flat-playlist -i -J -s --no-playlist
  --extractor-args "youtube:player_client=default,web_safari"
  --cookies-from-browser vivaldi
  https://www.youtube.com/watch?v=...
```

コマンドラインオプションの細かい説明は避けますが、重要なオプションは `-J -s` です。

- `-J` … 動画情報を JSON で出力する
- `-s` … ダウンロードしない

yt-dlp は取得可能なフォーマットの一覧を返すだけで、どれを再生するかは Resonite が決めています。

このコマンドラインはそのままコピーして手元で実行できます。Resonite を起動せずに挙動を確認できるので、切り分けに使えます。

---

## 2. Resonite は映像と音声が1本になっているフォーマットしか選ばない

Resonite のログには、フォーマットを1つずつ比較していく様子が出ます。

```
Codec avc1.42001E + mp4a.40.2 vs none + mp4a.40.2 ... HasAudio: True,  HasVideo: True
Codec avc1.640028 + none      vs avc1.42001E + none ... HasAudio: False, HasVideo: True
Codec av01.0.12M.08 + none    vs avc1.42001E + none ... HasAudio: False, HasVideo: True
```

`+ none` は音声を持たないという意味です。1080p や 1440p の候補が並んでいても、音声がないものは採用されません。

最終的に選ばれたものは、この行に出ます。

```
Best Format: Video: avc1.42001E, Audio: mp4a.40.2, Size: 640x360, Url: https://...
```

---

## 3. YouTube 側の配信形式は3種類ある

| 形式 | 中身 | 音声 |
|---|---|---|
| progressive | 動画ファイル1本 | 込み。ただし itag 18（360p）だけ |
| HLS（m3u8） | 断片リスト＋断片群 | itag 91〜96 は込み（1080pまで）。提供されないことがある |
| DASH | 映像と音声が別ファイル | 分離。合成が必要 |

360p を超える画質は基本的に DASH で配信されます。Resonite は映像と音声を合成しないので、DASH の 1080p が並んでいても使えません。

Resonite の選択肢は次のようになります。

- 音声つき HLS（itag 91〜96）があれば、それを選ぶ（最大1080p）
- なければ itag 18（360p）だけ

動画によって 360p だったり 1080p だったりするのは、この差によるものです。

---

## 4. Cookie を渡すとクライアントが変わる

yt-dlp は YouTube にアクセスするとき、公式クライアントの種類を指定します。Resonite は `player_client=default,web_safari` を渡していますが、`default` の中身は Cookie の有無で変わります。

| Cookie | 使われるクライアント | URL の `c=` |
|---|---|---|
| なし | visionos + android vr | `c=ANDROID_VR` |
| ログイン済み | tv downgraded | `c=TVHTML5` |

（`c=` は progressive の URL に付くものです。HLS のマニフェスト URL は別系統です）

visionos と android vr はアカウント認証に対応していないため、Cookie を渡すと yt-dlp が候補から外します。明示的に指定しても外されます。

```
WARNING: [youtube] Skipping client "visionos" since it does not support cookies
```

---

## 5. progressive 形式で403が発生する

itag 18 を Cookie なしで15回ダウンロードしたところ、8回が `HTTP Error 403: Forbidden` になりました。同じ動画、同じ URL 形式、同じ時間帯です。

成功時と失敗時で URL を比較すると、`fexp` の値が対応していました。`fexp` は YouTube がリクエストごとに割り当てる実験フラグです。

| fexp の値 | 試行 | 結果 |
|---|---|---|
| `51946837` | 7回 | すべて成功 |
| `51946838` | 8回 | すべて403 |

15回すべてで一致しました。他のパラメータ（`pcm2`、`mt`、`fvip` など）は成否と対応しません。

IPv4 と IPv6 でも比較しましたが、IPv6 で5回中2回、IPv4 で5回中1回の成功でした。差は1回分で試行回数も少ないため、経路による違いがあるとは言えません。少なくとも IPv4 に切り替えれば直るという結果ではありませんでした。

一方、次の2つは失敗していません。

- HLS（m3u8）… 5回中5回成功
- `c=TVHTML5` の itag 18 … CLI で10回中10回、Resonite 実機で8回中8回成功

403 は `c=ANDROID_VR` の progressive に集中しています。

TVHTML5 だと通る理由は分かりません。URL の構造は違っていて、TVHTML5 側には `n` パラメータ（nsig 解読を経たもの）や `siu=1`、`sefc=1` が付きます。この違いが実験の除外条件になっているのではないかと考えていますが、**これは推測です。**確かめていません。

---

## 6. タイトルは出るのに再生されない場合の見分け方

Resonite は 403 をログに記録しません。

再生できたとき。

```
Best Format: ... Size: 640x360 ...
Loading video from: https://rr6---sn-...googlevideo.com/videoplayback?...
Audio tracks:                          ← 0.3〜4秒後に出る
  Audio Track 0, Channels: 2, ...
```

再生できなかったとき。

```
Best Format: ... Size: 640x360 ...
Loading video from: https://rr1---sn-...googlevideo.com/videoplayback?...
（Audio tracks: が出ない。エラーもタイムアウトも記録されない）
```

タイトルは yt-dlp が返すメタデータから取得しているので表示されます。その先の実データ取得が失敗している状態です。Resonite のログには理由が残らないので、同じコマンドラインを手元で実行して確認したところ、403 が返っていました（5章）。

自分の環境がどちらか確認するには、Resonite のインストールフォルダ内の `Logs` から最新のログを開き、`Best Format:` と `Audio tracks:` を検索してください。

---

## 7. ログインするとライブが見られなくなる理由

進行中のライブ配信は HLS でしか配信されません。全体の長さが決まらないため、1本のファイルとして配信できないからです。

HLS を返すのは visionos と android vr です。ログイン Cookie を渡すとこれらが除外されるため、取得できるフォーマットが0件になります。

```
Downloading tv downgraded player API JSON
ERROR: [youtube] ...: No video formats found!
```

通常動画は DASH や progressive が残るので再生できますが、ライブには代替経路がありません。

---

## 8. Chromium 系ブラウザを閉じないといけない理由

Chrome / Edge / Brave / Vivaldi は、起動中に Cookie の SQLite ファイルを排他ロックします。yt-dlp はこのファイルをコピーして読もうとするため、拒否されます。

```
ERROR: Could not copy Chrome cookie database.
PermissionError: [Errno 13] Permission denied:
  '...\Vivaldi\User Data\Default\Network\Cookies'
```

この段階で処理が止まるので、動画が一切再生できません。タスクトレイに常駐している場合も同じです。

Firefox は `cookies.sqlite` を平文で持ち、排他ロックしないため、開いたままでも読めます。

念のため書いておくと、これは Firefox の方が安全という話ではありません。Chrome は 127 以降、Cookie の復号鍵をブラウザ自身に紐づける仕組みを導入しており、外部ツールから読めない場合があります。ここでいう「扱いやすい」は、外部ツールから読めるかどうかという意味だけです。

---

## 9. 試したが効かなかったこと

`RuntimeData\yt-dlp.conf` を置くと yt-dlp 自体には読み込まれます（実機ログで確認）。しかし次の2つは効きませんでした。

- `-f`（フォーマット指定）… `-J` の出力は `-f` の影響を受けず、Resonite は一覧を全部受け取って自前で選ぶため
- `-S`（ソート順）… 並び順は変わるが、選択結果は変わらない。音声なしのフォーマットは順序を変えても採用されない

画質と再生成否を yt-dlp 側から制御する手段は見つかりませんでした。

---

## まとめ

まあまあ面倒くさい現象でした。

---

## 検証環境

- Windows 11 Pro (26200)
- Resonite `2026.8.12.1196`（RESO Launcher 環境、ResoniteModLoader 導入済み）
- 同梱 yt-dlp `nightly@2026.08.04.234419`
- 検証日：2026年8月9日〜17日

Resonite は起動時に同梱の yt-dlp を自動更新します（ログに `Updating yt-dlp` と出ます）。バージョンを固定する手段はありません。

---

## この記事の作り方について

ログの解析と仮説の組み立ては Claude（Opus 5）と対話しながら進めました。文章の草案も Claude に書かせ、こちらで加筆・修正しています。

ただし途中で Claude は誤った結論を何度か出しています。設定を次々に変えながら結果を見る進め方では変数が分離できず、時系列の観測から因果を読もうとして間違えていました。最終的には条件を1つずつ変える対照実験に組み直して確定させています。5章の数字はその結果です。