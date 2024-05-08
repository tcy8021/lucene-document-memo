# Search Basics

- 検索を行うとき、通常は `IndexSearcher.search(Query, int)` を呼ぶ
  - スコアリング処理が開始
  - いくつかのインフラストラクチャのセットアップ
  - `Weight` の実装と、その `Scorer` or `BulkScorer` インスタンスに制御が移る
  - 詳細はアルゴリズムセクションを参照

# Query Classes

## TermQuery

- 指定された `Term` （特定のフィールドに出現する単語）を含む全てのドキュメントにマッチする
  - `Term` はフィールド名と文字列のテキストから構成される
- 以下の `TermQuery` は"fieldName"というフィールドに"term"という単語を含むドキュメントにマッチする

```Java
TermQuery tq = new TermQuery(new Term("fieldName", "term"))
```

## BooleanQuery

- 2つ以上の `BooleanClause` で構成される（`Iterable<BooleanClause>` をimplementsしている）
- `BooleanClause` は2つの要素で構成される
  - サブクエリ: `Query` インスタンス
  - 演算子（`BooleanClause.Occuer`）: サブクエリを他の節とどのように組み合わせるか記述
- `BooleanClause.Occuer` はEnumで、4つのメンバーがある
  - `SHOULD`
    - 節が結果セットに含まれる可能性はあるが、必須ではない場合に使用する
    - `BooleanQuery` が `MUST` 節を持たない場合、1つ以上の `SHOULD` 節にマッチしなければならない
  - `MUST`
    - 節が結果セットに必ず含まれ、スコアリングに使用させる場合に使用する
  - `FILTER`
    - 節が結果セットに必ず含まれ、スコアリングに使用させない場合に使用する
  - `MUST_NOT`
    - 節が結果セットに含まれてはならない場合に使用する
- 節が多すぎると検索中に `TooManyClauses` 例外が投げられる
  - `WildcardQuery` などによって `Query` が多くの `TermQuery` 節を持つ `BooleanQuery` に書き換えられる場合によく発生する
  - 最大節数のデフォルト値は1024だが、`IndexSearcher.setMaxClauseCount(int)` で変更可能

## Phrases

- フレーズ検索を処理する方法は様々な方法がある
  - `PhraseQuery`
    - `Term` 列にマッチする
    - "new york"のような入力に対して `QueryParser` によって構築される
    - `PhraseQuery` で定義されたタームの位置と文書内のタームの位置との編集距離であるslopを与えることができる（デフォルトは0）
  - `MultiPhraseQuery`
    - フレーズ内の1つの位置に対して複数のタームを受け付ける
    - 同義語を含むフレーズクエリを実行するために使用できる
  - `queries` モジュールのインターバルクエリ
    - フィールド内の区間を検索するためのクラス

## PointRangeQuery

- 数値の範囲に含まれるすべてのドキュメントにマッチする
- このクラスを動作させるためには、数値フィールド（`IntPoint`, `LongPoint`, `FloatPoint`, `DoublePoint`）のいずれかを使用してインデックスを作成する必要がある
  - これらのフィールドに対しては `IntPoint.newRangeQuery​(String field, int lowerValue, int upperValue)` のようにファクトリメソッドでレンジクエリを作成できる
  - 多次元フィールド用のメソッドもある
- ちなみに `XXXPoint` 型は高速なレンジフィルターのためのindexedなフィールド
  - 値を格納する場合は別の `StoredField` を追加する必要がある

## PrefixQuery, WildcardQuery, RegexpQuery

- `PrefixQuery`
  - 特定の文字列で始まるタームを持つドキュメントにマッチする
- `WildcardQuery`
  - `*` （0文字以上にマッチ）と `?` （正確に1文字にマッチ）を使って `PrefixQuery` を一般化したもの
  - かなり遅くなる可能性がある
  - 非常に遅くなるので、`*` や `?` で始めてはいけない
    - `QueryParser` の中にはデフォルトでこれを許可していないものもあるが、`setAllowLeadingWildcad` でそれを解除できる
- `RegexpQuery`
  - 正規表現パターンにマッチするタームを持つドキュメントを特定する

## FuzzyQuery

- 指定したタームに類似したタームを含むドキュメントにマッチする
- 類似度はレーベンシュタイン距離を用いて決定される

# Scoring

- `Query`: どのドキュメントがマッチするか決定する
- `Similarity`: マッチしたドキュメントにどのようなスコアを割り当てるか決定する
  - TODO: `Similarity` の説明
- `BoostQuery` は `Query` をラップし、スコアをブーストさせることができる

# Changing Scoring

## Changing the scoring formula

