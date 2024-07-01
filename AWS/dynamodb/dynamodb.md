# DynamoDB

## コンポーネント

- コントロールプレーン
- クエリフロントエンド
- パーティションマネージャー（パーティションサーバーへのルーティング）
- パーティションサーバー

## データの格納先

- パーティションキーのハッシュ値により格納するパーティションサーバーが決定される
- 同じパーティションサーバーが 3 つの AZ にそれぞれ存在するため、1 つのデータは 3 つのサーバーに格納される

## ビリングモード

- オンデマンドキャパシティー
- プロビジョンドキャパシティー
  - 読み取りキャパシティーユニットと書き込みキャパシティーユニットを設定する必要がある
  - ApplicationAutoScaling を使用してキャパシティーユニットをスケールできる

## テーブルクラス

- DynamoDB Standard（標準）
  - 頻繁にアクセスされるデータに最適
  - ストレージコスト: 高い
  - データアクセスコスト: 低い
- DynamoDB Standard-Infrequent Access (DynamoDB Standard-IA)
  - 滅多にアクセスされないデータに最適
  - ストレージコスト: 低い
  - データアクセスコスト: 高い

## プライマリーキー

- パーティションキー（必須）
- ソートキー（任意）

## データ型

- スカラ型
  - S: 文字列
  - N: 数値
  - B: バイナリー
  - NULL: null
  - BOOL: boolean
- ドキュメント型
  - Map: マップ
  - List: 配列
- セット型
  - SS: 文字列セット
  - NS: 数値セット
  - BS: バイナリーセット

## TTL

- テーブルに TTL を設定できる（Unix エポック秒を格納する属性を指定する必要がある）
- TTL は PutItem や UpdateItem で属性に TTL を指定する必要がある
- DynamoDB が期限になると自動的に数日以内に削除する
- 期限になってすぐ削除されるわけではないため、TTL を過ぎた項目をフィルターしたクエリを作成して実行する必要がある

## グローバルセカンダリーインデックス

任意の属性を指定してテーブルに対するコピーを作成可能。テーブルに対してはパーティションキーとソートキー以外での検索は Scan という非効率な検索方法しか存在しないため、別の属性をパーティションキーとソートキー（任意）にしてテーブルのコピーを作成することで効率的な検索を実現可能。

- テーブルのコピー
- 新しい属性をプライマリーキーとする（ただしプライマリーキーの値が重複しても良い）
- 強力な整合性読み取り使用不可
- 属性の射影オプション
  - KYES_ONLY: テーブルソースのパーティションキーとソートキーの値、およびインデックスキーの値のみ
  - INCLUDE: KEYS_ONLY の属性に加えて、セカンダリインデックスにその他の非キー属性が含まれるように指定できる
  - ALL: テーブルソースの全ての属性

## ローカルセカンダリーインデックス

テーブルソースと同一のパーティションキーと任意の属性のソートキーを指定してテーブルソースのコピーを作成可能。例えばテーブルソースのソートキーは登録日時だが更新日時順に検索したい場合はソートキーを更新日時にしてローカルセカンダリーインデックスを作成することで更新日時順のクエリを実行できる。

- テーブルのコピー
- テーブルソースとパーティションキーは同一
- ソートキーは必須
- 属性の射影オプション
  - KYES_ONLY: テーブルソースのパーティションキーとソートキーの値、およびインデックスキーの値のみ
  - INCLUDE: KEYS_ONLY の属性に加えて、セカンダリインデックスにその他の非キー属性が含まれるように指定できる
  - ALL: テーブルソースの全ての属性

## 読み取りクエリ

基本的にパーティションキーとソートキーに対してしか条件を付与できない（だからこそグローバルセカンダリーインデックスが必要）

- GetItem
  - プライマリーキーを指定して単一の項目を取得する
  - グローバルセカンダリーインデックスに対しては使用不可（プライマリーキーが重複して複数の項目を取得する可能性があるから）
