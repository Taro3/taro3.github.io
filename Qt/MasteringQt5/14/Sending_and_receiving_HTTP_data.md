# HTTPデータの送受信

HTTP サーバに情報を要求するのはよくある作業です。ここでもQtはそれを簡単にするための便利なクラスを用意してくれました。これを実現するために、3つのクラスに頼ることにします。

* QNetworkAccessManager: このクラスは、アプリケーションが要求を送信し、応答を受信することを可能にします。
* QNetworkRequest: このクラスは、送信するリクエストをすべての情報（ヘッダ、URL、データなど）と一緒に保持します。
* QNetworkReply: このクラスは、ヘッダーとデータを持つQNetworkRequestクラスの結果を含みます。

QNetworkAccessManager クラスは、Qt HTTP API 全体の中心点です。これは、クライアントの設定、プロキシ設定、キャッシュ情報などを保持する単一の QNetworkAccessManager オブジェクトを中心に構築されています。このクラスは非同期に設計されているので、現在のスレッドをブロックすることを心配する必要はありません。

それでは、カスタムのHttpRequestクラスで実際に見てみましょう。まず、ヘッダ。

```C++
#include <QObject>
#include <QNetworkAccessManager>
#include <QNetworkReply>

class HttpRequest : public QObject
{
    Q_OBJECT
public:
    HttpRequest(QObject* parent = 0);
    void executeGet();

private slots:
    void replyFinished(QNetworkReply* reply);

private:
    QNetworkAccessManager mAccessManager;
};
```

QNetworkAccessManagerクラスはシグナル/スロットの仕組みで動作するので、HttpRequestはQObjectを継承してQ_OBJECTマクロを使用します。以下の関数とメンバを宣言します。

* executeGet(): これは、HTTP GET リクエストをトリガするために使用されます。
* replyFinished(): これは、GETリクエストが完了したときに呼び出されるスロットです。
* mAccessManager: これはすべての非同期リクエストに使用されるオブジェクトです。

ここでは、HttpRequest.cppのHttpRequestクラスのコンストラクタに注目してみましょう。

```C++
HttpRequest::HttpRequest(QObject* parent) :
    QObject(parent),
    mAccessManager()
{
    connect(&mAccessManager, &QNetworkAccessManager::finished,
            this, &HttpRequest::replyFinished);
}
```

コンストラクタの本体では、mAccessManager からの finished() シグナルを replyFinished() スロットに接続します。これは、mAccessManager を通して送信されるすべてのリクエストが、このスロットをトリガーすることを意味します。

準備はもういいから、実際にリクエストと返事を見てみよう。

```C++
// Request
void HttpRequest::executeGet()
{
    QNetworkRequest request(QUrl("http://httpbin.org/ip"));
    mAccessManager.get(QNetworkRequest(request));
}

// Response
void HttpRequest::replyFinished(QNetworkReply* reply)
{
    int statusCode =
reply->attribute(QNetworkRequest::HttpStatusCodeAttribute).toInt();
    qDebug() << "Reponse network error" << reply->error();
    qDebug() << "Reponse HTTP status code" << statusCode;
    qDebug() << "Reply content:" << reply->readAll();
    reply->deleteLater();
}
```

HTTP GETリクエストは、mAccessManager.get()を使用して処理されます。QNetworkAccessManagerクラスは、他のHTTP動詞（head()、post()、put()、delete()など）のための機能を提供します。これは、コンストラクタで URL を取る QNetworkRequest アクセスを期待しています。これは、HTTP リクエストの最も単純な形式です。

URL <http://httpbin.org/ip> を使用してリクエストを行いましたが、これは JSON 形式でエミッターの IP アドレスに応答します。

```json
{
 "origin": "1.2.3.4"
}
```

このウェブサイトは実用的な開発者向けリソースであり、テストリクエストを送信して有益な情報を送り返してもらうことができます。いくつかのリクエストをテストするだけのために、カスタムのウェブサーバーを立ち上げる必要はありません。このウェブサイトは、Runscope が自由にホストしているオープンソースのプロジェクトです。もちろん、リクエストの URL を任意のものに置き換えることができます。

***

## Info

<http://httpbin.org/> を見て、サポートされているすべてのリクエストタイプを確認してください。

***

executeGet() 関数が完了すると、mAccessManager オブジェクトは別のスレッドでリクエストを実行し、 結果として得られた QNetworkReply* オブジェクトを使用して replyFinished() スロットを呼び出します。このコードスニペットでは、HTTP ステータスコードを取得してネットワークエラーが発生したかどうかを確認する方法と、 reply->readAll() でレスポンスの本文を取得する方法を見ることができます。

QNetworkReplyクラスはQIODeviceを継承しているので、readAll()で一気に読み込んだり、read()でループを使ってチャンク単位で読み込んだりすることができます。これにより、使い慣れた QIODevice API を使用して、必要に応じて読み取りを適応させることができます。

QNetworkReply*オブジェクトの所有者はあなたであることに注意してください。手で削除してはいけません（そうするとアプリケーションがクラッシュする可能性があります）。代わりに、 reply->deleteLater() 関数を使用した方が良いでしょう。