- `Similarity` の変更はスコアリングに影響を与える簡単な方法
  - index-time: `IndexWriterConfig.setSimilarity(Similarity)`
  - query-time: `IndexSearcher.setSimilarity(Similarity)`
  - 上記の2つで同じ `Similarity` を使用する

## Integrating field values into the score

- フィールド値をスコアリングに使用する方法は2つある
  - `FeatureField` 型のフィールドに値を格納し、クエリ時にその値を使う
    - TODO: `FeatureField` の説明
  - スコアリングの素性をdocValuesフィールドにインデックスしておき、`FunctionScoreQuery` で類似度スコアと組み合わせる（非効率だが柔軟性がある）

# Custom Queries

- Luceneの検索は4つの主要なクラスで構成される
  - `Query`
    - ユーザが求めるものの抽象表現
  - `Weight`
    - クエリを与えられたインデックスに対して特化させたクエリの内部表現
    - `Query` オブジェクトとスコア計算に使用するインデックスの統計情報を紐づける
  - `Scorer`
    - スコア計算プロセスの核となるクラス
    - `iterator()` は、与えられたセグメントに対して、クエリにマッチするドキュメントのdoc idの昇順のイテレータ（ `DocIdSetIterator` ）を返す
    - サブクラスが持つ `score()` でドキュメントのスコアを返す
  - `BulkScorer`
    - ドキュメントの集まりに対して1度でスコア計算を行う抽象クラス
    - デフォルトの実装では `Scorer` からのヒットを単純に反復するが、`BooleanQuery` などいくつかのクエリではより効率的な実装がある

## The Query Class

- `Query` クラスは他のスクアリングクラスを作成したり、クラス感の機能を調整する役割を担うことが多い
- 重要なメソッドは2つ
  - `createWeight(IndexSeacher searcher, ScoreMode scoreMode, float boost)`
    - `Weight` はクエリの内部表現なので、各 `Query` の実装は `Weight` の実装を提供しなければならない
    - 下の "The Weight Interface" で詳述
  - `rewrite(IndexSearcher searcher)`
    - クエリを "primitive query" に書き換える
    - "primitive query" とは、`createWeight(IndexSeacher searcher, ScoreMode scoreMode, float boost)` を実装するクエリのこと
    - 例えば、`PrefixQuery` は `TermQuery` からなる `BooleanQuery` に書き換えられる

## The Weight Interface

- `IndexSearcher` に依存する状態は `Query` クラスではなく `Weight` の実装に格納されなければならない
- インターフェースは3つの主要メソッドを定義している
  - `scorer()`
    - 新しい `Scorer` を作成する
    - `Scorer` の定義は下の "The Scorer Class" で詳述
  - `explain(LeafReaderContext context, int doc)`
    - 与えられたドキュメントがなぜそのようにスコア付けされたのか説明する
    - 通常、`TermWeight` のような `Similarity` を介してスコア付けするweightは `Similarity` の実装 `SimScore.explain(Explanation freq, long norm)` を使用する
  - `matches(LeafReaderContext context, int doc)`
    - マッチの位置とオフセットに関する情報を与える
    - 通常、ハイライト表示の実装に便利

## The Scorer Class

- `Scorer` は以下のメソッドを定義し、実装する必要がある
  - `iterator()`
    - クエリにマッチするすべてのドキュメントを反復処理できる `DocIdSetIterator` を返す
  - `docId()`
    - マッチした内容を含む `Document` のidを返す
  - `score()`
    - 現在のドキュメントのスコアを返す
    - アプリケーションに適した方法で決定することができる
    - 例えば、`TermScorer` は設定された `Similarity` に従う（つまり内部で `SimScore.score(float freq, long norm)` を呼ぶ）
  - `getChildren()`
    - サブスコアラーを返す

## The BulkScorer Class

- 抽象メソッドは1つ
  - `score(LeafCollector,Bits,int,int)`
    - 指定された最大ドキュメントまでスコア付けする

# Appendix: Search Algorithm

- `IndexSearcher.search(Query, int)` でスコアリングが開始
- `IndexSearcher` は `TopScoreDocCollector` を作成し、`Weight` とともに別の `search` メソッドに渡す
- `search` メソッド内で `Weight` が `Scorer` を作成
  - `Weight` が返す `Scorer` は `Query` のタイプにより異なる
  - 例えば、複数のクエリタームを使用するアプリケーションでは、`BooleanWeight` から作成される `BooleanScorer2`
- `Scorer.score()` が `Collector` を取り込んでスコア計算を行う
- `TopDocs` が返される
  - `TopScoreDocCollector` は `PriorityQueue` を使って上位の検索結果を収集する
