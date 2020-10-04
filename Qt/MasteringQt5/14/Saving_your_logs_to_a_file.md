# ログをファイルに保存する

開発者にとって一般的なニーズは、ログを持っていることです。状況によっては、コンソール出力にアクセスできなかったり、アプリケーションの状態を後から調査しなければならないことがあります。どちらの場合も、ログをファイルに出力する必要があります。

Qtは、ログ（qDebug、qInfo、qWarningなど）を自分の都合の良いデバイスにリダイレクトする実用的な方法を提供しています。QtMessageHandlerです。これを使うには、ログを目的の出力に保存する関数を登録する必要があります。

例えば、main.cpp に以下の関数を追加します。

```C++
#include <QFile>
#include <QTextStream>

void messageHander(QtMsgType type,
                   const QMessageLogContext& context,
                   const QString& message) {
    QString levelText;
    switch (type) {
    case QtDebugMsg:
        levelText = "Debug";
        break;
    case QtInfoMsg:
        levelText = "Info";
        break;
    case QtWarningMsg:
        levelText = "Warning";
        break;
    case QtCriticalMsg:
        levelText = "Critical";
        break;
    case QtFatalMsg:
        levelText = "Fatal";
        break;
    }
    QString text = QString("[%1] %2")
                        .arg(levelText)
                        .arg(message);
    QFile file("app.log");
    file.open(QIODevice::WriteOnly | QIODevice::Append);
    QTextStream textStream(&file);
    textStream << text << endl;
}
```

Qtで適切に呼び出されるためには、関数のシグネチャを尊重しなければなりません。パラメータを確認してみましょう。

* QtMsgType type: メッセージを生成した関数(qDebug(), qInfo(), qWarning()など)を記述したenumです。
* QMessageLogContext& context: これには、ログメッセージに関する追加情報（ログが生成されたソースファイル、関数名、行番号など）が含まれます。
* const QString& message: これが実際にログに記録されたメッセージです。

この関数の本体は、ログメッセージをフォーマットしてから app.log という名前のファイルに追加します。この関数では、回転するログファイルを追加したり、ネットワーク経由でログを送信したり、その他の機能を簡単に追加することができます。

最後に足りないのは、messageHandler()の登録ですが、これはmain()関数で行います。

```C++
int main(int argc, char *argv[])
{
    QCoreApplication a(argc, argv);
    qInstallMessageHandler(messageHander);
    ...
}
```

qInstallMessageHander() 関数を呼び出すだけで、すべてのログメッセージを app.log にリルートすることができます。これが完了すると、ログはコンソール出力に表示されなくなり、app.logにのみ追加されます。

***

## Tips

カスタムメッセージハンドラ関数の登録を解除する必要がある場合は、qInstallMessageHandler(0) を呼び出します。

***

**[戻る](../index.html)**
