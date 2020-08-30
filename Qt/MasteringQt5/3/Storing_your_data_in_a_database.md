# データをデータベースに保存する

データクラスの準備ができたので、データベース層の実装に進みます。Qtにはすぐに使えるsqlモジュールが用意されています。Qtでは、SQLデータベースドライバを使って様々なデータベースをサポートしています。gallery-desktopでは、sqlモジュールに含まれているSQLITE3ドライバを使用します。

* **非常にシンプルなデータベーススキーマ**: 複雑なクエリは必要ありません。
* **並行トランザクションが非常に少ない、または全くない**: 複雑なトランザクションモデルは不要。
* **単一目的のデータベースです**: システムサービスを起動する必要がなく、データベースは単一のファイルに格納され、複数のアプリケーションからアクセスする必要がありません。

データベースには複数の場所からアクセスすることになります。そのためには、単一のエントリ・ポイントが必要です。DatabaseManagerという名前の新しいC++クラスを作成し、DatabaseManager.hを以下のように変更します。

```C++
#include <QString>

class QSqlDatabase;

const QString DATABASE_FILENAME = "gallery.db";

class DatabaseManager
{
public:
    static DatabaseManager& instance();
    ~DatabaseManager();

protected:
    DatabaseManager(const QString& path = DATABASE_FILENAME);
    DatabaseManager& operator=(const DatabaseManager& rhs);

private:
    QSqlDatabase* mDatabase;
};
```

最初に注意すべき点は、第2章「QMakeの秘密を探る」の「SysInfoをシングルトンで変換する」で行ったように、DatabaseManagerクラスにシングルトンパターンを実装していることです。DatabaseManagerクラスは、mDatabaseフィールドで接続を開き、他の使用する可能性のあるクラスに貸し出します。

また、QSqlDatabaseは前方に宣言され、mDatabaseフィールドのポインタとして使用されます。
QSqlDatabaseヘッダを含めることもできましたが、望ましくない副作用が発生してしまいます: DatabaseManagerをインクルードするすべてのファイルに、QSqlDatabaseをインクルードしてしまいます。そのため、アプリケーション内に（gallery-coreライブラリにリンクする）他動的なインクルージョンがある場合、アプリケーションはSQLモジュールを有効にしなければなりません。結果として、ストレージレイヤはライブラリを介してリークします。アプリケーションはストレージレイヤの実装について何も知らないはずです。アプリケーションが気にすることは、SQLでもXMLでも何でもいいのです。ライブラリは、設計を尊重し、データを永続化すべきブラックボックスです。

DatabaseManager.cppに切り替えて、データベース接続を開いてみましょう。

```C++
#include "databasemanager.h"

#include <QSqlDatabase>

DatabaseManager &DatabaseManager::instance()
{
    static DatabaseManager singleton;
    return singleton;
}

DatabaseManager::~DatabaseManager()
{
    mDatabase->close();
    delete mDatabase;
}

DatabaseManager::DatabaseManager(const QString &path) :
    mDatabase(new QSqlDatabase(QSqlDatabase::addDatabase("QSQLITE")))
{
    mDatabase->setDatabaseName(path);
    mDatabase->open();
}
```

QSqlDatabase::addDatabase("QSQLITE")関数呼び出しによるmDatabaseフィールドの初期化で正しいデータベースドライバが選択されます。以下の手順は、データベース名（ちなみにこれはSQLITE3のファイルパスです）を設定し、mDatabase->open()関数で接続を開くだけです。DatabaseManagerのデストラクタでは、接続が閉じられ、mDatabaseポインタが適切に削除されます。

データベースのリンクが開かれました。あとは、AlbumとPictureのクエリを実行するだけです。DatabaseManager の両方のデータクラスに CRUD (Create/Read/Update/Delete) を実装すると、DatabaseManager.cpp はあっという間に数百行になります。さらにいくつかのテーブルを追加すると、DatabaseManagerがどのような怪物になるか、もうおわかりでしょう。

このため、各データクラスには専用のデータベースクラスがあり、すべてのデータベース CRUD 操作を担当します。まず、Album クラスから始めます。新しい C++ クラス名 AlbumDao （データアクセスオブジェクト）を作成し、AlbumDao.h を更新します。

```C++
class QSqlDatabase;

class AlbumDao
{
public:
    AlbumDao(QSqlDatabase& database);
    void init() const;

private:
    QSqlDatabase& mDatabase;
};
```

AlbumDao クラスのコンストラクタは QSqlDatabase& パラメータを取ります。このパラメータは、 AlbumDao クラスが行うすべての SQL クエリに使用されるデータベース接続です。
init() 関数は albums テーブルを作成することを目的としており、 mDatabase が開かれたときに呼び出される必要があります。

