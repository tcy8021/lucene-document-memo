# document

- `Document`
  - インデックス作成と検索の単位
  - `IndexableField` の集合
- `IndexableField`
  - インデックス作成用の1フィールドの表現
  - 各フィールドは名前とテキスト値を持つ
    - 多分これはインデックスでの表現の話
    - Javaでの表現では、このインターフェースを実装した `Field` クラスが3つの（Java用語の）フィールドを持つ
      - `IndexableFieldType type`: フィールドの型
      - `String name`: フィールド名
      - `Object fieldData`: テキスト（ `String`, `Reader`, 分析済みの `TokenStream` ）、バイナリ（ `byte[]` ）、数値（ `Number` ）
  - `IndexWriter` はドキュメントを `Iterable<IndexableField>` として消費する
  - Luceneがどのように扱うべきかを表す多数のプロパティを持つ（indexed, tokenized, storedなど）
