# QThreadを理解する

Qt は洗練されたスレッディングシステムを提供しています。ここでは、スレッド化の基本とそれに関連する問題（デッドロック、スレッド同期、リソース共有など）をすでに知っていることを前提に、Qt がどのように実装しているかに焦点を当てて説明します。

QThread は Qt スレッディングシステムの中心的なクラスです。QThread インスタンスは、プログラム内の 1 つの実行スレッドを管理します。

QThread をサブクラス化して run() 関数をオーバーライドすることで、QThread フレームワークで実行されるようになります。QThreadを作成して起動する方法をご紹介します。

```C++
QThread thread;
thread.start();
```

start()関数の呼び出しは、自動的にスレッドのrun()関数を呼び出し、start()シグナルを発します。この時点でのみ、新しい実行スレッドが作成されます。run() が完了すると、thread オブジェクトは finished() シグナルを発します。

QThread はシグナル/スロット機構とシームレスに動作するという QThread の基本的な側面を持っています。Qt はイベント駆動型のフレームワークで、メインイベントループ（または GUI ループ）がイベント（ユーザー入力、グラフィカルなど）を処理して UI をリフレッシュします。

各 QThread には、メインループの外でイベントを処理できる独自のイベントループが付属しています。オーバーライドされていない場合、run() は QThread::exec() 関数を呼び出し、 thread オブジェクトのイベントループを開始します。また、QThread をオーバーライドして自分で exec() を呼び出すこともできます。

```C++
class Thread : public QThread
{
    Q_OBJECT

protected:
    void run()
    {
        Object* myObject = new Object();
        connect(myObject, &Object::started,
        this, &Thread::doWork);
        exec();
    }

private slots:
    void doWork();
};
```

start()シグナルは、exec()呼び出し時にのみ Thread イベントループによって処理されます。QThread::exit() が呼び出されるまでブロックして待機します。

注意すべき重要な点は、スレッドイベントループは、そのスレッド内に存在するすべての QObject に対してイベントを配信するということです。これには、そのスレッドで作成された、あるいはそのスレッドに移動されたすべてのオブジェクトが含まれます。これをオブジェクトのスレッド親和性といいます。例を見てみましょう。

```C++
class Thread : public QThread
{
    Thread() :
    mObject(new QObject())
    {
    }

private :
    QObject* myObject;
};

// Somewhere in MainWindow
Thread thread;
thread.start();
```

このスニペットでは、ThreadクラスのコンストラクタでmyObjectが構築され、MainWindowで順番に作成されています。この時点では、thread は GUI スレッド内に存在しています。したがって、myObjectもGUIスレッドの中で生きています。

***

## Info

QCoreApplication オブジェクトの前に作成されたオブジェクトは、スレッドの親和性がありません。結果として、イベントはそれにディスパッチされません。

***

シグナルやスロットを独自の QThread で扱えるのは素晴らしいことですが、複数のスレッドにまたがってシグナルを制御するにはどうすればいいのでしょうか？典型的な例としては、別のスレッドで実行される長時間実行のプロセスがあり、そのプロセスは UI に通知して状態を更新する必要があります。

```C++
class Thread : public QThread
{
    Q_OBJECT

    void run() {
        // long running operation
        emit result("I <3 threads");
    }

signals:
    void result(QString data);
};

// Somewhere in MainWindow
Thread* thread = new Thread(this);
connect(thread, &Thread::result, this, &MainWindow::handleResult);
connect(thread, &Thread::finished, thread, &QObject::deleteLater);
thread->start();
```

直感的には、最初の connect は複数のスレッドに渡ってシグナルを送信し（結果を MainWindow::handleResult で利用できるようにするため）、2 番目の connect はスレッドのイベントループのみで動作すると想定しています。

幸いなことに、これは connect() 関数のシグネチャのデフォルト引数である接続タイプのためです。完全なシグネチャを見てみましょう。

```C++
QObject::connect(
    const QObject *sender, const char *signal,
    const QObject *receiver, const char *method,
    Qt::ConnectionType type = Qt::AutoConnection)
```

typeキーワードは、Qt::AutoConnectionをデフォルト値とします。Qt::ConectionType 列挙の可能な値を、Qt の公式ドキュメントに記載されている通りに見てみましょう。

