+++
title = "FAQ: よくある質問とその答え"
weight = 3

[menu.main]
name = "FAQ: よくある質問とその答え"
parent = "learn"
+++

# デザインで定義したバリデーションが実行されるタイミング

サービスのデザインで定義されているバリデーションの実行に関しては、パフォーマンスと堅牢性の間にトレードオフがあります。
Goa はユーザーのコードは正しく実行されるものと信頼し、外部データ（入力）に対してのみバリデーションを実施します。
これは、Goa がサーバ側では入力リクエストを、クライアント側では入力レスポンスをバリデーションすることを意味します。
このように、ユーザーのコードは常に（バリデーションされた）有効なデータを取得することが保証されていますが、サーバー側で書かれているレスポンスやクライアント側で送信されているリクエストごとに、バリデーションの代価を払う必要はありません。

# 生成された構造体のフィールドはどのようなときにポインタになるか？

コード生成アルゴリズムによって生成される構造体のフィールドをポインタにするか、そうしないか、これを決定するために考慮されることがいくつかあります。
目的は、ポインタはコードを複雑にしエラーを生じさせる傾向があるため、必要でないときはポインタの使用を避けることです。
この議論は、基本型を用いるアトリビュート（attribute）にだけ影響します。
オブジェクトのアトリビュートに対応するフィールドは常にポインタが用いられます。
配列やマップのアトリビュートに対応するフィールドには決してポインタは用いられません。

一般的な考え方は、型のデザインで特定のアトリビュートが必須であるかまたはデフォルト値を持つことが指定されている場合、対応するフィールドは nil になることはなく、したがってポインタになる必要はないというものです。
しかしながら、入力 HTTP リクエストもしくはレスポンスをデコードするために生成されたコードは、いくつかのフィールドが（リクエストやレスポンスが不正な形式で）欠落している可能性があるということを考慮する必要があるため、
Unmarshal されたデータの nil 値をテストするために、すべてのフィールドにポインタを使用したデータ構造を使用する必要があります。

以下の表は、ユーザータイプアトリビュートで基本型の生成されたフィールドがポインター (\*) または直接値 (-) のどちらを使用するかを示しています。

* (s) : サーバ用に生成されたデータ構造
* (c) : クライアント用に生成されたコード

| Properties / Data Structure | Payload / Result | Req. Body (s) | Resp. Body (s) | Req. Body (c) | Resp. Body (c) |
------------------------------|------------------|---------------|----------------|---------------|----------------|
| 必須 もしくは デフォルト       | -                | *             | -              | -             | *              |
| 必須でない, デフォルトでない    | *                | *             | *              | *             | *              |

# デフォルト値はどのように使われるか

DSL ではアトリビュートのデフォルト値を指定することが出来ます。
デフォルト値はコードジェネレーターによって2箇所で使われます。

Marshal するコードを生成するとき（サーバ側ならレスポンスの Marshal、クライアント側ならリクエストの Marshal）データ構造体のフィールドが nil ならば初期化にデフォルト値が使われます。
前節で説明したように、アトリビュートが基本型で定義されているならば、フィールドはポインタではないので、これは起こり得ません。
しかし、配列やマップのアトリビュートに対して起こり得ます。

Unmarshal するコードを生成するとき（サーバ側なら入力リクエストの Unmarshal、クライアント側ならレスポンスの Unmarshal）デフォルト値は欠落したフィールドに値をセットするのに使われます。
アトリビュートが必須である場合にそのフィールドが欠落している場合には、生成されたコードはエラーを返すことに注意してください。
すなわち、これはデフォルト値を持つ必須ではないアトリビュートにのみ適用されます。
gRPC エンドポイントの場合、プロトコルバッファ v3 はメッセージをエンコードするときに欠落しているフィールドにデフォルト値（つまり、型のゼロ値）を設定するため、Unmarshal のデフォルト値の動作は異なります。
https://developers.google.com/protocol-buffers/docs/proto3#default を参照してください.
これらのゼロ値がアプリケーション自体によって設定されるのか、プロトコルバッファによって設定されるのかを知ることは不可能です。
したがって、プロトコルバッファメッセージを Unmarshal するときに、メッセージにそのようなフィールドの値がゼロである場合にのみ、Goa はオプションのアトリビュートに対してのみデフォルト値を設定します。
ブール型のオプションアトリビュートは例外です。すなわち、ブール型のアトリビュートがゼロ値（false）の場合、Unmarshal するコードは対応するアトリビュートにデフォルト値を設定しません。
必須かつデフォルト値を持つアトリビュートは、それらがゼロ値の場合は、デフォルト値で設定されません。

# gRPCでオプションアトリビュートはどのように Unmarshal されるか？

プロトコルバッファ v3 はオプショナルと必須のアトリビュートを区別しません（https://developers.google.com/protocol-buffers/docs/proto3#default を参照してください）.
ゼロ値がアプリケーションによって設定されたのかプロトコルバッファによって設定されたのかを知ることは不可能です。
Unmarshal するコードを生成するとき（サーバ側では入力プロトコルバッファリクエストメッセージを Unmarshal するとき、また、クライアント側では入力プロトコルバッファレスポンスメッセージを Unmarshal するとき）対応するプロトコルバッファのメッセージフィールドがゼロ値以外の値である場合にのみ、オプションのアトリビュートは非 nil 値に設定されます。
ブール型のオプショナルなアトリビュートは例外です。すなわち、ブール型のアトリビュートがゼロ値（false）の場合、Unmarshal するコードは対応するアトリビュートをゼロ値に設定します。

# レスポンスタイプのビューはどのように求められるか

レスポンスタイプに対してビューが定義できます。
メソッドがレスポンスタイプを返す場合、

* レスポンスタイプに複数のビューがある場合、サービスメソッドはリザルト（result）とエラーとともに追加のビューを返します。
  生成されたエンドポイント関数はこのビューを使用して、ビューで指定されたレスポンスタイプを作成します。
* ビューのパッケージはサービスレベルで生成され、各メソッドのリザルトに対してビューで指定されたレスポンスタイプを定義します。
  このビューで指定されたレスポンスタイプは同じフィールド名とタイプを持ちますが、ビュー固有の検証ロジックが生成されるようにすべてのフィールドにポインターを使用します。
  レスポンスタイプをビューで指定されたレスポンスタイプに変換したり、その逆の変換を行ったりするために、サービスパッケージ内にコンストラクタが生成されます。

サーバ側のレスポンスを Marshal するコードは、エンドポイントから返されたビューで指定されたレスポンスタイプを、nil アトリビュートを省略したサーバータイプに変換して Marshal します。
レスポンスタイプをレンダリングするために使用されるビューは、"Goa-View" ヘッダーでクライアントに渡されます。
クライアント側のレスポンスを Unmarshal するコードは、レスポンスをクライアントタイプに Unmarshal し、クライアントタイプはビューで指定されたリザルトタイプに変換されます。そして "Goa-View" ヘッダからビューで指定されたレスポンスタイプのビューアトリビュートを設定します。
ビューで定義されているビューで指定されたレスポンスタイプのアトリビュートをバリデーションし、適切なコンストラクタを使用してビューで指定されたレスポンスタイプをサービスのレスポンスタイプに変換します。

>注：レスポンスタイプがビューなしで定義されている場合、 "default" ビューが Goa によってレスポンスタイプに追加されます。
>ビューを気にしないのであれば、ビュー固有のロジックをバイパスする `Type` DSL を使ってメソッドのリザルトを定義できます。