- BatchGetItem
  - 1 度の API 呼び出しで複数の GetItem を実行する
- Query
  - パーティションキーを指定して異なるソートキーの複数の項目を取得する
  - パーティションキー: 必須/等価条件(=)での比較のみ
  - ソートキー: 任意/範囲比較なども可能
  - グローバルセカンダリーインデックスに対して使用可能
  - パーティションキーとソートキーを指定して項目取得後にフィルター式にパーティションキーとソートキー以外の属性を指定して項目を絞り込むこともできる（ただしパフォーマンスが劣化する可能性あり）
- Scan
  - テーブルの全ての項目をスキャンする（非効率）

## 読み込み整合性

DynamoDB は各データを 3 つのパーティションに書き込むため必ずしも取得したデータが最新とは限らない。

- 結果整合性のある読み取り
  - 1 つのレプリカから読み取りを行い即座に結果を返す
  - 最新のデータでない可能性がある
- 強力な整合性のある読み取り
  - 3 つのパーティションサーバーから返却された最新のタイムスタンプの値を採用する
  - 結果整合性のある読み取りの倍の読み取りキャパシティーユニットを消費する

## 書き込みクエリ

- PutItem
  - Upsert を実行する
  - プライマリーキーの指定は必須
- UpdateItem
  - Upsert を実行する
  - プライマリーキーの指定は必須
  - 特定の項目のみ更新可能
- DeleteItem
  - Delete を実行する
  - プライマリーキーの指定は必須
- BatchWriteItem
  - 1 度の API 呼び出しで複数の PutItem/UpdateItem/DeleteItem を実行する

## 条件式

条件式に一致しない場合は書き込みクエリを失敗させることができる（RDB の排他制御の代わりに使用する）

## 書き込み整合性

DynamoDB は書き込み API を呼び出してから少なくとも 2 つの AZ のパーティションに書き込みが完了した時点で書き込みを成功とみなしクライアントにレスポンスを返す。

## トランザクション

DynamoDB のトランザクションの分離レベルは Serializable。よって複数のトランザクションが同一の項目に対して書き込みまたは読み込みをする場合はロックの開放を待つ必要がある。

以下にクライアント A が先にトランザクションを開始しクライアント B が後から同じ項目に書き込みまたは読み込みを実行した場合の挙動パターンを示す。

| クライアント A           | クライアント B           | 結果                                        |
| :----------------------- | :----------------------- | :------------------------------------------ |
| トランザクション書き込み | トランザクション書き込み | クライアント B はロック開放まで書き込み待機 |
| トランザクション書き込み | トランザクション読み込み | クライアント B はロック開放まで読み込み待機 |
| トランザクション書き込み | 書き込み                 | クライアント B は即書き込む                 |
| トランザクション書き込み | 読み込み                 | クライアント B は即読み込む                 |
| トランザクション読み込み | トランザクション書き込み | クライアント B はロック開放まで書き込み待機 |
| トランザクション読み込み | トランザクション読み込み | クライアント B はロック開放まで読み込み待機 |
| トランザクション読み込み | 書き込み                 | クライアント B は即書き込む                 |
| トランザクション読み込み | 読み込み                 | クライアント B は即読み込む                 |

- TransactionWriteItems
  - 複数の PutItem/UpdateItem/DeleteItem をトランザクションで実行可能
- TransactionGetItems
  - 複数の GetItem をトランザクションで実行可能

## トランザクションの分離レベル

| オペレーション | 分離レベル                        |
| :------------- | :-------------------------------- |
| DeleteItem     | SERIALIZABLE/通常書き込み         |
| PutItem        | SERIALIZABLE/通常書き込み         |
| UpdateItem     | SERIALIZABLE/通常書き込み         |
| BatchWriteItem | 通常書き込み                      |
| GetItem        | SERIALIZABLE/コミット済み読み取り |
| BatchGetItem   | コミット済み読み取り              |
| Query          | コミット済み読み取り              |
| Scan           | コミット済み読み取り              |