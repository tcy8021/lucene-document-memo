- テキストをインデックス/検索可能なトークンに変換するクラス群
- Luceneはプレーンテキストのみ受け付ける
- `CharFilter`, `Tokenizer`, `TokenFilter` の順で処理される

# トークナイゼーション

- 入力テキストを小さなインデックス要素（トークン）に分割すること
- 前処理
  - HTMLのマークアップの削除
  - 特定パターンに一致するテキストの変換や削除
- 後処理
  - ステミング
  - ストップワードフィルタリング
  - テキスト正規化
  - 同義語の拡張

# Analyzer

- 責務: `TokenStream` を提供すること
  - 具体的には `createComponents(String)` で `TokenStreamComponents` を定義する
- 分析チェーンのfactoryで、`CharFilter`, `Tokenizer`, `TokenFilter` を生成する
- テキスト処理は行わない
- field aware
- インデックス作成時は `IndexWriter.addDocument(doc)` で呼ばれる
- クエリ時は `QueryParser` のパース中に呼ばれるが、ワイルドカードクエリなどでは分析が実行されない
- 「正しい」 `Analyzer` を選択するための経験則
  - 厚めにテストする
  - インデックス作成のパフォーマンスを低下させるので、分析が多くなりすぎないようにする
  - インデックス作成と検索に同じ `Analyzer` を使う
  - しかし異なる `Analyzer` を使うこともある
    - クエリ時はより多くのストップワードをフィルタリングしたい場合など

# CharFilter

- 責務: `Tokenizer` の前にテキストの変換を行い、修正されたオフセットを提供すること
- `Reader` のサブクラス
- 0個でも複数個でもよい

# Tokenizer

- 責務: 入力テキストをトークンに分割すること
- `TokenStream` のサブクラス
- 分析チェーンの最初に1個だけ定義できる
- `Reader` を入力に取る
- `Tokenizer` は抽象クラスで、サブクラスは `TokenStream.incrementToken()` をoverrideしなければならない
  - `incrementToken` では `attribute` を設定する前に `AttributeSource.clearAttribute()` を呼ばなければならない

# TokenFilter

- 責務: `Tokenizer` によって作成されたトークンの変更を行うこと
- `TokenStream` のサブクラス
- 0個でも複数個でもよい
- 別の `TokenStream` を入力に取る
- `TokenFilter` は抽象クラスで、サブクラスは `TokenStream.incrementToken()` をoverrideしなければならない

# Field Section Boundaries

- `Document.add(field)` が複数回呼び出されると、ドキュメントにそのフィールドの新しいセクションが作られるが、`Analyzer` のデフォルトの動作ではそれらを1つの大きなセクションとして扱う（フレーズ検索などでセクションをまたいで検索できる）
- `Analyzer.getPositionIncrementGap(fieldName)` をoverrideすることでセクション間のポジションギャップを導入できる

# Token Position Increments

- あるトークンの位置が前のトークンの位置からどれだけ離れているかを表す
- デフォルトは1
- "blue is the sky" のストップワードをフィルターすると、"blue" と "sky" がインデックスされ、`position("sky")=3+position("blue")` となる。フレーズクエリ "blue is the sky" は同じようにストップワードがフィルターされるのでドキュメントを見つけられるが、フレーズクエリ "blue sky" は位置の差が1なのでドキュメントを見つけられない。
- トークンをフィルタリングする場合、position incrementを正しくインクリメントする必要がある
- 用途
  - 文の境界でのフレーズ検索を禁止する（position incrementを大きくする）
  - 同義語の挿入（position incrementを0にする）

# Token Position Length

- トークンが占める位置の長さ
- 複数単語の同義語のために使用する（ドキュメントのサンプルがわかりやすい）

# その他

- トークンのconsumerは `TokenStream.incrementToken()` で消費を始める前に `TokenStream.rest()` を呼ばなければならない