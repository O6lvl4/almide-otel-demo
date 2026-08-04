# almide-otel-demo

English: [README.md](./README.md)

純粋な [Almide](https://github.com/almide/almide) で書いた、ゼロからのOpenTelemetryトレーシングSDK（4モジュール・約166行）と、Jaegerに1本の分散トレースを表示する2サービスdemo。

OpenTelemetry CollectorもSDK依存もありません。spanをOTLP/HTTP + JSONでエンコードして、JaegerのネイティブOTLPエンドポイントに直接POSTします。コンテキストはW3C `traceparent` ヘッダでHTTP境界を越えます。

## 得られるもの

2プロセス・4 span・1トレース：

```
almide-checkout    checkout         50.3ms   root
almide-checkout    GET /inventory   37.7ms   └─ クライアントspan
almide-inventory   GET /inventory   30.1ms      └─ サーバspan（traceparentで接続）
almide-inventory   SELECT items     30.1ms         └─ DB作業のシミュレーション
```

属性は「全部文字列」ではなく、proto3の本来の型のまま届きます：

```
order.id         = "A-1042"  (string)
order.items      = 2         (int64)
order.total      = 38.5      (float64)
order.express    = true      (bool)
```

そして失敗した呼び出しは失敗したspanになります。inventoryサービスを止めてcheckoutだけ動かすと：

```
GET /inventory   otel.status_code        = ERROR
                 otel.status_description = connection failed: Connection refused (os error 61)
```

## 動かす

必要なのはDockerと、**v0.53.2以降**のalmideです：

```bash
curl -fsSL https://raw.githubusercontent.com/almide/almide/main/tools/install.sh | sh
```

その後：

```bash
docker run -d --name jaeger -p 16686:16686 -p 4318:4318 jaegertracing/all-in-one:latest

almide run service_b.almd   # ターミナル1: inventoryサービス（:7002）
almide run service_a.almd   # ターミナル2: checkout — trace idを出力
```

http://localhost:16686 を開いて `almide-checkout` サービスを選ぶと、ウォーターフォールが表示されます。

`minimal.almd` は1ファイルの出発点です：span 1本、SDKなし——OTLPのワイヤをそれを鏡映するCodec型として宣言し、導出encodeがそのままexporterになります。

## テストを動かす

エンコーダは純粋関数なので、コレクタもソケットも時計もなしにワイヤを検証できます。Dockerは不要です：

```bash
almide test          # 36件
```

トレーシングSDKが静かに間違えがちなところを固定してあります：oneofの各armがキーを**ちょうど1つ**だけ出すこと、rootのspanには`parentSpanId`が**そもそも存在しない**こと、statusのmessageはerror armにしか現れないこと、壊れた`traceparent`は**修復せず拒否**すること、そしてOTLPドキュメント1件のバイト単位の一致。

## 構成

SDKは4モジュール・166行で、**責務**で分けてあります。うち3つは純粋——時計もソケットも乱数も触りません——だからこそ36件中31件が単なる値の比較で済みます。

```
span ← w3c
span ← otlp ← otel
```

| モジュール | 責務 | 副作用 |
|---|---|---|
| `src/span.almd` | spanとは何か：kind/value/statusのmatrixと、span自身の動詞 | なし |
| `src/w3c.almd` | W3C Trace Contextヘッダのinject/parse | なし |
| `src/otlp.almd` | ワイヤ：proto3ミラー型と、ドメイン→ワイヤの橋渡し | なし |
| `src/otel.almd` | tracer：時計、ID生成、ソケット | **あり** |

| プログラム | |
|---|---|
| `service_a.almd` | checkout — rootとクライアントspanを記録し、外向きリクエストに `traceparent` を注入 |
| `service_b.almd` | inventory — `traceparent` を抽出し、サーバspanとDB子spanを記録 |
| `minimal.almd` | Jaegerにspanを1本出す最小のプログラム。SDKなし、`src/` からのimportもなし |

サービス側がimportするのは、tracerのための `otel`、spanの動詞のための `span`、ヘッダのための `w3c` の3つだけです。`otlp` はimportしません——ワイヤ形式は `export` の実装詳細であり、その関数の外では誰も名前を呼びません。

## 仕組み

**ワイヤは型である。** OTLPドキュメントは、フィールド名がそのままワイヤ名（`traceId`、`startTimeUnixNano`）であるCodec型として宣言され、導出encodeが写像レイヤなしにドキュメントを出力します。`none`のOptionフィールドはキーごと省略——まさにproto3の"unset"——なので、rootのspanには`parentSpanId`がそもそも存在しません（almideのencode省略則、[C-209](https://github.com/almide/almide/blob/main/docs/contracts/contracts.toml)）。

**oneofは直和型である。** proto3の`AnyValue`はoneofなので、SDK側では `Str | Int64 | Bool | Double` になります——構築の時点でarmはちょうど1つです。ワイヤ側ではall-Optionのミラー型になり、生き残るキーもちょうど1つ。両者を繋ぐのは`match`ひとつだけで、oneofの規律は実行時チェックではなくその`match`そのものです。

**statusは完全なmatrix。** `Unset | Success | Failure(String)` がproto3のstatus code 3値すべてを覆い、messageを持つのはfailure armだけです——`message`が意味を持つのは`code = 2`のときだけ、というプロトコルの規定そのままです。外向きリクエストの`Result`をunwrapせず`match`しているので、失敗した呼び出しも「理由を語る、完了してexportされたspan」になります。

**タイムスタンプ。** `datetime.now()` は秒精度で、spanのdurationには使いものになりません。SDKはtracer生成時にwall clockを一度だけアンカーし、以後のすべてのタイムスタンプを `datetime.monotonic_ns()` から導出します — 本物のOTel SDKと同じ手法です。OTLPのfixed64ナノ秒フィールドは、proto3 JSONマッピングの規定によりJSONでは*文字列*になります。

**伝播。** クライアントspanは `00-{trace_id}-{span_id}-01` にシリアライズされ、`traceparent` ヘッダとしてリクエストに乗ります。サーバ側は `http.req_header(req, "traceparent")` で読み戻し、そこに自分のspanをぶら下げます。ヘッダが無い場合**も**壊れている場合も `none` になり、呼び出し元のトレースを汚さずに新しいトレースを開始します——`parse_traceparent` のチェックはすべてW3C勧告のMUSTで、仕様が無効と定める全ゼロのIDも含みます。

**エクスポート。** バッチごとに `http://localhost:4318/v1/traces` へPOST 1回。レスポンスが `{"partialSuccess":{}}` なら全部受理されています。

## API

シグネチャはひとつの規則で決まります：**生成する**関数はtracerを先に取り、span を**変換する**関数はspanを先に取る。だからspanの一生はパイプ1本で書けて、最後に時計を刻む唯一の副作用ステップで終わります。

```almide
let checkout = otel.root(t, "checkout")!
  |> span.attr("order.id", span.Str("A-1042"))
  |> span.attr("order.total", span.Double(38.5))
  |> span.succeed
  |> otel.finish(t)!
```

このチェーンのモジュール接頭辞はノイズではなく、各ステップが**どの層に属すか**を語っています。`span.` と書かれているものはすべて純粋な値の変換で、時計に触れるのは `otel.` の2つだけです。

入口は `root` と `child` の2つです。`child` は `Option` ではなく `SpanCtx` を取るので、「親のいない子span」はそもそも書けません。`Option[SpanCtx]` を取る生の `start` は、親が本当に存在しないかもしれない唯一のケース——受信側のサーバハンドラ——のためにあります。

## 経緯

最初の版を書いた過程でalmideコンパイラのバグが6件見つかり（[#1049–#1054](https://github.com/almide/almide/issues/1049)）、翌日までにすべて修正されました。`service_b.almd` の型付きハンドラシグネチャは、その修正バッチの産物です。

モジュール分割の過程でさらに6件見つかり、[#1087–#1092](https://github.com/almide/almide/issues/1087) として登録しました。**6件とも v0.53.2 で修正済み**で、このリポジトリは回避策ではなく修正後のコンパイラに合わせて書いてあります：

| かつての制約 | 今のコード |
|---|---|
| [#1087](https://github.com/almide/almide/issues/1087) convention methodがimport越しに到達不能 | `p.encode()`・`x.repr()`・`[T: Codec]` がすべて解決。`SpanKind` が `repr` をオーバーライドしているので `"${kind}"` はどのモジュールでも `client` を出す |
| [#1088](https://github.com/almide/almide/issues/1088) デフォルト引数がexport署名から落ちる | `otel.tracer("almide-checkout")` と `otel.root(t, "checkout")` がデフォルトを受け取る |
| [#1089](https://github.com/almide/almide/issues/1089) `json.encode` がimport越しに存在しない | エンコードはどこでも1呼び出し |
| [#1090](https://github.com/almide/almide/issues/1090) `almide fmt` がレコード内コメントを削除 | フィールドのコメントがformatを生き延びる |
| [#1091](https://github.com/almide/almide/issues/1091) 複数行メソッドチェーンが構文エラー | チェーンはここでは読みやすい `\|>` のままだが、両形式とも動く |
| [#1092](https://github.com/almide/almide/issues/1092) コンパイルできないドキュメント例 | — |

未解決の形がひとつだけ残っていて、効いている箇所にはコードにそう書いてあります：`otlp` のspanミラーが `Span` ではなく `WireSpan` なのは、ドメイン側のモジュール名が `span` で型名が `Span` だからです——両方をbareで綴ると、あのファイル内の参照が誤った方に解決されます（[#1093](https://github.com/almide/almide/issues/1093)、[#1094](https://github.com/almide/almide/issues/1094)）。OTLPミラーの他の型はワイヤ本来の名前のままです。

## ライセンス

MIT
