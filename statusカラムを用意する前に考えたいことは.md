

statusカラムと題に載せているが、複数の定数を選択肢として持つような所謂[enum](https://d.hatena.ne.jp/keyword/enum)型のカラムに関することを述べていく（statusという名で定義されることが多いような気がしたので）

そのようなstatus系のカラムは楽で便利が故によく提案されると思うのだが、追加する前にちょっと整理・考慮しておきたい点がある。

因みにアンチstatusカラムという訳ではなく、メリデメをしっかり議論されていればOKな立ち位置なのである

## そのカラムを使ってどのような[SQL](https://d.hatena.ne.jp/keyword/SQL)を構成する予定か

まずはstatusカラムを使ってどのような[SQL](https://d.hatena.ne.jp/keyword/SQL)を書く予定があげてみよう。 意外にもそのカラム追加によって[SQL](https://d.hatena.ne.jp/keyword/SQL)パフォーマンスに影響を与えることがある。

### where条件によく現れますか？

```sql
select name
from users
where status = 'active'
and (another condition)
```

usersというユーザのテーブルに"active"（アクティブ）、"inactive"（非アクティブ）、"pending"（保留）の選択肢のstatusカラムを追加したかったとする。 また、上記のようなstatusカラムをwhere条件で記述する[SQL](https://d.hatena.ne.jp/keyword/SQL)を書く予定があったとする。

ここで考えたいのは<mark>「このwhere条件は頻繁について回る可能性があるのではないか」</mark>ということである。つまり、参照する際に常にstatusカラムの値を気にする必要があるか、ということである。今回の例のようなusersのアクティブ性を問うカラムであれば、常に評価する必要がありそうに感じる。

もしも該当するのであれば、クエリパフォーマンスや今後の対応に難が出る可能性が高い。 常に評価する必要があるので、既存もしくは今後の[SQL](https://d.hatena.ne.jp/keyword/SQL)にも調整が必要になる。よってそれらの[SQL](https://d.hatena.ne.jp/keyword/SQL)にstatusカラムにフィルタリングが発生してしまう（カバ[リングイン](https://d.hatena.ne.jp/keyword/%A5%EA%A5%F3%A5%B0%A5%A4%A5%F3)デックスだった[SQL](https://d.hatena.ne.jp/keyword/SQL)もあるかもしれない）。これは、フェッチしてきた行数が多ければ多いほどダメージはデカい。

statusカラムにインデックスを貼るという判断になったとしても、「このwhere条件は頻繁について回る可能性がある」のであれば、今後のインデックスにstatusカラムを毎回考慮する必要がある。また、[enum](https://d.hatena.ne.jp/keyword/enum)であるが故にカーディナリティは高くないので、そこまで貢献してくれるインデックスにはなり得ない。

## そのカラムに関わる定義変更が起こる可能性があるか

次にそのカラム追加をした後後の[スキーマ](https://d.hatena.ne.jp/keyword/%A5%B9%A5%AD%A1%BC%A5%DE)の姿をイメージしよう。

### 前提
リレーショナルデータベースではテーブル論理設計は「実世界の切り取り」というような表現をされることがある（厳密にリレーション等々の言葉を使うべきだが、便宜上一旦は置いておく）。[スキーマ](https://d.hatena.ne.jp/keyword/%A5%B9%A5%AD%A1%BC%A5%DE)という枠組み自身は本来時間と共に不変であり、中身は時間と共に変化をする。

例えばusersのnameというカラム自体は時間と共に不変であるが、usersレコードのnameの値は時間と共に変化する可能性はある。後々になってユーザ名を変えた経験は誰だってあるはず。

つまりカラムは時間変化の軸となるものである。 例えばusersにname,emailというカラムがあったとする。時間経過とともに各レコードのname,emailの値は変わる。ここにstatusというカラムを追加すれば、もう１つ軸が増えて時間経過とともに変わる要素が増えるのである（リレーションは[ドメイン](https://d.hatena.ne.jp/keyword/%A5%C9%A5%E1%A5%A4%A5%F3)の直積で表されることをナントナク言い換えている）

ちょっと大袈裟に聞こえるかもしれないが、軸が増えたり減ったり変更されると、そのテーブルの概念や意味が大きく変わる。それに伴ってテーブルの扱いも変わる訳である。そのため、カラム追加を始め、[スキーマ](https://d.hatena.ne.jp/keyword/%A5%B9%A5%AD%A1%BC%A5%DE)に対する変更はやや慎重になっていいかもしれない（そのくらいの構えが意外と丁度よいかも）

以降はそんな[スキーマ](https://d.hatena.ne.jp/keyword/%A5%B9%A5%AD%A1%BC%A5%DE)変更に対する考慮である。

### そのカラムに依存するカラムは増えますか？

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL, --顧客ID
    order_date DATE NOT NULL, --注文日
    total_amount DECIMAL(10, 2) NOT NULL, --注文金額
    status ENUM('placed', 'shipped', 'delivered') NOT NULL, --ステータス
    delivery_received_date DATE , --受取日
    delivery_date TEXT, --配達日
);
```

上記のように、ordersという注文テーブルに"placed"（注文済み）、"shipped"（出荷済み）、"delivered"（配達済み）の選択肢のstatusカラムを追加したかったとする。

ここでdelivery_received_date（受取日）、delivery_date（配達日）は、その言葉の意義からもわかるように、statusがdelivered（配達済）の時のみしか値が入らないのである。このように
<mark>「statusカラムの選択肢に依存してしまうカラム</mark>が複数追加されないか」を考慮する必要がある。

今回の例の２つのカラムは、以下のようにstatusが'placed', 'shipped'の場合は<mark>NULL</mark>となってしまう。すなわち、statusが'placed', 'shipped'の時は考慮する必要のないカラムになってしまう。

| order_id | status    | delivery_received_date | delivery_date |
| -------- | --------- | ---------------------- | ------------- |
| 1        | placed    | NULL                   | NULL          |
| 2        | shipped   | NULL                   | NULL          |
| 3        | delivered | 2023-09-14             | 2023-09-13    |
| 4        | canceled  | NULL                   | NULL          |
| 5        | placed    | NULL                   | NULL          |


### そのカラムの選択肢は今後増える可能性はありますか？

[SQL](https://d.hatena.ne.jp/keyword/SQL)[アンチパターン](https://d.hatena.ne.jp/keyword/%A5%A2%A5%F3%A5%C1%A5%D1%A5%BF%A1%BC%A5%F3)の31 Flavorsとやや重複するが

[www.oreilly.co.jp](https://www.oreilly.co.jp/books/9784873115894/)

先ほどの注文テーブルのstatusカラムを導入して、後々にまた新たな状態を用意したい.....と思ったとする。 この欲求の度に、[スキーマ](https://d.hatena.ne.jp/keyword/%A5%B9%A5%AD%A1%BC%A5%DE)変更（ALTER TABLE）する必要がもちろんある。 [ENUM](https://d.hatena.ne.jp/keyword/ENUM)の定義変更は[メタデータ](https://d.hatena.ne.jp/keyword/%A5%E1%A5%BF%A5%C7%A1%BC%A5%BF)変更のみだったりするが、これらの変更コストが避けられるなら避けたいものです。

また、前提でも書いたように、状態が増えるという事はテーブルで管理される責務や意味合いが膨らむ事に繋がる。 それによって、このテーブルの抱えるレコード数やカラム数が増えることにつながる恐れがある（１つ前の依存カラムが増える恐れもある）。

このように 「**今後その選択肢の増減する可能性はあるのか**」 ということを考慮すると良いかもしれない。

## じゃあどうしようか

### テーブル分割とかどうですか

以上のような点に対して、有効と思われる打開策は何であろうか。 自分がよく提案するのは
<mark>「状態ごとのテーブル分割」</mark>である。 つまり、状態をカラムで管理するのではなくテーブル単位で管理する方針である。

【メリット】
- テーブルで状態が分けられてるが故に、**状態をwhere条件に持つ必要がない**。
- 状態に依存するカラムはその状態に**該当するテーブルでのみ管理**すれば良くなる。 
- **新たな状態を用意しても、新規テーブル作成で済む。**

### 実際どんな感じで分割？

では、実際どんな感じでテーブル分割するのかだが、 これにはいくつか方法があり、
<mark>スーパータイプとサブタイプに分ける形</mark>と<mark>サブタイプのみで分ける形</mark>がある。 ※概念設計の段階で登場する用語であるが便宜上用いる

スーパータイプとは共通の要素を持つ複数のエンティティからその共通要素を抜き出したものであり、サブタイプとはそれぞれのエンティティ独自の要素を持つものを言う（汎化と特化などと言う）。[こちらのhmatsuさんの記事](https://qiita.com/hmatsu47/items/19c1d6bf636f38234680)が非常にわかりやすい

先ほどの注文テーブル（statusカラムを入れた状態）を例にとる。

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL, --顧客ID
    order_date DATE NOT NULL, --注文日
    total_amount DECIMAL(10, 2) NOT NULL, --注文金額
    status ENUM('placed', 'shipped', 'delivered') NOT NULL, --ステータス
    delivery_received_date DATE , --受取日
    delivery_date TEXT, --配達日
);
```


### スーパータイプとサブタイプで分ける

共通の要素が多ければこちらの方が合致するかもしれない。
```sql
--スーパータイプ
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL, --顧客ID（各ステータスの共通の情報）
);

--サブタイプ
CREATE TABLE placed_orders (
    order_id INT PRIMARY KEY,
    order_date DATE NOT NULL, --注文日（注文済みステータスの情報）
    total_amount DECIMAL(10, 2) NOT NULL, --注文金額（注文済みステータスの情報）
);

--サブタイプ
CREATE TABLE shipped_orders (
    order_id INT PRIMARY KEY,
);

--サブタイプ
CREATE TABLE delivered_orders (
    order_id INT PRIMARY KEY,
    delivery_received_date DATE,--受取日（配達済みステータスの情報）
    delivery_date TEXT--配達日（配達済みステータスの情報）
);
```


場合によっては、注文金額と注文日は共通処理で用いるのであればスーパータイプの方で管理しても良いかもしれない。ここら辺は議論のしどころ。

### サブタイプのみで分ける

各状態の独立性が高い、つまり各状態に依存する情報や処理が多いのであればこちらの方が合致するかもしれない。
```sql
--サブタイプ
CREATE TABLE placed_orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL, --顧客ID
    order_date DATE NOT NULL, --注文日（注文済みステータスの情報）
    total_amount DECIMAL(10, 2) NOT NULL, --注文金額（注文済みステータスの情報）
);

--サブタイプ
CREATE TABLE shipped_orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL, --顧客ID
);

--サブタイプ
CREATE TABLE delivered_orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL, --顧客ID
    delivery_received_date DATE,--受取日（配達済みステータスの情報）
    delivery_date TEXT--配達日（配達済みステータスの情報）
);
```

## 最後に

もちろん他にもいい案があるかもしれないし、いろいろな都合が出没するかもしれない。 ここで言いたいのはあくまでメリデメしっかり考慮して議論した上で導入しようぜ〜ということである。

他にもstatusカラムの懸念だったり、いい案があったらぜひ教えてください。