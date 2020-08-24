# シグナルとスロット

Qtフレームワークは、3つの概念を通して柔軟なメッセージ交換メカニズムをもたらします。
シグナル、スロット、コネクションです。

* シグナルは、オブジェクトから送られるメッセージです。
* スロットは、シグナルがトリガされたときに呼び出される関数です。
* connect関数は、どのシグナルがどのスロットにリンクされているかを指定します。

Qtは、最初からクラスのシグナルとスロットを提供していて、アプリケーションで使用することができます。
例えば、QPushButtonにはclocked()というシグナルがあり、ユーザーがボタンをクリックしたときにトリガされます。
QApplicationクラスには、アプリケーションを終了させたいときに呼び出すことができるスロットのquit()関数があります。

Qtのシグナルとスロットが好ましい理由は以下にあります。

* スロットは普通の関数なので、自分で呼び出すことができます。
* 1つのシグナルを複数のスロットにリンクすることができます。
* 1つのスロットは、リンクされた複数のシグナルから呼び出すことができます。
* 異なるオブジェクトからのシグナルとスロットや、さらには異なるスレッド内のオブジェクト間でも接続することができます！

シグナルとスロットを接続するためには、それらのメソッドのシグネチャが一致していなければならないことに注意してください。引数の数、順序、型は同じでなければなりません。また、シグナルとスロットは値を返さないことに注意してください。

以下がQtのコネクト構文です。

```C++
connect(sender, &Sender::signalName, receiver, &Receiver::slotName);
```

この素晴らしいメカニズムを使って最初に行うことは、既存のシグナルと既存のスロットを接続することです。connect呼び出しをMainWindowコンストラクタに追加します。

```C++
MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);
    connect(ui->addTaskButton, &QPushButton::clicked, QApplication::instance(), &QApplication::quit);
}
```

接続がどのように行われているかを分析してみましょう。

* **sender**: シグナルを送信するオブジェクトです。この例では、UIデザイナから追加されたaddTaskButtonという名前のQPushButtonです。
* **&Sender::signalName**: メンバのシグナル関数へのポインタです。ここでは、clockedシグナルが鳥がされたときに何かをしたいと考えています。
* **receiver**: シグナルを受信して処理するオブジェクトです。ここでは、main.cppで作成されたQApplicationオブジェクトです。
* **&Receiver::slotName**: receiverのメンバのスロット関数へのポインタです。この例では、アプリケーションを終了させるQApplicationの組み込みのquit()スロットを使用しています。

これで、この短い例をコンパイルして実行することができます。MainWindowのaddTaskButtonをクリックするとアプリケーションが終了します。

<br>

## Qt tip

シグナルを別のシグナルに接続することができます。最初のシグナルがトリガされると2つ目のシグナルが発せられます。

<br>

シグナルを既存のスロットに接続する方法がわかったので、MainWindowクラスでカスタムのaddTask()スロットを宣言して実装する方法を見てみましょう。このスロットは、ユーザがui->addTaskButtonをクリックしたときに呼び出されます。

以下が、更新されたMainWindow.hです。

```C++
class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

public slots:
    void addTask();

private:
    Ui::MainWindow *ui;
};
```

Qtでは、スロットを識別するためにslotキーワードを使用します。スロットは関数なので、必要に応じて常に可視性(public、protected、private)を指定することができます。

このスロットの実装をMainWindow.cppファイルに追加します。

```C++
void MainWindow::addTask()
{
    qDebug() << "User clocked on the button";
}
```

QtはQDebugクラスを使ってデバッグ情報を効率的に表示する方法を提供しています。QDebugオブジェクトを取得する簡単な方法は、qDebug()関数を呼び出すことです。そして、ストリーム演算子を使ってデバッグ情報を送信します。

ファイルの先頭を以下のように更新します。

```C++
#include <QDebug>

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);
    connect(ui->addTaskButton, &QPushButton::clicked, this, &MainWindow::addTask);
}
```

スロットでqDebug()を使用するので\<QDebug\>をincludeする必要があります。更新されたconnectはアプリケーションを終了する代わりにカスタムスロットを呼び出すようになりました。

アプリケーションをビルドして実行します。ボタンをクリックするとQt Creatorのアプリケーション出力タブの中にデバッグメッセージが表示されます。

***
**[戻る](../index.html)**
