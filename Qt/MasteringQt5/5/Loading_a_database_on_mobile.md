# モバイルでのデータベースの読み込み

UIの実装を続ける前に、モバイルでのデータベースの展開に気をつけなければなりません。ネタバレです。これでは面白くないでしょう。

gallery-coreプロジェクトのDatabaseManager.cppにジャンプして戻る必要があります。

```C++
DatabaseManager& DatabaseManager::instance()
{
    return singleton;
}
DatabaseManager::DatabaseManager(const QString& path) :
    mDatabase(new QSqlDatabase(QSqlDatabase::addDatabase("QSQLITE"))),
    albumDao(*mDatabase),
    pictureDao(*mDatabase)
{
    mDatabase->setDatabaseName(path);
    ...
}
```

デスクトップでは、SQLite3 データベースは mDatabase->setDatabaseName() という命令で作成されますが、モバイルでは全く動作しません。これは、モバイルプラットフォーム（Android と iOS）ごとにファイルシステムが非常に特殊であるという事実に起因しています。アプリケーションは、ファイルシステムの他の部分に干渉できない狭いサンドボックスにしかアクセスできません。アプリケーションディレクトリ内のすべてのファイルは、特定のファイルパーミッションを持っていなければなりません。SQLite3にデータベースファイルを作成させると、そのファイルは適切なパーミッションを持っておらず、OSはデータベースを開くことをブロックしてしまいます。

その結果、データベースが正しく作成されず、データが永続化されません。ネイティブAPIを使用している場合は、データベースの適切な設定はOSが行ってくれるので問題ありません。Qtを使って開発しているので、このAPIに簡単にアクセスすることはできません（JNIなどのブラックマジックを使う場合を除く）。回避策としては、アプリケーションのパッケージに「すぐに使える」データベースを埋め込んで、それを正しいファイルシステムのパスに正しい権限でコピーすることです。

このデータベースには、内容のない空の作成されたデータベースが含まれているはずです。このデータベースは、この章のソースコードにあります（第4章「デスクトップUIの征服」のソースコードから生成することもできます）。gallery.qrcファイルに追加することができます。

レイヤーが明確に定義されているので、この場合はDatabaseManager::instance()の実装を変更して処理する必要があります。

```C++
DatabaseManager& DatabaseManager::instance()
{
#if defined(Q_OS_ANDROID) || defined(Q_OS_IOS)
    QFile assetDbFile(":/database/" + DATABASE_FILENAME);
    QString destinationDbFile = QStandardPaths::writableLocation(
        QStandardPaths::AppLocalDataLocation)
        .append("/" + DATABASE_FILENAME);

        if (!QFile::exists(destinationDbFile)) {
            assetDbFile.copy(destinationDbFile);
            Qfile::setPermissions(destinationDbFile,
            QFile::WriteOwner | QFile::ReadOwner);
        }
    }
    static DatabaseManager singleton(destinationDbFile);
#else
    static DatabaseManager singleton;
#endif
    return singleton;
}
```

まず最初に、便利なQtクラスを使ってアプリケーションのプラットフォーム固有のパスを取得します。QStandardPathsです。このクラスは、複数のタイプ（AppLocalDataLocation、DocumentsLocation、PicturesLocationなど）のパスを返します。データベースはアプリケーションデータディレクトリに格納されている必要があります。存在しない場合はアセットからコピーします。

最後に、OSがデータベースのオープンをブロックしないようにファイルのパーミッションを修正します(パーミッションが十分に制限されていないため)。最後に、OSがデータベースのオープンをブロックしないようにファイルのパーミッションを変更します。

すべてが完了すると、DatabaseManagerシングルトンが正しいデータベースファイルパスでインスタンス化され、コンストラクタはこのデータベースを透過的に開くことができます。

***

## Info

iOS シミュレータでは、QStandardPaths::writableLocation()関数が適切なパスを返しません。iOS 8以降、ホスト上のシミュレータのストレージパスが変更されており、Qtには反映されていません。詳しくは <https://bugreports.qt.io/browse/QTCREATORBUG-13655> をご覧ください。

***

これらの回避策は些細なことではありませんでした。これは、モバイル上でのクロスプラットフォームアプリケーションの限界を示しています。それぞれのプラットフォームには、ファイルシステムの扱い方やコンテンツのデプロイ方法が固有のものがあります。QMLでプラットフォームに依存しないコードを書くことができたとしても、OS間の違いに対処しなければなりません。

***

**[戻る](../index.html)**