AlbumDao.cppの実装を見てみましょう。

```C++
#include "albumdao.h"

#include <QSqlDatabase>
#include <QSqlQuery>
#include <QStringList>

#include "databasemanager.h"

AlbumDao::AlbumDao(QSqlDatabase &database) :
    mDatabase(database)
{
}

void AlbumDao::init() const
{
    if (!mDatabase.tables().contains("albums")) {
        QSqlQuery query(mDatabase);
        query.exec("CREATE TABLE albums (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT)");
    }
}
```

いつものように、mDatabaseフィールドはデータベースパラメータで初期化されます。init()関数では、実際のSQLリクエストの動作を見ることができます。テーブルalbumsクラスが存在しない場合、mDatabase接続を実行するためのQSqlQueryクエリが作成されます。mDatabase を省略した場合、クエリはデフォルトの匿名接続を使用します。query.exec() 関数は、クエリを実行する最もシンプルな方法です。クエリのQString型を渡すだけで実行されます。ここでは、データクラス Album (id と name) にマッチするフィールドを持つ albums テーブルを作成します。

## Tip

QSqlQuery::exec() 関数は、リクエストが成功したかどうかを示す bool 値を返します。実運用コードでは、常にこの値をチェックしてください。QSqlQuery::lastError() を使用して、さらにエラーを調べることができます。例は、DatabaseManager::debugQuery() の章のソースコードにあります。

AlbumDao クラスの骨格ができました。次のステップは、DatabaseManager クラスにリンクすることです。DatabaseManager クラスを次のように更新します。

```C++
// In DatabaseManager.h

#include "albumdao.h"

...

private:
    QSqlDatabase* mDatabase;

public:
    const AlbumDao albumDao;
};

// In DatabaseManager.cpp
DatabaseManager::DatabaseManager(const QString &path) :
    mDatabase(new QSqlDatabase(QSqlDatabase::addDatabase("QSQLITE"))),
    albumDao(*mDatabase)
{
    mDatabase->setDatabaseName(path);
    mDatabase->open();

    albumDao.init();
}
```

albumDao フィールドは DatabaseManager.h ファイル内で public const AlbumDao として宣言されています。これには説明が必要です 。

* public の可視性は、DatabaseManager クライアントに albumDao フィールドへのアクセスを与えることです。API は十分に直感的になります。もし album でデータベース操作を行いたい場合は、DatabaseManager::instance().albumDao を呼び出すだけです。

* const キーワードは、誰も albumDao を変更できないようにするためのものです。これはpublicなので、オブジェクトの安全性を保証することはできません（誰でもオブジェクトを変更することができます）。副作用として、AlbumDao のすべての公開関数を const にすることを強制しています。これは理にかなっています。
結局のところ、 AlbumDao フィールドは、たくさんの関数を持つ名前空間である可能性があります。mDatabase フィールドでデータベース接続への参照を保持できるので、クラスである方が便利です。

DatabaseManagerのコンストラクタでは、albumDaoクラスは、mDatabase派生ポインタで初期化されます。albumDao.init() 関数は、データベース接続が開かれた後に呼び出されます。

私たちは今、より興味深い SQL クエリを実装するために進むことができます。私たちは、AlbumDaoクラスで新しいアルバムを作成することから始めることができます。

```C++
// In AlbumDao.h

class QSqlDatabase;
class Album;

class AlbumDao
{
public:
    AlbumDao(QSqlDatabase& database);
    void init() const;
    void addAlbum(Album& album) const;
    ...
};

// In AlbumDao.cpp

#include <QSqlDatabase>
#include <QSqlQuery>
#include <QStringList>
#include <QVariant>

...

void AlbumDao::addAlbum(Album &album) const
{
    QSqlQuery query(mDatabase);
    query.prepare("INSERT INTO albums (name) VALUES (:name)");
    query.bindValue(":name", album.name());
    query.exec();
    album.setId(query.lastInsertId().toInt());
}
```

関数 addAlbum() は album パラメータを受け取り、その情報を抽出して対応するクエリを実行します。ここでは、準備されたクエリの概念にアプローチします。
query.prepare() 関数は query パラメータを受け取ります。ここでは、nameパラメータに :name という構文を指定します。
二つの構文がサポートされています。コロン-名前を使用した Oracle スタイル (例 :name) と、クエスチョンマークを使用した ODBC スタイル (例 ?name) の 2 つです。

次に、:name 構文を album.name() 関数の値にバインドします。
QSqlQuery::bind()はパラメータ値としてQVariantを期待しているので、このクラスにincludeディレクティブを追加する必要があります。

