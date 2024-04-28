# document

- Document
  - インデックス作成と検索の単位
  - `IndexableField` の集合
- IndexableField
  - インデックス作成用の1フィールドの表現
  - 各フィールドは名前とテキスト値を持つ
  - `IndexWriter` はドキュメントを `Iterable<IndexableField>` として消費する
  - Luceneがどのように扱うべきかを表す多数のプロパティを持つ（indexed, tokenized, storedなど）