* Qt::AutoConnection: 受信機がシグナルを発信するスレッド内に存在する場合は、Qt::DirectConnection が使用されます。そうでない場合は Qt::QueuedConnection が使用されます。接続タイプは、シグナルが発信されたときに決定されます。
* Qt::DirectConnection: このスロットは、シグナルが放出されるとすぐに呼び出されます。このスロットはシグナリングスレッドで実行されます。
* Qt::QueuedConnection: このスロットは、制御が受信機のスレッドのイベントループに戻ったときに呼び出される。このスロットは、レシーバのスレッドで実行されます。
* Qt::BlockingQueuedConnection: これは Qt::QueuedConnection と同じですが、スロットが戻るまでシグナリングスレッドがブロックされます。レシーバがシグナリングスレッド内に存在する場合は、この接続を使用してはいけません。
* Qt::UniqueConnection: これは、ビット単位の OR を使用して、前の接続タイプのいずれかと組み合わせることができるフラグです。Qt::UniqueConnection が設定されている場合、QObject::connect() は、接続が既に存在する場合に失敗します (つまり、同じオブジェクトのペアに対して同じシグナルが既に同じスロットに接続されている場合)。

Qt::AutoConnectionを使用する場合、最終的なConnectionTypeは、シグナルが効果的に放出された場合にのみ解決されます。例をもう一度見てみると、最初のconnect()のところで

```C++
connect(thread, &Thread::result,
    this, &MainWindow::handleResult);
```

result() が放出されると、Qt は handleResult() のスレッドの親和性を見ますが、これは result() シグナルのスレッドの親和性とは異なります。threadオブジェクトはMainWindowに住んでいますが（MainWindowで作成されていることを覚えておいてください）、result()シグナルはrun()関数の中で放出されており、実行中の別のスレッドで実行されています。その結果、Qt::QueuedConnection スロットが使用されることになります。

次に、2つ目のconnect()を見てみましょう。

```C++
connect(thread, &Thread::finished, thread, &QObject::deleteLater);
```

ここでは deleteLater() と finished() は同じスレッドに存在しています; そのため、Qt::DirectConnection スロットが使用されます。

Qtはオブジェクトのスレッドがどのようなものであるかは気にせず、"実行のコンテキスト "というシグナルだけを見ているということを理解しておくことが重要です。

この知識を積んだ上で、最初の QThread クラスの例を見て、このシステムを完全に理解することができます。

```C++
class Thread : public QThread
{
    Q_OBJECT

protected:
    void run()
    {
        Object* myObject = new Object();
        connect(myObject, &Object::started,
        this, &Thread::doWork);
        exec();
    }

private slots:
    void doWork();
};
```

Object::start()関数が出ると、Qt::QueuedConnectionスロットが使われます。ここで脳がフリーズします。Thread::doWork()関数は、Run()で生成されたObject::start()とは別のスレッドに住んでいます。もし Thread が UI スレッドでインスタンス化されていたら、ここが doWork() の所属する場所になるはずです。

このシステムは強力ですが、複雑です。物事をよりシンプルにするために、Qt はワーカーモデルを採用しています。これはスレッドの配管を実際の処理から分離します。以下に例を示します。

```C++
class Worker : public QObject
{
    Q_OBJECT

public slots:
    void doWork()
    {
    emit result("workers are the best");
    }

signals:
    void result(QString data);
};

// Somewhere in MainWindow
QThread* thread = new Thread(this);
Worker* worker = new Worker();
worker->moveToThread(thread);

connect(thread, &QThread::finished,
    worker, &QObject::deleteLater);
connect(this, &MainWindow::startWork,
    worker, &Worker::doWork);
connect(worker, &Worker::resultReady,
    this, handleResult);

thread->start();

// later on, to stop the thread
thread->quit();
thread->wait();
```

まず、Worker クラスを作成します。

* doWork() スロットには、以前の QThread::run() の内容が含まれています。
* 結果データを出力する result() シグナル．

次に MainWindow クラスでは、単純な thread オブジェクトと Worker のインスタンスを作成します。worker->moveToThread(thread)でマジックが起こります。これにより、worker オブジェクトの親和性が変更されます。これで、worker は thread オブジェクトの中に存在することになります。

現在のスレッドから別のスレッドにオブジェクトをプッシュすることしかできません。逆に、他のスレッドに住んでいるオブジェクトを引っ張ることはできません。オブジェクトが自分のスレッドに存在しない場合は、そのオブジェクトのスレッド親和性を変更することはできません。thread->start() が実行されると、この新しいスレッドからでない限り worker->moveToThread(this) をコールすることはできません。

その後、3つのconnect()を行います。

1. スレッドが終了したら刈り上げてワーカーのライフサイクルを処理します。このシグナルはQt::DirectConnectionを使用します。
2. UI イベントが発生する可能性があるときに Worker::doWork() を開始します。このシグナルは Qt::QueuedConnection を使用します。
3. 結果として得られたデータを handleResult() で UI スレッドで処理します。このシグナルは Qt::QueuedConnection を使用します。

要約すると、QThread はサブクラス化することもできますし、 worker クラスと組み合わせて使用することもできます。一般的にワーカーのアプローチが好まれるのは、スレッドの親和性を高めるための配管を、並列に実行したい実際の操作からよりきれいに分離できるからです。

***

**[戻る](../index.html)**