一言で言えば、QVariantは、広範囲のプリミティブ型（char、int、doubleなど）や複雑な型（QString、QByteArray、QUrlなど）を受け付ける汎用データホルダーです。

query.exec() 関数が実行されると、バインドされた値が適切に置き換えられます。prepare()文のテクニックを使うと、SQLインジェクションに強くなり(隠されたリクエストを注入すると失敗します)、コードがより読みやすくなります。

クエリの実行は、オブジェクトクエリ自体の状態を変更します。QSqlQueryクエリは、単にSQLクエリの実行者ではなく、アクティブなクエリの状態も含んでいます。query.lastInsertId()関数を使用して、クエリに関する情報を取得することができます。この ID は、addAlbum() パラメータで指定した album に与えられます。私たちは album を変更するので、パラメータを const としてマークすることはできません。あなたのコードの const の正しさを厳密に守ることは、他の開発者にとって良いヒントになります。

残りの更新および削除操作は、addAlbum() で使用したのと厳密に同じパターンに従います。次のコードスニペットでは、期待される関数のシグネチャを提供するだけです。完全な実装については、この章のソースコードを参照してください。しかし、データベース内のすべてのアルバムを取得するリクエストを実装する必要があります。これはよく見てみる価値があります。

```C++
// In AlbumDao.h

#include <QVector>

    ...
    void addAlbum(Album& album) const;
    void updateAlbum(const Album& album) const;
    void removeAlbum(int id) const;
    QVector<Album*> albums() const;
    ...
};

// In AlbumDao.cpp

QVector<Album *> AlbumDao::albums() const
{
    QSqlQuery query("SELECT * FROM albums", mDatabase);
    query.exec();
    QVector<Album*> list;
    while (query.next()) {
        Album* album = new Album();
        album->setId(query.value("id").toInt());
        album->setName(query.value("name").toString());
        list.append(album);
    }
    return list;
}
```

albums()関数はQVector\<Album*\>値を返さなければなりません。関数の本体を見てみると、QSqlQueryのもう一つのプロパティがあります。与えられたリクエストの複数の行をウォークスルーするために、queryは現在の行を指す内部カーソルを処理します。次に、new Album*() 関数を作成し、 query.value() ステートメントで行データを埋めます。この新しい album パラメータが list に追加され、最後にこの list が呼び出し元に返されます。

PictureDaoクラスは、使い方も実装もAlbumDaoクラスに非常に似ています。主な違いは、ピクチャがアルバムへの外部キーを持っていることです。PictureDao関数は、albumIdパラメータによって条件付けされなければなりません。次のコードスニペットは、PictureDaoヘッダとinit()関数を示しています。

```C++
// In PictureDao.h

#include <QVector>

class QSqlDatabase;
class Picture;

class PictureDao
{
public:
    explicit PictureDao(QSqlDatabase& database);
    void init() const;

    void addPictureInAlbum(int albumId, Picture& picture) const;
    void removePicture(int id) const;
    void removePictureForAlbum(int albumId) const;
    QVector<Picture*> pictureForAlbum(int albumId) const;

private:
    QSqlDatabase& mDatabase;
};

// In PictureDao.cpp

void PictureDao::init() const
{
    if (!mDatabase.tables().contains("pictures")) {
        QSqlQuery query(mDatabase);
        query.exec(QString("CREATE TABLE pictures")
                   + " (id INTEGER PRIMARY KEY AUTOINCREMENT, "
                   + "album_id INTEGER, "
                   + "url TEXT");
    }
}
```

ご覧のように、複数の関数は、画像と所有するalbumパラメータの間のリンクを作るために、albumIdパラメータを取ります。init()関数では、外部キーはalbum_id INTEGER構文で表現されます。SQLITE3には適切な外部キー型がありません。これは非常にシンプルなデータベースであり、このタイプのフィールドには厳密な制約はありません。単純な整数が使用されます。

最後に、AlbumDaoで行ったのと同じように、DatabaseManagerクラスにPictureDao関数を追加します。多くの Dao クラスがある場合、DatabaseManager クラスに const Dao メンバを追加して init() 関数を呼び出すのは、すぐに面倒になるという議論もあるかもしれません。

解決策としては、純粋な仮想 init() 関数を持つ抽象的な Dao クラスを作ることが考えられます。DatabaseManager クラスは Dao レジストリを持ち、各 Dao を QHash<QString, const Dao> mDaos で QString キーにマッピングします。init() 関数呼び出しは for ループ内で呼び出され、QString キーを使用して Dao オブジェクトにアクセスされます。これはこのプロジェクトの範囲外ですが、興味深いアプローチです。

***
**[戻る](../index.html)**
