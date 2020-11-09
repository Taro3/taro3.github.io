# 細かいことはいいから、とりあえず QSqlDatabase を使ってデータベースにアクセスしたい

環境: Linux Mint 20 + Qt 5.15.1

**[全ソースはここ](https://github.com/Taro3/QSqlDatabaseSample)**

タイトル通り、とりあえずデータベースにアクセスしたい！というサンプルです。
と言っても、現在は SQLite の例だけです。^^;

## 準備

SQL 関連の API を使用する場合、 .pro ファイルの QT に sql を追加する必要があります。

```QMake
QT       += core gui sql
```

## データベースのオープン

QSqlDatabase のインスタンスを得るためには、 QSqlDatabase::addDatabase を使用します。
引数には、データベースタイプを示す文字列を渡します。
ここでは、 "QSQLITE" を渡して、SQLite のデータベースを追加しています。
取得した QSqlDatabase の setDatabaseName を使用してデータベースの名前を指定します。(SQLite ではなく、 MySQL や Postgresql などを使用する場合は、ホスト名やユーザ情報を設定する必要があります)

```C++
    QSqlDatabase db = QSqlDatabase::addDatabase("QSQLITE");
    db.setDatabaseName(DBNAME);
```

データベースが存在しない場合は作成されます。

データベースをオープンします。

```C++
    if (!db.open()) {
        ...
    }
```

## テーブルの作成

SQL を使用してテーブルを作成してみます。

```C++
    static const QString TABLENAME = "test";
    static const QString SQL1 = "CREATE TABLE %1(id INTEGER, name TEXT, memo TEXT)";
    // QSqlQuery q1 = db.exec(SQL1.arg(TABLENAME));
    QSqlQuery q1(db);
    if (!q1.exec(SQL1.arg(TABLENAME))) {
        qDebug() << q1.lastError().text();
        return;
    }
```

QSqlDatabase::exec で直接 SQL を実行することも可能ですが、エラーハンドリングを行うためには QSqlQuery::exec を使用するほうが良いと思います。

QSqlDatabase を引数に渡して、 QSqlQuery のインスタンスを作り、 QSqlQuery::exec を使って SQL 文を実行します。
戻り値が bool で返ってくるので、エラーハンドリングを行います。
エラーハンドリングは、 QSqlQuery::lastError で QSqlError を取得し、 QSqlError::text でエラーメッセージを取得して表示しています。

ここでは、念の為にテーブルの存在を確認しています。

```C++
    if (!db.tables().contains(TABLENAME)) {
        ...
    }
```

QSqlDatabase::tables でテーブル名リストが取得できるので、作成したテーブルが存在しているかを確認しています。

## レコードの INSERT

次は、レコードを挿入してみます。

```C++
    QSqlQuery q2(db);
    static const QString SQL2 = "INSERT INTO %1 VALUES(1, 'taro', 'memo1')";
    if (!q2.exec(SQL2.arg(TABLENAME.arg(TABLENAME)))) {
        ...
    }
```

テーブル作成と同じ方法で INSERT 文を実行しているだけです。

次に、複数のレコードを一気に追加してみます。

```C++
    static const QString SQL3 = "INSERT INTO %1 VALUES(?, ?, ?)";
    QSqlQuery q3(db);
    q3.prepare(SQL3.arg(TABLENAME));
    QVariantList v1 = { 2, 3, 4, 5 };
    q3.addBindValue(v1);
    QVariantList v2 = { "jiro", "saburo", "shiro", "hanako" };
    q3.addBindValue(v2);
    QVariantList v3 = { "memo2", "memo3", "memo4", "memo5" };
    q3.addBindValue(v3);
    if (!q3.execBatch()) {
        qDebug() << q3.lastError();
        return;
    }
```

ここでは、 QSqlQuery::execBatch を使用して複数のレコードを追加しています。
まず、フィールドの値に "?" を指定した SQL 文を QSqlQuery::prepare に渡して準備します。
次に、QVariantList を使用して、 "?" に挿入するデータをフィールごとに QSqlQuery::addBindValue を使って追加していきます。
以上の準備ができたら、 QSqlQuery::execBatch を使って一気に SQL を実行します。

## SELECT でのレコード取得

次は、 SELECT 文でレコードを抽出してみます。

```C++
    static const QString SQL4 = "SELECT * FROM %1";
    QSqlQuery q4(db);
    if (!q4.exec(SQL4.arg(TABLENAME))) {
        qDebug() << q4.lastError();
        return;
    }
    while (q4.next()) {
        QStringList record;
        for (int i = 0; i < q4.record().count(); ++i)
            record << q4.record().value(i).toString();
        qDebug() << record.join(" ");
    }
```

SELECT 文の実行はこれまでの SQL 実行と同じです。
SQL の実行結果を取得するために、 QSqlQuery::next を使ってレコードを順に処理していきます。
QSqlQuery::record が示す　QSQLRecord に現在のレコードの情報が入っています。
QSqlRecord::count にフィールドの数が入ってるので、これを使ってレコード内のデータを走査していきます。
QSqlRecord::value で指定インデックスのフィールド値が QVariant で取得できるので、 QVariant::toString で文字列に変換して表示しています。

## データベースを閉じる

最後に QSqlDatabase::close でデータベースを閉じます。

```C++
    db.close();
```

***

という感じで、ざっくり QSqlDatabase を使ってみました。
ここでは、トランザクションやロールバックを使っていないので、そのうちに Postgresql か MySql でやってみようかな・・・？と思っています。^^;

以上、 **「とりあえず QSqlDatabase を使ってデータベースにアクセスしたい！」** というときの基本でした。

***

**[戻る](../Qt.md)**