ここでは、HTTP POST メソッドを使用した QNetworkReply のより複雑な例を見てみましょう。QNetworkReply クラスを追跡し、そのライフサイクルをより細かく制御する必要がある場合があります。

ここでは、HttpRequest::mAccessManager にも依存する HTTP POST メソッドの実装を示します。

```C++
void HttpRequest::executePost()
{
    QNetworkRequest request(QUrl("http://httpbin.org/post"));
    request.setHeader(QNetworkRequest::ContentTypeHeader,
                      "application/x-www-form-urlencoded");
    QUrlQuery urlQuery;
    urlQuery.addQueryItem("book", "Mastering Qt 5");

    QUrl params;
    params.setQuery(urlQuery);

    QNetworkReply* reply = mAccessManager.post(
                         request, params.toEncoded());
    connect(reply, &QNetworkReply::readyRead,
        [reply] () {
        qDebug() << "Ready to read from reply";
    });
    connect(reply, &QNetworkReply::sslErrors,
        [this] (QList<QSslError> errors) {
        qWarning() << "SSL errors" << errors;
    });
}
```

カスタムヘッダーを持つQNetworkRequestクラスを作成することから始めます。Content-TypeはHTTP RFCを尊重するためにapplication/x-www-form-urlencodedになりました。その後、URLフォームが作成され、リクエストと一緒に送信する準備ができています。urlQueryオブジェクトには、好きなだけ多くの項目を追加することができます。

次の部分が面白くなってきます。リクエストとURLエンコードされたフォームでmAccessManager.post()を実行すると、すぐにQNetworkReply*オブジェクトが返ってきます。ここからは、mAccessManage スロットを使用するのではなく、直接 reply に接続されたいくつかのラムダス スロットを使用します。これにより、各リプライに対して何が起こるかを正確に制御することができます。

QNetworkReploy::readyReadシグナルは、QIODevice APIから来ており、パラメータにQNetworkReply*オブジェクトを渡さないことに注意してください。どこかのメンバ フィールドにリプライを格納するか、シグナルのエミッタを取得するのはあなたの仕事です。

最後に、このコードスニペットは、mAccessManagerに接続されている前のスロット、replyFinished()を元に戻しません。このコードを実行すると、以下のような出力シーケンスになります。

```shell
Ready to read from reply
Reponse network error QNetworkReply::NetworkError(NoError)
Reponse HTTP status code 200
```

QNetworkReply::readyReadシグナルに接続されたラムダが最初に呼び出され、その後、HttpRequest::replyFinishedシグナルが呼び出されます。

Qt HTTP スタックの最後の機能は、同期リクエストです。リクエストのスレッドを自分で管理する必要がある場合、QNetworkAccessManagerのデフォルトの非同期作業モードが邪魔になることがあります。これを回避するには、カスタム QEventLoop を使用します。

```C++
void HttpRequest::executeBlockingGet()
{
    QNetworkAccessManager localManager;
    QEventLoop eventLoop;
    QObject::connect(
        &localManager, &QNetworkAccessManager::finished,
        &eventLoop, &QEventLoop::quit);

    QNetworkRequest request(
        QUrl("http://httpbin.org/user-agent"));
    request.setHeader(QNetworkRequest::UserAgentHeader,
        "MasteringQt5Browser 1.0");

    QNetworkReply* reply = localManager.get(request);
    eventLoop.exec();

    qDebug() << "Blocking GET result:" << reply->readAll();
    reply->deleteLater();
}
```

この関数では、HttpRequestで宣言したものと干渉しない別のQNetworkAccessManagerを宣言します。直後にQEventLoopオブジェクトを宣言し、localManagerに接続します。QNetworkAccessManagerがfinished()シグナルを出すと、eventLoopが終了し、呼び出し関数が再開されます。

いつものようにrequestがビルドされ、replyオブジェクトが取得され、eventLoop.exec()の呼び出しで関数がブロックされます。この関数は、localManagerが終了信号を出すまでブロックされます。言い換えれば、リクエストはまだ非同期で行われています。唯一の違いは、リクエストが完了するまで関数がブロックされることです。

最後に、関数の最後に reply オブジェクトを安全に読み込んで削除することができます。このQEventLoopトリックは、Qtシグナルの同期待ちが必要なときにいつでも使用できます。UI スレッドをブロックしないように賢く使いましょう!

***

## まとめ

この章では、Qtの知識を完成させるためのヒントを学びました。あなたは今、Qt Creatorを簡単かつ効率的に使用できるようになっているはずです。QDebug形式は今では秘密を保持していないはずで、ログを瞬きもせずにファイルに保存できるようになりました。見栄えの良いCLIインターフェースを作成したり、プログラムのメモリをブレることなくデバッグしたり、自信を持ってHTTPリクエストを実行したりすることができるようになりました。

私たちが本書を書いたのと同じくらい楽しんで読んでいただけたことを心から願っています。私たちの考えでは、Qt は素晴らしいフレームワークであり、本（または何冊かの本！）で深めるに値する多くの分野をカバーしています。効率的で美しく作られたアプリケーションを構築することで、C++ Qt のコードを楽しくコーディングし続けていただけることを願っています。

***

**[戻る](../index.html)**
