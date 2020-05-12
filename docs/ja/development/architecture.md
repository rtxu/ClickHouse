---
machine_translated: true
machine_translated_rev: d734a8e46ddd7465886ba4133bff743c55190626
toc_priority: 62
toc_title: "\u30AF\u30EA\u30C3\u30AF\u30CF\u30A6\u30B9\u306E\u6982\u8981"
---

# クリックハウスの概要 {#overview-of-clickhouse-architecture}

ClickHouseは真の列指向DBMSです。 データは、列によって、および配列（ベクトルまたは列のチャンク）の実行中に格納されます。 可能な限り、操作は個々の値ではなく、配列にディスパッチされます。 それは呼ばれます “vectorized query execution,” そしてそれは実際のデータ処理の費用を下げるのを助けます。

> この考え方は、新しいものではない。 それはに遡ります `APL` プログラミング言語とその子孫: `A +`, `J`, `K`、と `Q`. 配列プログラミングは科学的データ処理に使用されます。 このアイデアは、リレーショナルデータベースで新しいものでもありません。 `Vectorwise` システム。

クエリ処理の高速化には、vectorizedクエリの実行とランタイムコードの生成という二つのアプローチがあります。 後者は、すべての間接指定と動的ディスパッチを削除します。 ずれのアプローチは厳重によります。 ランタイムコードを生成するときにヒューズが多く、業務として活用cpuの実行単位のパイプライン vectorizedクエリを実行できる実用的では一時的ベクトルを明記のことは、キャッシュを読みます。 一時的なデータがl2キャッシュに収まらない場合、これが問題になります。 しかし、ベクトル化されたクエリの実行は、cpuのsimd機能をより簡単に利用します。 a [研究論文](http://15721.courses.cs.cmu.edu/spring2016/papers/p5-sompolski.pdf) 書面による友人達しないことであり、両アプローチ。 ClickHouse用vectorizedクエリを実行して初期支援のためにランタイムコード。

## 列 {#columns}

`IColumn` interfaceは、メモリ内の列（実際には列のチャンク）を表すために使用されます。 このインタフェースのヘルパーの方法の実施のための様々な関係です。 ほとんどすべての操作は不変です：元の列は変更されませんが、新しい変更された列を作成します。 たとえば、 `IColumn :: filter` 法を受け入れフィルタのバイトマスクです。 それはのために使用されます `WHERE` と `HAVING` 関係演算子。 その他の例： `IColumn :: permute` サポートする方法 `ORDER BY`、を `IColumn :: cut` サポートする方法 `LIMIT`.

様々な `IColumn` 実装 (`ColumnUInt8`, `ColumnString`、というように）列のメモリレイアウトを担当しています。 メモリレイアウトは、通常、連続した配列です。 列の整数型の場合、次のような連続した配列にすぎません `std :: vector`. のために `String` と `Array` すべての配列要素のベクトル、連続して配置されたベクトル、および各配列の先頭にオフセットするためのベクトルです。 また、 `ColumnConst` これはメモリに一つの値だけを格納しますが、列のように見えます。

## フィールド {#field}

それにもかかわらず、個々の価値を扱うことも可能です。 個々の値を表すには、 `Field` 使用される。 `Field` のちょうど差別された組合である `UInt64`, `Int64`, `Float64`, `String` と `Array`. `IColumn` は、 `operator[]` n番目の値をaとして取得するメソッド `Field` そして `insert` aを追加するメソッド `Field` 列の終わりまで。 これらの方法はあまり効率的ではありません。 `Field` 個々の値を表すオブジェクト。 より効率的な方法があります。 `insertFrom`, `insertRangeFrom`、というように。

`Field` テーブルの特定のデータ型に関する十分な情報がありません。 例えば, `UInt8`, `UInt16`, `UInt32`、と `UInt64` すべてとして表されます `UInt64` で `Field`.

## 漏れやすい抽象化 {#leaky-abstractions}

`IColumn` は方法のための共通の関係変容のデータもあるんですが、そのすべて満たす。 例えば, `ColumnUInt64` 二つの列の合計を計算する方法を持っていない、と `ColumnString` 部分文字列検索を実行するメソッドはありません。 これらの無数のルーチンは、 `IColumn`.

列のさまざまな関数は、次のように一般的で効率的でない方法で実装できます `IColumn` 抽出する方法 `Field` 値、または特定のデータの内部メモリレイアウトの知識を使用して、特殊な方法で `IColumn` 実装。 これは、関数を特定の `IColumn` 内部表現を直接入力して処理する。 例えば, `ColumnUInt64` は、 `getData` 内部配列への参照を返し、別のルーチンがその配列を直接読み取ったり塗りつぶしたりするメソッドです。 我々は持っている “leaky abstractions” さまざまなルーチンの効率的な専門化を可能にする。

## データ型 {#data_types}

`IDataType` バイナリ形式またはテキスト形式で列または個々の値のチャンクを読み書きするためのシリアル化と逆シリアル化を担当します。 `IDataType` テーブルのデータ型に直接対応します。 たとえば、次のとおりです `DataTypeUInt32`, `DataTypeDateTime`, `DataTypeString` というように。

`IDataType` と `IColumn` 互いにゆるやかに関係しているだけです 異なるデータ型は同じでメモリ内に表すことができます `IColumn` 実装。 例えば, `DataTypeUInt32` と `DataTypeDateTime` どちらも `ColumnUInt32` または `ColumnConstUInt32`. また、同じデータ型を異なるデータ型で表すこともできます `IColumn` 実装。 例えば, `DataTypeUInt8` で表すことができます `ColumnUInt8` または `ColumnConstUInt8`.

`IDataType` 貨物のメタデータを指すものとします。 例えば, `DataTypeUInt8` 何も全く保存しません(vptrを除きます)。 `DataTypeFixedString` 店だけ `N` （固定サイズの文字列のサイズ）。

`IDataType` はヘルパーの方法のための様々なデータフォーマット たとえば、引用符で値をシリアル化したり、JSONの値をシリアル化したり、XML形式の一部として値をシリアル化したりするメソッドがあります。 データ形式への直接の対応はありません。 たとえば、さまざまなデータ形式 `Pretty` と `TabSeparated` 同じを使用できます `serializeTextEscaped` からヘルパーメソッド `IDataType` インタフェース

## ブロック {#block}

A `Block` メモリ内のテーブルのサブセット(チャンク)を表すコンテナです。 それはちょうどトリプルのセットです: `(IColumn, IDataType, column name)`. クエリの実行中、データは次の方法で処理されます `Block`我々が持っている場合 `Block`、我々はデータを持っている（で `IColumn` オブジェクト）、そのタイプに関する情報があります `IDataType`）それはその列をどう扱うかを教えてくれるので、列名があります。 テーブルの元の列名か、計算の一時的な結果を得るために割り当てられた人工的な名前のいずれかになります。

ブロック内の列に対していくつかの関数を計算すると、その結果を持つ別の列がブロックに追加され、関数の引数の列には触れません。 その後、不要な列はブロックから削除できますが、変更はできません。 それは共通の部分式の除去のために便利です。

ブロックの作成のための各処理チャンクのデータです。 同じタイプの計算では、列名とタイプは異なるブロックで同じままで、列データのみが変更されることに注意してください。 ブロックヘッダーからブロックデータを分割する方が良いのは、小さなブロックサイズでは、shared\_ptrsと列名をコピーするための一時文字列のオーバーヘッドが

## ブロックの流れ {#block-streams}

ブロッ を使用していま流のブロックからデータを読み込むためのどこかに、データ変換、または書き込みデータをどこかということです。 `IBlockInputStream` は、 `read` 利用可能な間に次のブロックを取得するメソッドです。 `IBlockOutputStream` は、 `write` ブロックをどこかにプッシュする方法。

ストリー:

1.  テーブルへの読み書き。 のテーブルだけを返しますストリームを読み取りまたは書き込みブロックとなります。
2.  データ形式の実装。 たとえば、データを端末に出力する場合は、 `Pretty` ブロックをプッシュするブロック出力ストリームを作成し、それらをフォーマットします。
3.  データ変換の実行。 たとえば、 `IBlockInputStream` いを作ろうというストリームです。 あなたが作成する `FilterBlockInputStream` ストリームで初期化します。 その後、ブロックを引き出すと `FilterBlockInputStream` で引きブロックからストリーム、フィルタでは、フィルタを返しますブロックします。 クエリの実行パイプラインで表現しました。

より洗練された変換があります。 たとえば、あなたから引くとき `AggregatingBlockInputStream` ソースからすべてのデータを読み取り、集計してから、集計データのストリームを返します。 別の例: `UnionBlockInputStream` コンストラクタ内の多数の入力ソースと、多数のスレッドを受け入れます。 複数のスレッドを起動し、複数のソースから並行して読み取ります。

> ブロックストリームは、 “pull” 制御フローへのアプローチ：最初のストリームからブロックをプルすると、ネストされたストリームから必要なブロックがプルされ、実行パイプライン全体 どちらも “pull” また “push” 制御フローは暗黙的であり、複数のクエリの同時実行（多くのパイプラインを一緒にマージする）などのさまざまな機能の実装を制限するため、最適なソ この制限は、コルーチンや、お互いを待つ余分なスレッドを実行するだけで克服することができます。 ある計算単位から別の計算単位にデータを渡すためのロジックを特定すると、制御フローを明示的にすると、より多くの可能性があります。 これを読む [記事](http://journal.stuffwithstuff.com/2013/01/13/iteration-inside-and-out/) より多くの思考のために。

クエリ実行パイプラインは、各ステップで一時的なデータを作成します。 ブロックサイズを十分に小さくして、一時的なデータがcpuキャッシュに収まるようにします。 その仮定では、一時的なデータの書き込みと読み取りは、他の計算と比較してほとんど無料です。 したことを検討してはどうかと思うの代替、ヒューズの多くの業務のパイプラインです。 パイプラインをできるだけ短くし、一時的なデータの多くを削除することができますが、これは利点かもしれませんが、欠点もあります。 たとえば、分割パイプラインを使用すると、中間データのキャッシュ、同時に実行される類似クエリからの中間データの盗み取り、類似クエリのパイプラ

## 形式 {#formats}

データフォーマットにて実施しブロックわれている。 あります “presentational” 次のように、クライアントへのデータの出力にのみ適した書式を設定します `Pretty` のみを提供する形式 `IBlockOutputStream`. また、次のような入力/出力形式があります `TabSeparated` または `JSONEachRow`.

行ストリームもあります: `IRowInputStream` と `IRowOutputStream`. ブロックではなく、個々の行でデータをプル/プッシュすることができます。 また、行指向のフォーマットの実装を単純化するためにのみ必要です。 その他の機能 `BlockInputStreamFromRowInputStream` と `BlockOutputStreamFromRowOutputStream` 行指向のストリームを通常のブロック指向のストリームに変換できます。

## I/O {#io}

バイト指向の入出力には、以下のものがあります `ReadBuffer` と `WriteBuffer` 抽象クラス。 それらはC++の代わりに使用されます `iostream`すべての成熟したC++プロジェクトは、 `iostream`良い理由のためのs。

`ReadBuffer` と `WriteBuffer` 単に連続したバッファであり、そのバッファ内の位置を指すカーソルです。 実装にはない独自のメモリにバッファです。 バッファに次のデータを入力するための仮想メソッドがあります `ReadBuffer` または、バッファをどこかにフラッシュする（ `WriteBuffer`). 仮想手法が呼び出されます。

の実装 `ReadBuffer`/`WriteBuffer` 使用ファイルを操作するため、ファイル記述子およびネットワークソケット実施のための圧縮 (`CompressedWriteBuffer` is initialized with another WriteBuffer and performs compression before writing data to it), and for other purposes – the names `ConcatReadBuffer`, `LimitReadBuffer`、と `HashingWriteBuffer` 自分のために話す。

Read/WriteBuffersはバイトのみを処理します。 からの機能があります `ReadHelpers` と `WriteHelpers` 入力/出力の書式設定に役立つヘッダーファイル。 たとえば、decimal形式で数値を書くヘルパーがあります。

結果セットを書きたいときに何が起こるかを見てみましょう `JSON` 標準出力にフォーマットします。 あなたは結果セットをフェッチする準備ができています `IBlockInputStream`. あなたが作成する `WriteBufferFromFileDescriptor(STDOUT_FILENO)` stdoutにバイトを書き込む。 あなたが作成する `JSONRowOutputStream`、それで初期化 `WriteBuffer`、行を書き込む `JSON` stdoutに。 あなたが作成する `BlockOutputStreamFromRowOutputStream` その上に、としてそれを表すために `IBlockOutputStream`. その後、呼び出す `copyData` からデータを転送するには `IBlockInputStream` に `IBlockOutputStream`、そしてすべてが動作します。 内部的には, `JSONRowOutputStream` さまざまなJSON区切り文字を書き込み、 `IDataType::serializeTextJSON` への参照を持つメソッド `IColumn` そして、引数として行番号。 その結果, `IDataType::serializeTextJSON` からメソッドを呼び出します `WriteHelpers.h`：例えば, `writeText` 数値型の場合 `writeJSONString` のために `DataTypeString`.

## テーブル {#tables}

その `IStorage` インタフェースです。 異なる実装のインタフェースの異なるテーブルエンジンです。 例としては `StorageMergeTree`, `StorageMemory`、というように。 これらのクラスのインスタ

キー `IStorage` 方法は `read` と `write`. また、 `alter`, `rename`, `drop`、というように。 その `read` メソッドは次の引数を受け取ります:テーブルから読み取る列のセット。 `AST` 検討するクエリ、および返すストリームの必要数。 一つまたは複数を返します `IBlockInputStream` クエリの実行中にテーブルエンジン内で完了したデータ処理のステージに関するオブジェクトと情報。

ほとんどの場合、readメソッドは、それ以降のデータ処理ではなく、テーブルから指定された列を読み取るだけです。 すべてのデータ処理が行われるクエリの通訳や外部の責任 `IStorage`.

しかし、顕著な例外があります:

-   ASTクエリは、 `read` 法により処理し、テーブルエンジンを使用できる指の利用と読みの少ないデータを表示します。
-   時々のテーブルエンジンを処理できるデータそのものである場合でも特定の段階にある。 例えば, `StorageDistributed` クエリをリモートサーバーに送信し、異なるリモートサーバーのデータをマージできるステージにデータを処理するように要求し、その前処理されたデータを返すことが クエリの通訳を仕上げ加工のデータです。

テーブルの `read` メソッドは、複数の `IBlockInputStream` 並列データ処理を可能にするオブジェクト。 これらの複数のブロックの入力ストリームでテーブルから行なった。 次に、これらのストリームを、独立して計算して作成できるさまざまな変換(式の評価やフィルター処理など)でラップできます。 `UnionBlockInputStream` それらの上に、並行して複数のストリームから読み取ります。

また、 `TableFunction`s.これらは一時的な関数を返す関数です。 `IStorage` で使用するオブジェクト `FROM` クエリの句。

テーブルエンジンを実装する方法を簡単に知るには、次のような単純なものを見てください `StorageMemory` または `StorageTinyLog`.

> の結果として `read` 方法, `IStorage` を返します `QueryProcessingStage` – information about what parts of the query were already calculated inside storage.

## パーサー {#parsers}

手書きの再帰的降下パーサーは、クエリを解析します。 例えば, `ParserSelectQuery` クエリのさまざまな部分の基礎となるパーサーを再帰的に呼び出すだけです。 パーサーは `AST`. その `AST` ノードによって表されます。 `IAST`.

> パーサジェネレータは、使用しない歴史的な理由があります。

## 通訳者 {#interpreters}

クエリ実行パ `AST`. 以下のような単純な通訳があります `InterpreterExistsQuery` と `InterpreterDropQuery` または、より洗練された `InterpreterSelectQuery`. クエリの実行パイプラインの組み合わせたブロック入力または出力ストリーム. たとえば、次のように解釈されます。 `SELECT` クエリは、 `IBlockInputStream` 結果セットを読み取るには、INSERTクエリの結果は次のようになります。 `IBlockOutputStream` に挿入するためのデータを書き込むには、 `INSERT SELECT` クエリは、 `IBlockInputStream` これは、最初の読み取りで空の結果セットを返しますが、データをコピーします `SELECT` に `INSERT` 同時に。

`InterpreterSelectQuery` 使用 `ExpressionAnalyzer` と `ExpressionActions` クエリ分析と変換のための機械。 これは、ほとんどのルールベースのクエリ最適化が行われる場所です。 `ExpressionAnalyzer` モジュラー変換やクエリを可能にするために、さまざまなクエリ変換と最適化を別々のクラスに抽出する必要があります。

## 機能 {#functions}

通常の関数と集約関数があります。 集計関数は、次のセクションを参照してください。

Ordinary functions don’t change the number of rows – they work as if they are processing each row independently. In fact, functions are not called for individual rows, but for `Block`ベクトル化されたクエリ実行を実装するためのデータです。

いくつかの雑多な機能があります [ブロックサイズ](../sql-reference/functions/other-functions.md#function-blocksize), [rowNumberInBlock](../sql-reference/functions/other-functions.md#function-rownumberinblock)、と [runningAccumulate](../sql-reference/functions/other-functions.md#function-runningaccumulate)、それはブロック処理を悪用し、行の独立性に違反します。

ClickHouseには厳密な型指定があるため、暗黙の型変換はありません。 関数が型の特定の組み合わせをサポートしていない場合は、例外がスローされます。 ものの機能で作業する過負荷のもとに多くの異なる組み合わせます。 たとえば、 `plus` 機能(実装するには `+` 演算子）数値型の任意の組み合わせに対して機能します: `UInt8` + `Float32`, `UInt16` + `Int8`、というように。 また、いくつかの可変個引数関数は、任意の数の引数を受け入れることができます。 `concat` 機能。

実施の機能が少し不便での機能を明示的に派遣サポートされているデータの種類と対応 `IColumns`. たとえば、 `plus` 関数は、数値型の各組み合わせのためのC++テンプレートのインスタンス化によって生成されたコードを持っており、定数または非定数左右引数。

優れた場所をランタイムコード生成を避けるテンプレートコードで膨張. また、fused multiply-addようなfused関数を追加したり、あるループ反復で多重比較を行うこともできます。

ベクトル化されたクエリの実行により、関数は短絡されません。 たとえば、次のように書くと `WHERE f(x) AND g(y)`、両側は、行に対しても計算されます。 `f(x)` はゼロです(ただし、 `f(x)` はゼロ定数式である。 しかしの選択率 `f(x)` 条件が高い、との計算 `f(x)` よりもはるかに安いです `g(y)`、それはマルチパス計算を実装する方が良いでしょう。 それは最初に計算します `f(x)`、その後、結果によって列をフィルタリングし、計算 `g(y)` フィルター処理された小さいデータチャンクのみ。

## 集計関数 {#aggregate-functions}

集計関数はステートフル関数です。 渡された値をある状態に蓄積し、その状態から結果を得ることができます。 それらはと管理されます `IAggregateFunction` インタフェース 状態はかなり単純なものにすることができます `AggregateFunctionCount` 単なる `UInt64` 値)または非常に複雑な(の状態 `AggregateFunctionUniqCombined` は、線形配列、ハッシュテーブル、および `HyperLogLog` 確率的データ構造）。

状態は `Arena` 高カーディナリティを実行しながら複数の状態を処理する（メモリプール） `GROUP BY` クエリ。 たとえば、複雑な集計状態では、追加のメモリ自体を割り当てることができます。 それは、州を創設し、破壊し、所有権と破壊秩序を適切に渡すことにある程度の注意を払う必要があります。

分散クエリの実行中にネットワーク経由で渡すか、十分なramがないディスクに書き込むために、集約状態をシリアル化および逆シリアル化できます。 それらはテーブルで貯えることができます `DataTypeAggregateFunction` データの増分集計を可能にする。

> 集計関数状態のシリアル化されたデータ形式は、現在バージョン化されていません。 集約状態が一時的にのみ格納されている場合は問題ありません。 しかし、我々は持っている `AggregatingMergeTree` テーブルエンジンが増えた場合の集約、人々に基づき使用されている。 これは、将来の集約関数の直列化形式を変更する際に下位互換性が必要な理由です。

## サーバ {#server}

サーバを実装し複数の複数のインタフェース:

-   外部クライアントのhttpインターフェイス。
-   ネイティブclickhouseクライアント用のtcpインターフェイス、および分散クエリ実行中のサーバー間通信。
-   インターフェース転送データレプリケーション.

内部的には、コルーチンやファイバーのない原始的なマルチスレッドサーバーです。 サーバーは、単純なクエリの高いレートを処理するが、複雑なクエリの比較的低いレートを処理するように設計されていないので、それらのそれぞれは、分析

サーバーは、 `Context` クエリ実行に必要な環境を持つクラス:利用可能なデータベース、ユーザーおよびアクセス権、設定、クラスター、プロセスリスト、クエリログなどのリスト。 通訳者はこの環境を使用します。

古いクライアントは新しいサーバーと通信でき、新しいクライアントは古いサーバーと通信できます。 しかし、私たちは永遠にそれを維持したくない、と私たちは約一年後に古いバージョンのサポートを削除しています。

!!! note "メモ"
    ほとんどの外部アプリケーションの利用を推奨します。http面でシンプルで使いやすいです。 tcpプロトコルは、内部データ構造により緊密にリンクされています。 そのプロトコルのcライブラリは、実用的ではないclickhouseコードベースのほとんどをリンクする必要があるため、リリースしていません。

## 分散クエリの実行 {#distributed-query-execution}

サーバーにクラスターセットアップがほぼ独立しています。 を作成することができ `Distributed` クラスター内の一つまたはすべてのサーバーの表。 その `Distributed` table does not store data itself – it only provides a “view” クラスターの複数ノード上のすべてのローカルテーブル。 から選択すると `Distributed` クエリを書き換え、ロードバランシング設定に従ってリモートノードを選択し、クエリをクエリに送信します。 その `Distributed` テーブル要求をリモートサーバー処理クエリーだけで最大のペースでの中間結果から異なるサーバできます。 その後、中間結果を受け取り、それらをマージします。 のテーブルを配布してできる限りの仕事へのリモートサーバーを送信しない多くの中間データのネットワーク.

INまたはJOIN節にサブクエリがあり、それぞれがaを使用すると、事態はより複雑になります。 `Distributed` テーブル。 これらのクエリの実行にはさまざまな戦略があります。

分散クエリ実行用のグローバルクエリプランはありません。 各ノ リモートノードのクエリを送信し、結果をマージします。 しかし、これは、高い基数グループbysまたは結合のための大量の一時的なデータを持つ複雑なクエリでは不可能です。 そのような場合、 “reshuffle” 追加の調整が必要なサーバー間のデータ。 ClickHouseはそのような種類のクエリ実行をサポートしておらず、その上で作業する必要があります。

## ツリーをマージ {#merge-tree}

`MergeTree` 家族のストレージエンジンを支える指数付けによりその有効なタイプを利用します。 主キーは、列または式の任意のタプルにすることができます。 Aのデータ `MergeTree` テーブルは “parts”. 各パーツは主キーの順序でデータを格納するので、データは主キータプルによって辞書式で順序付けされます。 すべてのテーブル列は別々に格納されます `column.bin` これらの部分のファイル。 ファイルは圧縮ブロックで構成されます。 各ブロックは、平均値のサイズに応じて、通常64KBから1MBの非圧縮データです。 ブロックは、連続して配置された列の値で構成されます。 列の値は各列の順序が同じです（主キーで順序が定義されています）ので、多くの列で反復すると、対応する行の値が取得されます。

主キー自体は次のとおりです “sparse”. なアドレス毎に単列、ある種のデータです。 独立した `primary.idx` ファイルには、N番目の各行の主キーの値があります。 `index_granularity` (通常、N=8192)。 また、各列について、 `column.mrk` 以下のファイル “marks,” これは、データファイル内の各N番目の行にオフセットされます。 各マークは、圧縮されたブロックの先頭にファイル内のオフセット、およびデータの先頭に圧縮解除されたブロック内のオフセットのペアです。 通常、圧縮ブロックはマークによって整列され、圧縮解除されたブロックのオフセットはゼロです。 データのための `primary.idx` 常にメモリに常駐し、データのための `column.mrk` ファイ

我々は一部から何かを読むつもりされているとき `MergeTree`、我々は見て `primary.idx` 要求されたデータを含む可能性のあるデータと範囲を見つけ、次に `column.mrk` これらの範囲の読み取りを開始する場所のオフセットを計算します。 まばらであるため、余分なデータを読み取ることができます。 ClickHouseは、単純なポイントクエリの高負荷には適していません。 `index_granularity` 各キーに対して行を読み取り、圧縮されたブロック全体を各列に対して圧縮解除する必要があります。 インデックスのメモリ消費量が目立つことなく、単一のサーバーごとに数兆行を維持できる必要があるため、インデックスを疎にしました。 また、主キーはスパースであるため、一意ではありません。 テーブルに同じキーを持つ多くの行を持つことができます。

ときあなた `INSERT` データの束に `MergeTree` その束は主キーの順序でソートされ、新しい部分を形成します。 が背景のスレッドを定期的に、選択部分と統合して単一のソート部の部品点数が比較的低い。 それが呼び出される理由です `MergeTree`. もちろん、マージにつながる “write amplification”. すべての部分は不変です：作成され、削除されますが、変更されません。 SELECTが実行されると、テーブル（部品のセット）のスナップショットが保持されます。 マージした後、我々はまた、我々はいくつかのマージされた部分は、おそらく壊れていることを確認した場合、我々は、そのソース部分とそれを置き換えることがで

`MergeTree` LSMツリーが含まれていないためではありません “memtable” と “log”: inserted data is written directly to the filesystem. This makes it suitable only to INSERT data in batches, not by individual row and not very frequently – about once per second is ok, but a thousand times a second is not. We did it this way for simplicity’s sake, and because we are already inserting data in batches in our applications.

> MergeTreeテーブルには、一つの(プライマリ)インデックスしか持たせることができません。 たとえば、データを複数の物理的順序で格納したり、元のデータとともに事前に集計されたデータを使用して表現することもできます。

バックグラウンドマージ中に追加作業を行っているmergetreeエンジンがあります。 例としては `CollapsingMergeTree` と `AggregatingMergeTree`. この処理として特別支援しました。 なぜなら、ユーザーは通常、バックグラウンドマージが実行される時間と、バックグラウンドマージが実行されるデータを制御できないからです。 `MergeTree` テーブルは、ほとんどの場合、完全にマージされた形式ではなく、複数の部分に格納されます。

## 複製 {#replication}

ClickHouseでのレプリケーションは、テーブルごとに構成できます。 きも複製されない一部の複製のテーブルと同じサーバです。 また、テーブルをさまざまな方法でレプリケートすることもできます。

レプリケーションは `ReplicatedMergeTree` 貯蔵エンジン。 のパス `ZooKeeper` ストレージエンジンのパラメータとして指定します。 同じパスを持つすべてのテーブル `ZooKeeper` 互いのレプリカになる：彼らは彼らのデータを同期させ、一貫性を維持する。 テーブルを作成または削除するだけで、レプリカを動的に追加および削除できます。

レプリケーション 次のセッションを持つ任意のレプリカにデータを挿入できます `ZooKeeper` データは、他のすべてのレプリカに非同期的に複製されます。 でClickHouseをサポートしていない更新、複製する紛争は無料です。 挿入のクォーラム確認がないため、あるノードが失敗すると、挿入されたばかりのデータが失われる可能性があります。

メタデータレプリケーションが格納され飼育係. 実行する操作を一覧表示するレプリケーションログがあります。 アクションとしては、get part、merge parts、drop partitionなどがあります。 各レプリカコピー、複製のログをキューにその行動からのキューに挿入します 例えば、挿入時には、 “get the part” actionを作成し、ログイン、レプリカのダウンロードいます。 マージはレプリカ間で調整され、バイトと同じ結果が得られます。 すべての部品を合併した場合と同様にすべてのレプリカ. では達成を補い、一つのレプリカのリーダーとして、レプリカを始めと融合し、書き込みます “merge parts” ログへのアクション。

クエリではなく、ノード間で転送されるのは圧縮された部分だけです。 合併処理され、各レプリカ多くの場合、自主的に下げるネットワークコストを回避することによるネットワークが増幅。 大合併をさらにネットワークする場合に限り重複製に遅れて波及してきています。

さらに、各レプリカは、部品とそのチェックサムのセットとしてzookeeperにその状態を保存します。 ローカルファイルシステム上の状態がzookeeperの参照状態から逸脱すると、レプリカは他のレプリカから欠落した部分や破損した部分をダウンロードする ローカルファイルシステムに予期しないデータや破損したデータがある場合、clickhouseはそれを削除せず、別のディレクトリに移動して忘れてしまいます。

!!! note "メモ"
    クリックハウスクラスタは独立したシャードから成り，各シャードはレプリカから成る。 クラスタは **ない弾性** したがって、新しいシャードを追加した後、データは自動的にシャード間で再調整されません。 代わりに、クラスタの負荷が不均一になるように調整されるはずです。 この実装では、より多くの制御が可能になり、数十のノードなど、比較的小さなクラスタでも問題ありません。 がクラスターの何百人ものノードを用いて、生産-このアプローチが大きな欠点. を実行すべきである"と述べていテーブルエンジンで広がる、クラスターを動的に再現れる可能性がある地域分割のバランスとクラスターの動します。

{## [元の記事](https://clickhouse.tech/docs/en/development/architecture/) ##}