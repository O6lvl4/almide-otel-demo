# almide-otel-demo

English: [README.md](./README.md)

純粋な [Almide](https://github.com/almide/almide) で書いた、ゼロからのOpenTelemetryトレーシングSDK（約150行）と、Jaegerに1本の分散トレースを表示する2サービスdemo。

OpenTelemetry CollectorもSDK依存もありません。spanをOTLP/HTTP + JSONでエンコードして、JaegerのネイティブOTLPエンドポイントに直接POSTします。コンテキストはW3C `traceparent` ヘッダでHTTP境界を越えます。

## 得られるもの

2プロセス・4 span・1トレース：

```
almide-checkout    checkout         59.2ms   root
almide-checkout    GET /inventory   47.8ms   └─ クライアントspan
almide-inventory   GET /inventory   29.8ms      └─ サーバspan（traceparentで接続）
almide-inventory   SELECT items     29.8ms         └─ DB作業のシミュレーション
```

## 動かす

必要なのはDockerと、v0.50.0より新しいalmideです — `service_b.almd` の型付き `HttpRequest` ハンドラは `develop` 上の修正バッチに依存していて、タグ付きリリースにはまだ入っていません。次のリリースまでは：

```bash
git clone https://github.com/almide/almide && cd almide && make install
```

その後：

```bash
docker run -d --name jaeger -p 16686:16686 -p 4318:4318 jaegertracing/all-in-one:latest

almide run service_b.almd   # ターミナル1: inventoryサービス（:7002）
almide run service_a.almd   # ターミナル2: checkout — trace idを出力
```

http://localhost:16686 を開いて `almide-checkout` サービスを選ぶと、ウォーターフォールが表示されます。

`minimal.almd` は1ファイルの出発点です：span 1本、SDKなし、生のOTLP JSONの形。

## 構成

| ファイル | |
|---|---|
| `src/otel.almd` | SDK本体：Tracer、span、ID生成、OTLP/HTTP JSONエクスポート、traceparentのinject/parse |
| `service_a.almd` | checkout — rootとクライアントspanを記録し、外向きリクエストに `traceparent` を注入 |
| `service_b.almd` | inventory — `traceparent` を抽出し、サーバspanとDB子spanを記録 |
| `minimal.almd` | Jaegerにspanを1本出す最小のプログラム |

## 仕組み

**タイムスタンプ。** `datetime.now()` は秒精度で、spanのdurationには使いものになりません。SDKはtracer生成時にwall clockを一度だけアンカーし、以後のすべてのタイムスタンプを `datetime.monotonic_ns()` から導出します — 本物のOTel SDKと同じ手法です。OTLPのfixed64ナノ秒フィールドは、proto3 JSONマッピングの規定によりJSONでは*文字列*になります。

**伝播。** クライアントspanは `00-{trace_id}-{span_id}-01` にシリアライズされ、`traceparent` ヘッダとしてリクエストに乗ります。サーバ側は `http.req_header(req, "traceparent")` で読み戻し、そこに自分のspanをぶら下げます。この1本のヘッダが、2つのプロセスを1つのトレースに繋ぎます。

**エクスポート。** バッチごとに `http://localhost:4318/v1/traces` へPOST 1回。レスポンスが `{"partialSuccess":{}}` なら全部受理されています。

## 経緯

この実装の過程でalmideコンパイラのバグが6件見つかり（[#1049–#1054](https://github.com/almide/almide/issues/1049)）、翌日までにすべて修正されました。`service_b.almd` の型付きハンドラシグネチャは、その修正バッチの産物です。

## ライセンス

MIT
