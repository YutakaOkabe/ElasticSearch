# ElasticSearch

## はじめに

[始めてみよう | Elasticsearchリファレンス [5.4] | Elastic](https://www.elastic.co/guide/jp/elasticsearch/reference/current/getting-started.html)

- ElasticsearchはデータをdocumentというJSONで扱う
- documentを保存する先がindexで、RDSでいうデータベース
    - index に document を登録することを 「インデックスする」「インデキシング」という
- 以前は type というテーブルに相当するもののがあったが、現在は非推奨となっており全てのindexで`_doc` というtypeの１種類のみを使用する
- スキーマ定義のことをmappingという
- Lucene(ルシーン)：Elasticsearchの内部で利用されているオープンソースの検索エンジンライブラリ
    
    Apacheが提供しておりJavaで書かれている
    
    [Elasticsearchを理解するためにLuceneを使った検索エンジン構築に入門してみた - 好奇心に殺される。](https://po3rin.com/blog/try-lucene)
    
- RDSとの違い
    
    [データベースとしてのElasticsearch - Qiita](https://qiita.com/rjkuro/items/95f71ad522226dc381c8)
    

---

## インデックス

```bash
GET _cat/indices #インデックス一覧
# health | status | index | uuid | pri | rep | docs.count | docs.deleted | store.size | pri.store.size
# green  | open | movies | UZbpfERBQ1-3GSH2bnM3sg | 1 | 1 | 1 | 0 | 7.7kb | 3.8kb
PUT customer # 作成(データを登録すると勝手に作られる)
DELETE customer
```

- インデックスに対してエイリアスを設定可能
    - 複数のインデックスを一つのエイリアスに登録しておいて、そのエイリアスでGETすることで複数のインデックスに渡ってデータを取得できる `GET _alias/alias-name`
        - さらに新しくインデックスを追加するときも、そのインデックスを同じエイリアスに登録してあげることでElasticSearch側でデプロイみたいなことをすることが可能
    - エイリアスが一つのインデックスに紐づく場合はドキュメントの追加が可能
    - 複数のインデックスに紐づく場合は拒否される
- Index Template
    
    [Elasticsearch Index Template のバージョニング | DevelopersIO](https://dev.classmethod.jp/articles/elasticsearch-index-template-versioning/)
    
    - 新しい Index を作成する際に命名規約に沿っていれば、共通の構成要素を自動で適用できる仕組み
    - 以下の内容を設定可能
        - シャードの構成（Primary Shards、Replica Shards）
        - エイリアスの定義（Index Aliases）
        - カスタマイズした言語処理の定義（Analyzers、Char Filters、Token Filters）
        - フィールドマッピングの定義（Mappings）

---

## アナライズ処理

[Workshop Studio](https://catalog.us-east-1.prod.workshops.aws/workshops/26c005b2-b387-454a-b201-9b8f37f92f92/ja-JP/full-text-search/basic-concepts/normalization)

↑ 日本語取り扱う時めっちゃ参考になる

[【Elasticsearch 入門３】Analyzer の設定と日本語の全部検索](https://hogetech.info/bigdata/elasticsearch3)

[Elasticsearch 入門。その3 | DevelopersIO](https://dev.classmethod.jp/articles/elasticsearch-starter3/)

- Elasticsearchではデフォルトで値の大文字を小文字に変換して転置インデックスを作成しているので、大文字小文字の区別なくヒットする
    - 転置インデックス：単語とそれが含まれるドキュメントIDの紐付け
- 現状のマッピングで検索がどのように効いているか確認
    
    ```json
    GET user_index/_analyze
    {
    	"field" : "address.city",
    	"text" : "東京都"
    }
    ```
    
- アナライザ：転置インデックス作成前に走る処理で以下の3要素から構成される
    - Character Filter：HTMLタグを除去したり、文字列を置換したりなどの機械的な前処理を行う
        - `html_strip` ：HTMLタグの削除
            
            ```json
            GET _analyze
            {
              "char_filter": [
                "html_strip",
                {
                  "type" : "mapping",
                  "mappings" : [
                    "色色 => 色々"
                    ]
                }
                ],
                "text": "<b>人生色色</b>" // 人生色々 で出力される
            }
            ```
            
        - `icu_normallizer`：大文字小文字、半角カナ全角カナなどの変換を行う
            
            Token Filterとしても利用できるが、正確なトークン分析をするために前段で行っておくのが良い
            
            ```json
            POST _analyze
            {
              "text": "OｐeｎsｅaｒCｈは①⓪⓪㌫ｵｰﾌﾟンｿｰｽの検索／分析スイートです", // opensearchは100パーセントオープンソースの検索/分析スイートです
              "char_filter": ["icu_normalizer"]
            }
            ```
            
        - `kuromoji_iteration_mark`：踊り字(々, ゝ, ヽ)を直前の文字で置き換える
    - Tokenizer：文字列を空白や形態素解析など一定のルールに従って分割する ← これで転置インデックスの単語を生成する
        - `analysis-kuromoji`：日本語の意味のある単語で区切ってくれる
            
            最新トレンドを反映した固有名詞や業界固有のキーワードまでは必ずしも登録されていないので、辞書の継続的なアップデートやユーザー辞書の活用が必要
            
            kuromojiがデフォルトで使っているIPADICという辞書は2007年から更新が止まっている
            
            kuromojiをneologd辞書に対応させたプラグインもある [https://christina04.hatenablog.com/entry/2016/05/18/193000](https://christina04.hatenablog.com/entry/2016/05/18/193000)
            
            ```json
            GET opensearch_dashboards_sample_data_ecommerce/_analyze
            {
              "tokenizer" : "kuromoji_tokenizer",
              "text" : "今日の天気は晴れです" // 今日　の　天気　は　晴れ　です になる
            }
            ```
            
        - `N-Gram`：テキストから N 文字ずつ取り出してトークン化する
            
            min_gramとmax_gramを設定することで例えば1文字のトークンから3文字のトークンまで取得でき、検索ヒット率は上がるが検索ノイズが増える
            
            [Workshop Studio](https://catalog.us-east-1.prod.workshops.aws/workshops/26c005b2-b387-454a-b201-9b8f37f92f92/ja-JP/full-text-search/basic-concepts/tokenization)
            
    - Token Filter：トークン分割が行われた後に各トークンに対して変換処理を行う
        
        Character Filterがシンプルな文字の置き換えを行うのに対して、Token Filterは様々な処理を提供
        
        - `kuromoji_baseform`：動詞や形容詞の変化形を原形に置き換える
            
            ※ ステミング：語尾が変化する語の、変化しない部分で検索する方法（[swims], [swimming], [swimmer] → [swim]）
            
            ```json
            POST _analyze
            {
              "tokenizer": "kuromoji_tokenizer", 
              "filter": ["kuromoji_baseform"],
              "text": "寿司を食べた。美味しかったな" // 寿司 を 食べる 美味しい た な
            }
            ```
            
        - `stop`：品詞を除外する
            
            jp_stopとの違いは日本語向けのリストがない点と、外部ファイルの読み込みに対応している点
            
            ja_stopとの併用も可能で、どちらも指定することで日本語向けの品詞を除去しつつ、外部ファイルを読み込める
            
        - `ja_stop`：ストップワード("てにをは"など)を除去する
            
            kuromoji_part_of_speechとは異なり、品詞単位ではなくストップワードリストに含まれるワードを除去（デフォルトリストも存在する）
            
        - `kuromoji_part_of_speech` ：指定した品詞タグに該当するトークンを削除する（上記の例で言う「を」「た」「な」を削除する）
        - `lowercase`：単語を小文字化する
            
            ```json
            GET _analyze
            {
             "tokenizer": "standard",
             "filter": ["lowercase", "stop" ],
             "text": "To Be Or Not To Be, That Is The Question" // question のみ出力される
            }
            ```
            
        - `kuromoji_readingform` ：トークンの読み仮名(カタカナ、ローマ字)を生成する ← 検索機能に使えそう
- kuromoji analyzer
    
    日本語を検索するためのアナライザで、日本語用のCharacter Filter, Tokenizer, Token Filterをまとめてプラグインとして提供してくれている
    
    | Analyzer | kuromoji Analyzer の構成要素 |
    | --- | --- |
    | Char Filter | CJKWidthCharFilter (Apache Lucene) |
    | Tokenizer | kuromoji_tokenizer |
    | Token Filter | kuromoji_baseform 
    kuromoji_part_of_speech
    ja_stop 
    kuromoji_stemmer 
    lowercase |
    
    [Elasticsearchで日本語検索を扱うためのマッピング定義 - ZOZO TECH BLOG](https://techblog.zozo.com/entry/elasticsearch-mapping-config-for-japanese-search)
    
    - kuromoji analyzerにカスタムフィルターを追加してひらがな・カタカナの表記揺れを解消する方法
        
        [Elasticsearchでカタカナも引っ掛かるようにしたい！【kuromoji】 - Qiita](https://qiita.com/toshinobu111/items/152be46443d7e5d0caeb)
        
- 同義語の吸収
    
    [https://catalog.us-east-1.prod.workshops.aws/workshops/26c005b2-b387-454a-b201-9b8f37f92f92/ja-JP/full-text-search/basic-concepts/synonym](https://catalog.us-east-1.prod.workshops.aws/workshops/26c005b2-b387-454a-b201-9b8f37f92f92/ja-JP/full-text-search/basic-concepts/synonym)
    
    - インデックス時に展開する方法
        - `synonym`：パインアップルとパインをパイナップルとして登録するToken Filter
            
            ```json
            "synonyms": [ "パインアップル,パイン=> パイナップル" ]
            ```
            
            `=>` ではなく`,` で繋げた時には各キーワードは相互に展開されインデックスに格納される
            
            シノニム定義ファイルimportする方法も可能
            
    - 検索時に展開する方法
        - インデックス時にも検索時にも同義語展開を行う方法
            - インデックス作成時に上記のシノニムを設定していれば、検索時にも適用してくれる
            - しかしシノニムの設定を更新したら、データの再登録(再インデックス)を行う必要があるので大変
        - インデックス時には同義語展開を行わず、検索時にのみ同義語展開を行う ← とりあえずこれを試すべき
            - 検索フィルターとして上記の `,` で区切るタイプを用いることで、検索文字列を展開して検索を行うことができる
            - 結果的に3単語で検索しているのと同じなので、性能はインデックス時に展開するよりも悪くはなる

---

## マッピング

[Field data types | Elasticsearch Guide [8.5] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)

- 既存のマッピングを後から変更できないのでreindexが必要
    - 以下の場合のみreindexせずに対応が可能
        - オブジェクト型のプロパティに対して新しいプロパティを追加
        - 既存のフィールドにmulti_fieldとして新しいフィールドを割り当てる場合（text型のデータにkeyword型を付与するなど）
        - `ignore_above` パラメータの変更 ← stringの上限値でこれを超えるとドキュメントはインデックスされるが、そのフィールドにデータが入らない
        
        [Update mapping API | Elasticsearch Guide [8.6] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/indices-put-mapping.html#add-new-field-to-object)
        
    - フィールド名は変更できないけど新しくフィールドを作ってそこにAliasを貼ることで対応は可能
- ドキュメントを追加する際に自動でマッピングしてくれる(Dynamic Mapping)が、以下のように手動で定義することを推奨
    
    ```json
    GET blogs/_mapping // 確認
    PUT blogs // インデックスを作成するときに一緒にマッピングまで定義
    {
      "mappings": {
        "properties": {
          "content" : {
            "type": "text",
            "analyzer": "kuromoji"
          },
          "url" : {
            "type": "keyword"
          },
          "timestamp": {
            "type": "date"
          },
          "author" : {
            "type": "text", // 検索するためにtext型にしてるが、完全一致でも検索できるようにauthor.rawにkeyword型を設定
            "fields": {
              "raw" : {
                "type" : "keyword"
    ```
    
- 複数のフィールドを定義する
    
    例えばLIKE検索も完全一致検索もしたい場合は、部分一致のtext型に加えて fieldsでrawに完全一致用のkeyword型を与えてやることでどちらでも検索が可能
    
    ```json
    PUT /articles
    {
      "mappings": {
        "properties": {
          "name": { 
            "type": "text",
            "fields": {
              "raw": { // ここの命名はなんでも良いけどrawが一般的
                "type": "keyword",
    // LIKE検索
    GET /articles/_search
    {"query": {"match": {"name": "ほげ"
    // 完全一致
    GET /articles/_search
    {"query": {"term": {"name.raw": "ほげ"
    ```
    
    [Elasticsearch のマッピングについて調べてみた！ - BookStore's Code ...](https://baubaubau.hatenablog.com/entry/2020/07/02/203000)
    

- 空文字は検索した時に見えるけど、nullは見えない
- nullの扱いについて
    - RDBではテーブルの全レコードは同じフィールドを持ち、値がない場合にNULLという値で表現される
    - Elasticsearchは同じマッピングタイプでもドキュメントごとにフィールドがあったりなかったりする事が可能なので、値がないことをnullではなく、フィールドがない、という事で表現する
    - フィールドがないドキュメントはマッピングタイプの定義上ではstore=trueなフィールドだったとしても取得時にフィールドの値が返されない
    
    [データベースとしてのElasticsearch - Qiita](https://qiita.com/rjkuro/items/95f71ad522226dc381c8#insert)
    

---

## CRUD

### ドキュメントの追加

```json
POST blogs/_doc
{"content" : "Elasticsearchやってみた", "url" : "http://test.example.com", "author" : "中村",  "timestamp" : 111}
PUT blogs/_doc/1 # PUTの場合はIDが必須
{}
```

### ドキュメントの更新

```json
// 部分更新(docが必ず必要)
POST <index>/_update/<id>
{
  "doc": {
    "field_name": "value"
  }
}
// 更新or作成
POST <index>/_update/<id>
{
  "upsert": {
    "field_name": "value"
  }
	// またはこの書き方
	"doc": {
    "field_name": "value"
  },
  "doc_as_upsert"true
}
```

### 複数処理

#### bulk

- _bulk API は単一の API リクエストで複数のドキュメントを一括で作成、更新、削除することが可能
- 1 つの処理ごとに都度 API リクエストを発行する場合と比較して、処理効率が向上する
- 個々のオペレーション操作の成否に関わらず、一連の処理が完了した時点でレスポンスコード 200 と合わせて結果を返却
- 全体を通してのエラー有無は `errors`から判断可能
- operation
    - create：作成（同一IDのドキュメントが存在する場合はエラー）
    - index：作成（同一IDのドキュメントが存在する場合は上書き）
    - update: 部分更新。デフォルトは更新対象が存在しなかったらエラー。doc_as_upsert=true の場合、更新対象が存在しなかったらドキュメントを新規作成
    - delete: 削除。削除対象が存在しない場合、resultはnot foundになるがbulk全体のerrorsはfalseになる

```json
POST _bulk
{ "<operation>" : { "_index" : "<index-name>", "_id" : "<id>" } }
{ "field1" : "value1", "field2": "value2", ... } // データは1行で書く必要がある
{ "<operation>" : { "_id" : "<id>" } } // POST <index>/_bulk とすることで_indexは省略可能
{ "doc": { "field1" : "value1", "field2": "value2", ... } } // updateの場合はdocが必要
{ "<operation>" : { "_index" : "<index-name>"} } // idを省略するとOpenSearch が作成したランダムな ID が割り当てられる
{ "field1" : "value1", "field2": "value2", ... }
```

#### mget

- 複数のドキュメントを一括で取得する
- インデックス名をパスに含めずに同 API を実行する場合、複数のインデックスにまたがってドキュメントを取得できる

```json
GET _mget
{
  "docs": [
    {"_index": "movies", "_id": 1},
    {"_index": "musics", "_id": 1}
  ]
}
```

#### update_by_query, delete_by_query

- query 内に記載した検索条件に合致したドキュメントを複数一括更新・削除

---

## 検索

[Full-text queries](https://opensearch.org/docs/latest/opensearch/query-dsl/full-text/)

- ElasticsearchではGETで検索してもPOSTで検索しても、リクエストボディに条件をJSONで書けばうまくいくようにできてる
    - GETにリクエストボディはRFC的には違反してはいない
    - curlやOpenAPIでは定義可能だが、Postmanでは不可
    - 現状はネストを含む構造化した検索条件の指定は、POSTメソッドで設計しておくのが無難
    
    [HTTP検索条件、GETにするか？POSTにするか？ | フューチャー技術ブログ](https://future-architect.github.io/articles/20210518a/)
    
- ElasticSearchではデフォルトだとリクエストする単語が1単語でも、アナライザーで解析されて最小言語単位でOR検索される！
- `"explain": true` を指定することでどのスコアで検索にヒットしているかがわかる

```bash
POST opensearch_dashboards_sample_data_ecommerce/_count
POST opensearch_dashboards_sample_data_ecommerce/_search
POST opensearch_dashboards_sample_data_ecommerce/_search?_source=title,more # 必要なフィールドだけ取得
POST opensearch_dashboards_sample_data_ecommerce/_search?filter_path=hits.hits._id,hits.hits._score # 必要な情報だけ取得

# _docを使ってidで取得
GET opensearch_dashboards_sample_data_ecommerce/_doc/TpbsOIUB-5Wv8n86_LxX
```

```json
// matchはtextフィールドに対する検索, termはkeywordに対する検索
{"query": {"match": { "business_name": "tokyo" }}} // 等値なものを1件だけ
{"query": {"match": { "business_name": "tokyo fukuoka" }}} // (OR検索) tokyo または fukuoka なもの
// ↑何もオブションを指定する必要がなければ key:search_word でよい
// ↓オブションを指定したかったら一つネストさせてqueryにsearch_wordを入れる
{"query": {"match": { "business_name": {"query": "tokyo fukuoka", "operator": "and"} }}} // (AND検索) tokyo かつ fukuoka なもの（語順は考慮されない）
{"query": {"match": { "business_name": {"query": "tokyo fukuoka osaka", "minimum_should_match": 2} }}} // (ゆるいAND検索) tokyo,fukuoka,osakaのうち2つが含まれるもの

{"query": {"match_phrase": { "business_name": "tokyo fukuoka" }}} // 語順を考慮したAND検索 ※コストが高い ※語順だけなので完全一致ではない

{"query": {"multi_match": {"query": "東京", "fields": [ "name", "address.city"]}}} // 複数のフィールドでOR検索

{"query": {"match_all": {}}} // インデックスのドキュメント全て
{"query": {"match_all": {}}, "from": 10, "size": 10} // 11~20件目まで
{"query": {"match_all": {}}, "sort": { "balance": { "order": "desc" } }} // ソート（デフォルトはscore順）
{"query": {"match_all": {}}, "_source": ["account_number", "balance"]} // sourceの内容を絞る

{"query": {"range": {"timestamp": {"gte": "now-1d/d","lt": "now/d"}}}} // 昨日から今日まで

// 40歳でUA以外に住んでいて、名前がhogeまたはfugaの人
{
  "query": {
    "bool": {
      "must": [ // スコアに影響する=検索条件を検索順位に反映したい場合に使用
        { "match": { "age": "40" } }
      ],
      "must_not": [ // スコアに影響しない
        { "match": { "state": "UA" } }
      ],
			"should": [ // スコアに影響する
        { "match": { "name": "hoge" } },
        { "match": { "name": "fuga" } }
      ]

// 全てのドキュメントの中でbalanceが20000以上30000以下
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": { // mustと同じくANDになるがスコアに影響しない=検索順位に反映する必要が無い場合に使用
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
```

### レスポンス

```json
{
  "took" : 7, // 実行時間(ミリ秒)
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 100,
      "relation" : "eq"
    },
    "max_score" : 4.016948, // デフォルトソートはスコア順（sortをしていするとsort条件も表示される）
    "hits" : [ // 実際にヒットした結果
      {
        "_index" : "opensearch_dashboards_sample_data_ecommerce",
        "_id" : "TpbsOIUB-5Wv8n86_LxX", // ElasticSearch側で一意に振られるID ※ ドキュメントIDを指定せずにドキュメントを作成すると、自動的にドキュメントIDが割り当てられる
        "_score" : 4.016948, // そのドキュメントが指定した検索クエリにどの程度一致しているかの相対的な基準となる数値
        "_source" : {
          "category" : [
            "Men's Clothing"
          ],
```

- スコア
    - Elasticsearchでは類似度の計算に**BM25**というアルゴリズムを使っている
        - **TF(term frequency)：**検索単語の頻出頻度が多い程スコアが高くなる
        - **IDF(inverse document frequency)：**検索単語がたくさんのドキュメントに存在するほど、スコアが低くなる
        - **Field length：**該当フィールドの値の長さが平均より短いほうがスコアが高くなる

---

## 高度な検索

- ブーストフィールド
    - あるフィールドの一致を他のフィールドの一致より重く重み付けする
    - デフォルトの重みは1
    - multi_match使用時の書き方がちょっと特殊
        
        ```json
        {"query": {"match": { "business_name": {"query": "tokyo fukuoka", "boost": 2} }}}
        {"query": {"multi_match": {"query": "john","fields": ["title^4", "plot^2", "actors", "directors"]}}}
        ```
        
- fuzzy queryであいまい検索
    - タイプミスを変換する程度のあいまい検索実装を簡単にできる
    
    ```json
    // AUTO: 自動で判定
    // "fuzziness" : 0 # 1~2文字向け
    // "fuzziness" : 1 # 3~5文字向け
    // "fuzziness" : 2 # 6文字以上向け
    "query": {"match": {"title": {"query": "ベそチャー","fuzziness" : "AUTO"}}}
    ```
    
    - 違う文字でヒットしてもスコアは変わらないことに注意
    
    [Elasticsearchのfuzzy queryを使って、あいまい検索を試してみる - Qiita](https://qiita.com/EastResident/items/5ebdd5d301838fd1be24)
    
- 集計
    
    [集約の実行 | Elasticsearchリファレンス [5.4] | Elastic](https://www.elastic.co/guide/jp/elasticsearch/reference/current/gs-executing-aggregations.html)
    
- nestedフィールド
    - プロパティに複数のオブジェクトが配列になったものが入ってくる時、ElasticSearchはデフォルトでそれらをフラット化して保存するので、一つのオブジェクト単位での結びつきがなくなる
    - オブジェクト単位で検索結果を求めたい時はフィールドをnestedにすることで解決できる
    - nested型の検索にはnested queryという特別なクエリを用いる必要がある
    - nested型のドキュメントは親要素+子要素分のドキュメントが内部で作られているのでドキュメントの数が増える傾向にある
    
    [Elasticsearch 入門。nested フィールドについて | DevelopersIO](https://dev.classmethod.jp/articles/elasticsearch-starter5/)
    
    [ZOZOTOWNの検索基盤におけるElasticsearch移行で得た知見 - ZOZO TECH BLOG](https://techblog.zozo.com/entry/migrating-zozotown-search-platform)
    
    [Nested field type | Elasticsearch Guide [8.6] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/8.6/nested.html)
    
- parent-child
    - nestedと同じくJOINを表現する方法の1つであるが違いとしては
        - nestedは親ドキュメントをインデックスする時点で子ドキュメントも一緒にいれてやる必要がある
        - parent-childは親子が独立しており、子ドキュメントをあとから追加したり、子ドキュメントだけを更新するということができる
            - ただし親と子は同じシャードに含まれないとならないという制限はある
    - nestedのほうが検索のパフォーマンスは良い
- scroll API
    - searchAPIはデフォルト10件しか取得できず、fromとsizeを指定しても1万件以上は取得できない
    - Scroll APIを利用すると、実行時点でのスナップショットを保存し、取得しきれなかった分を辿っていくことができる
    - レスポンスとして検索結果と「scroll_id」が返却されるので、「scroll_id」を元にリクエストを繰り返し投げることで全データを取得する
    
    ```json
    // 初回リクエスト
    GET article/_search?pretty&scroll=1m // スナップショットの保存期限を1分に設定
    {
      "size": 9000,
      "query": {
    // レスポンス
    {
      "_scroll_id": "XXXXXXXXX・・・XXX",
      ...
      "hits": {
        "total": 23514,
        "max_score": 1,
    // 2回目リクエスト
    GET article/_search/scroll?pretty
    {
      "scroll": "1m",
      "scroll_id": "XXXXXXXXX・・・XXX"
    }
    
    ```
    

---

## 日本語検索

- デフォルトではリクエストする単語が１単語でもアナライザーで解析され、分割された最小単語単位でORで検索される
    
    [Elasticsearch 日本語でフレーズ検索が必要なわけ](https://medium.com/hello-elasticsearch/elasticsearch-22a369387dc5)
    
    ↑の記事では”でくくってフレーズ検索にしてるけど、これに関してはkeyword検索で良さそう
    
- 人物漢字名をひらがなで部分一致させる方法
    
    [Elasticsearchのひらがなでの検索時のトリックについて雑談 - はてだBlog（仮称）](https://itdepends.hateblo.jp/entry/2020/01/22/001650)
    
- キーワードサジェストを実現する方法
    
    [Elasticsearch キーワードサジェスト日本語のための設計](https://medium.com/hello-elasticsearch/elasticsearch-%E3%82%AD%E3%83%BC%E3%83%AF%E3%83%BC%E3%83%89%E3%82%B5%E3%82%B8%E3%82%A7%E3%82%B9%E3%83%88%E6%97%A5%E6%9C%AC%E8%AA%9E%E3%81%AE%E3%81%9F%E3%82%81%E3%81%AE%E8%A8%AD%E8%A8%88-352a230030dd)
    
- 日本語辞書としてSudachiを利用
    
    [検索基盤チームのElasticsearch×Sudachi移行戦略と実践 - エムスリーテックブログ](https://www.m3tech.blog/entry/sudachi-es)
    
    - Sudachiの辞書
        
        [Sudachi辞書の紹介 Part 1　-3種類の辞書- - Qiita](https://qiita.com/sakamoto_mi/items/4b55898aa221c290f411)
        

---

## パフォーマンス向上

- インデックスの容量削減
    - _allや_sourceの使用を止めるだけでも結構効果がある
    
    [Elasticsearchインデクシングパフォーマンスのための考慮事項 - Qiita](https://qiita.com/rjkuro/items/e79eec7ffb0511b7c678)
    
    [Elasticsearchのマッピング設定最適化によるインデキシングパフォーマンス改善への取り組み - ZOZO TECH BLOG](https://techblog.zozo.com/entry/es-mapping-configuration)
    
- 大きなデータは返さないようにする
    
    Elasticsearchはデータベースではなく検索エンジン
    
    上位ドキュメントを取得するのに向いているが全件取得するものではない
    
    [Elasticsearchを使う前に先に読んでおくべきドキュメント - Qiita](https://qiita.com/HirokiSakonju/items/1a4488298b62d05de669)
    

---

## 移行

- インデックスの移行
    - `_reindex` が使える
        - `wait_for_completion=false`でバックグラウンドでの移行も可能？
        - queryで範囲指定できるので前日までのデータを全て移行するなども可能
        - またドキュメントIDが同じデータは重複されずに上書きされる
    - エイリアスを使っておくことでアプリケーション側を変更せずに移行が可能（エイリアスの削除と追加を同時に行える）
    
    [Workshop Studio](https://catalog.us-east-1.prod.workshops.aws/workshops/26c005b2-b387-454a-b201-9b8f37f92f92/ja-JP/full-text-search/advanced-topics/update-dictionary#)
    
    [Elasticsearch の reindex をするために試行錯誤して分かったこと - Uzabase for Engineers](https://tech.uzabase.com/entry/2022/04/18/193513)
    

- データベースとしてElasticSearchを単体を使うのはここが良くない
    - 更新、削除に向いてない
        - joinがない→正規化できない→RDBだと一つのフィールドの変更でいいものが全てのデータを変更する必要がある
        - トランザクションがない
    - スキーマの変更に弱い
        - マッピングは基本的に変更できない
    - ロバスト性が低い
        - 時間がかかるクエリをキャンセルできない、メモリ不足になった時にどうするのかなどが考えられてない
    
    [Elasticsearch as a NoSQL Database](https://www.elastic.co/jp/blog/found-elasticsearch-as-nosql)
    
    ElasticSearchは他のデータベースと併せて使うべき
    
    - RDSと一緒に使いながらも検索性能向上のためにElasticSearchに入れるデータを増やした事例もある
        
        [ZOZOTOWNの検索基盤におけるElasticsearch移行で得た知見 - ZOZO TECH BLOG](https://techblog.zozo.com/entry/migrating-zozotown-search-platform)
        
    

## 周辺技術

- 形態素解析
    - 自然言語処理の一部で、**自然言語で書かれた文を言語上で意味を持つ最小単位(＝形態素)に分け、品詞や変化などを判別すること**
    - 日本語は英語などの言語と異なり単語境界に明確な区切り文字が存在しないため、いい感じに文字を分割する必要がある
    - いい感じに分割する作業をトークナイズ、いい感じに分割した文字をトークンという
        - 代表的なトークナイザ　※これらのアルゴリズムは同じ分割結果を返すこともあれば，異なる分割結果を返すこともある
            - MeCab：辞書を用いてラティスを構築し，その後最適な単語列を決定する
                - NEologd：Web 上の言語資源から得た新語を追加した MeCab 用のシステム辞書
            - Kytea：文字レベルで単語境界を判定する
    
    [新語・固有表現に強い「mecab-ipadic-NEologd」の効果を調べてみた](https://engineering.linecorp.com/ja/blog/mecab-ipadic-neologd-new-words-and-expressions/)
    
    [トークナイザをいい感じに切り替えるライブラリ konoha を作った - Qiita](https://qiita.com/klis/items/bb9ffa4d9c886af0f531)
    
- Elastic Stackは Elasticsearch 関連製品の総称

| 製品名 | 機能 |
| --- | --- |
| Elasticsearch | ドキュメントを保存・検索します。 |
| Kibana | データを可視化します。 |
| Logstash | データソースからデータを取り込み・変換します。 |
| Beats | データソースからデータを取り込みます。 |
| X-Pack | セキュリティ、モニタリング、ウォッチ、レポート、グラフの機能を拡張します。 |