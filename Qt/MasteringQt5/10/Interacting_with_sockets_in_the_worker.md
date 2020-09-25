# ワーカーのソケットとの相互作用

これでSDKが完成したので、ワーカーに目を向けることができます。プロジェクトは十分に複雑なので、Mandelbrot-worker アーキテクチャを使って記憶をリフレッシュしましょう。

![image](img/8.png)

まず、Jobクラスを実装することから始めます。mandelbrot-workerプロジェクトの中に、Jobという名前の新しいC++クラスを作成します。ここにJob.hの内容があります。

```C++
#include <QObject>
#include <QRunnable>
#include <QAtomicInteger>

#include "JobRequest.h"
#include "JobResult.h"

class Job : public QObject, public QRunnable
{
    Q_OBJECT

public:
    explicit Job(const JobRequest& jobRequest,
    QObject *parent = 0);
    void run() override;

signals:
    void jobCompleted(JobResult jobResult);

public slots:
    void abort();

private:
    QAtomicInteger<bool> mAbort;
    JobRequest mJobRequest;
};
```

もし、第9章「マルチスレッドで正気を保つ」の Job クラスを覚えているなら、このヘッダに心当たりがあるはずです。唯一の違いは、ジョブのパラメータ(エリアサイズ、スケールファクタなど)が直接メンバ変数として保存されるのではなく、JobRequestオブジェクトから抽出されることです。

ご覧のように、JobRequest オブジェクトは Job のコンストラクタで提供されています。Job.cppは第9章「マルチスレッドで正気を保つ」のバージョンと非常によく似ているので、ここでは取り上げません。

次に、Worker クラスに進みます。このクラスには以下のような役割があります。

* QTcpSocketクラスを使用してマンデルブロットアプリと対話します。
* QThreadPoolクラスにJobRequestsをディスパッチし、結果を集約して、QTcpSocketクラスを介してMandelbrot-appアプリケーションに送り返します。

まずはQTcpSocketクラスとのやりとりを勉強していきます。Workerという名前の新しいクラスを作成し、以下のようなヘッダーを付けます。

```C++
#include <QObject>
#include <QTcpSocket>
#include <QDataStream>

#include "Message.h"
#include "JobResult.h"

class Worker : public QObject
{
    Q_OBJECT

public:
    Worker(QObject* parent = 0);
    ~Worker();

private:
    void sendRegister();

private:
    QTcpSocket mSocket;
};
```

WorkerクラスはmSocketのオーナーです。最初に実装するのは、mandelbrot-appとの接続です。Worker.cppのWorkerのコンストラクタです。

```C++
#include "Worker.h"

#include <QThread>
#include <QDebug>
#include <QHostAddress>

#include "JobRequest.h"
#include "MessageUtils.h"

Worker::Worker(QObject* parent) :
    QObject(parent),
    mSocket(this)
{
    connect(&mSocket, &QTcpSocket::connected, [this] {
        qDebug() << "Connected";
        sendRegister();
    });
    connect(&mSocket, &QTcpSocket::disconnected, [] {
        qDebug() << "Disconnected";
    });

    mSocket.connectToHost(QHostAddress::LocalHost, 5000);
}
```

コンストラクタは、this を親として mSocket を初期化し、関連する mSocket シグナルをラムダに接続します。

* QTcpSocket::connected: ソケットが接続されると、そのレジスタメッセージが送信されます。この機能については後ほど説明します。
* QTcpSocket::disconnected: ソケットが切断されると、コンソールにメッセージが表示されます。

最後に、mSocket はローカルホスト上のポート 5000 で接続を試みます。このコード例では、ワーカーとアプリケーションを同じマシン上で実行していると仮定しています。ワーカーとアプリケーションを別のマシンで実行する場合は、この値を自由に変更してください。

sendRegister()関数の本文は以下のようになります。

```C++
void Worker::sendRegister()
{
    QByteArray data;
    QDataStream out(&data, QIODevice::WriteOnly);
    out << QThread::idealThreadCount();
    MessageUtils::sendMessage(mSocket,
                              Message::Type::WORKER_REGISTER,
                              data);
}
```

QByteArray クラスにワーカーのマシンの idealThreadCount 関数を埋めます。その後、MessageUtils::sendMessage を呼び出してメッセージをシリアライズし、mSocket を介して送信します。

ワーカーが登録されると、仕事の依頼を受けて処理したり、仕事の結果を送り返したりするようになります。更新されたWorker.hは以下の通りです。

```C++
class Worker : public QObject
{
    ...

signals:
    void abortAllJobs();

private slots:
    void readMessages();

private:
    void handleJobRequest(Message& message);
    void handleAllJobsAbort(Message& message);
    void sendRegister();
    void sendJobResult(JobResult jobResult);
    void sendUnregister();
    Job* createJob(const JobRequest& jobRequest);

private:
    QTcpSocket mSocket;
    QDataStream mSocketReader;
    int mReceivedJobsCounter;
    int mSentJobsCounter;
};
```

新メンバーの一人一人の役割を見直してみましょう。

* mSocketReader: これは、mSocket コンテンツを読み込むための QDataStream クラスです。MessageUtils::readMessages() 関数のパラメータとして渡されます。
* mReceivedJobsCounter: これは新しいJobRequestをmandelbrot-appから受信するたびにインクリメントされます。
* mSentJobsCounter: これは、新しい JobResult が mandelbrot-app に送信されるたびにインクリメントされます。

さて、新機能です。

* abortAllJobs(): これは、Worker クラスが適切なメッセージを受信したときに発せられるシグナルです。
* readMessages(): これはmTcpSocketに何かが読み込まれるたびに呼び出されるスロットです。メッセージを解析し、メッセージタイプごとに対応する関数を呼び出します。
* handleJobRequest(): この関数は、メッセージパラメータに含まれるJobRequestオブジェクトに従って、ジョブクラスを作成し、ディスパッチします。
* handleAllJobsAbort(): この関数は、現在のすべてのジョブをキャンセルし、スレッドキューをクリアします。
* sendJobResult(): この関数は、JobResultオブジェクトをmandelbrot-appに送信します。
* この関数は、JobResultオブジェクトをmandelbrotappに送信します。
* sendUnregister(): この関数は、登録解除メッセージをmandelbrot-appに送信します。
* createJob(): 新しいジョブのシグナルを作成し、適切につなげるヘルパー機能です。

これでヘッダーが完成しました。Worker.cppの更新されたコンストラクタに進むことができます。

```C++
Worker::Worker(QObject* parent) :
    QObject(parent),
    mSocket(this),
    mSocketReader(&mSocket),
    mReceivedJobsCounter(0),
    mSentJobsCounter(0)
{
    ...
    connect(&mSocket, &QTcpSocket::readyRead,
            this, &Worker::readMessages);

    mSocket.connectToHost(QHostAddress::LocalHost, 5000);
}
```

QDataStream mSocketReader変数は、mSocketのアドレスで初期化されます。これは、QIODeviceクラスからその内容を読み取ることを意味します。その後、QTcpSocketシグナルのREADYRead()に新しい接続を追加します。そのデータがソケット上で読み込めるようになるたびに、私たちのスロットであるreadMessages()が呼び出されます。

ここでは、readMessages()の実装を紹介します。

```C++
void Worker::readMessages()
{
    auto messages = MessageUtils::readMessages(mSocketReader);
    for(auto& message : *messages) {
        switch (message->type) {
        case Message::Type::JOB_REQUEST:
            handleJobRequest(*message);
            break;
        case Message::Type::ALL_JOBS_ABORT:
            handleAllJobsAbort(*message);
            break;
        default:
            break;
        }
    }
}
```

メッセージは MessageUtils::readMessages() 関数で解析されます。C++11 のセマンティクスである auto を使用していることに注意してください。これは、スマート ポインター構文をエレガントに隠しながらもメモリを処理してくれます。

解析されたmessageごとに、switchの場合に処理されます。handleJobRequest()を復習してみましょう。

```C++
void Worker::handleJobRequest(Message& message)
{
    QDataStream in(&message.data, QIODevice::ReadOnly);
    QList<JobRequest> requests;
    in >> requests;

    mReceivedJobsCounter += requests.size();
    for(const JobRequest& jobRequest : requests) {
        QThreadPool::globalInstance()
            ->start(createJob(jobRequest));
    }
}
```

この関数では、messageオブジェクトはすでにデシリアライズされています。しかし、message.dataはまだデシリアライズする必要があります。これを実現するために、message.dataから読み込む変数にQDataStreamを作成します。

ここから、リクエストを QList で解析します。QList は既に >> 演算子をオーバーライドしているので、カスケードで動作し、JobRequest >> 演算子のオーバーロードを呼び出します。データのデシリアライズがこれまでになく簡単になりました。

その後、mReceivedJobsCounterをインクリメントし、これらのJobRequestsの処理を開始します。それぞれについて、Job クラスを作成し、グローバル QThreadPool クラスにディスパッチします。QThreadPool について疑問がある場合は、第 9 章「マルチスレッドで正気を保つ」に戻ってください。

createJob() 関数の実装は簡単です。

```C++
Job* Worker::createJob(const JobRequest& jobRequest)
{
    Job* job = new Job(jobRequest);
    connect(this, &Worker::abortAllJobs,
            job, &Job::abort);
    connect(job, &Job::jobCompleted,
            this, &Worker::sendJobResult);
    return job;
}
```

新しい Job クラスが作成され、そのシグナルが適切に接続されます。Worker::abortAllJobsが放出された場合、実行中の全てのJobはJob::abortスロットでキャンセルされるべきです。

2つ目のシグナルである Job::jobCompleted は、ジョブクラスが値の計算を終えたときに発せられます。接続されたスロット、sendJobResult()を見てみましょう。

```C++
void Worker::sendJobResult(JobResult jobResult)
{
    mSentJobsCounter++;
    QByteArray data;
    QDataStream out(&data, QIODevice::WriteOnly);
    out << jobResult;
    MessageUtils::sendMessage(mSocket,
                              Message::Type::JOB_RESULT,
                              data);
}
```

最初に mSentJobsCounter をインクリメントし、次に JobResult を QByteArray データにシリアライズして MessageUtils::sendMessage() に渡します。

これでJobRequestのハンドリングと、次のJobResultの出荷のツアーは終了しました。readMessages()から呼び出されるhandleAllJobsAbort()をまだカバーしなければなりません。

```C++
void Worker::handleAllJobsAbort(Message& /*message*/)
{
    emit abortAllJobs();
    QThreadPool::globalInstance()->clear();
    mReceivedJobsCounter = 0;
    mSentJobsCounter = 0;
}
```

最初に abortAllJobs() シグナルが発せられ、実行中のすべてのジョブにプロセスをキャンセルするように指示します。その後、QThreadPool クラスはクリアされ、カウンタはリセットされます。

Workerの最後のピースはsendUnregister()で、これはWorkerのデストラクタで呼び出されます。

```C++
Worker::~Worker()
{
    sendUnregister();
}

void Worker::sendUnregister()
{
    MessageUtils::sendMessage(mSocket,
                              Message::Type::WORKER_UNREGISTER,
                              true);
}
```

sendUnregister() 関数は、シリアライズするデータを一切持たずに sendMessage を呼び出すだけです。ソケットがフラッシュされ、mandelbrot-appアプリケーションが可能な限り早くメッセージを受信できるようにするために、 forceFlushフラグをtrueに渡していることに注意してください。

Workerインスタンスは、現在の計算の進捗状況を表示するウィジェットで管理されます。WorkerWidgetという名前の新しいクラスを作成し、WorkerWidget.hを以下のように更新します。

```C++
#include <QWidget>
#include <QThread>
#include <QProgressBar>
#include <QTimer>

#include "Worker.h"

class WorkerWidget : public QWidget
{
    Q_OBJECT

public:
    explicit WorkerWidget(QWidget *parent = 0);
    ~WorkerWidget();

private:
    QProgressBar mStatus;
    Worker mWorker;
    QThread mWorkerThread;
    QTimer mRefreshTimer;
};
```

WorkerWidgetのメンバーは

* mStatus: 処理されたJobRequestsの割合を表示するQProgressBar
* mWorker: WorkerWidget が所有し、起動した Worker インスタンス
* mWorkerThread: mWorker が実行される QThread クラス。
* mRefreshTimer: プロセスの進行状況を知るために定期的に mWorker をポーリングする QTimer クラス

WorkerWidget.cppに進みます。

```C++
#include "WorkerWidget.h"

#include <QVBoxLayout>

WorkerWidget::WorkerWidget(QWidget *parent) :
    QWidget(parent),
    mStatus(this),
    mWorker(),
    mWorkerThread(this),
    mRefreshTimer()
{
    QVBoxLayout* layout = new QVBoxLayout(this);
    layout->addWidget(&mStatus);

    mWorker.moveToThread(&mWorkerThread);

    connect(&mRefreshTimer, &QTimer::timeout, [this] {
        mStatus.setMaximum(mWorker.receivedJobsCounter());
        mStatus.setValue(mWorker.sentJobCounter());
    });

    mWorkerThread.start();
    mRefreshTimer.start(100);
}

WorkerWidget::~WorkerWidget()
{
    mWorkerThread.quit();
    mWorkerThread.wait(1000);
}
```

まず、WorkerWidgetのレイアウトにmStatus変数を追加します。その後、mWorkerスレッドのアフィニティをmWorkerThreadに移動し、mWorkerをポーリングしてmStatusデータを更新するようにmRefreshTimerを設定します。

最後に、mWorkerThreadが起動され、mWorkerプロセスがトリガーされます。mRefreshTimer オブジェクトもまた、各タイムアウトの間に 100 ミリ秒の間隔を置いて起動されます。

mandelbrot-workerで最後にカバーするのはmain.cppです。

```C++
#include <QApplication>

#include "JobResult.h"
#include "WorkerWidget.h"

int main(int argc, char *argv[])
{
    qRegisterMetaType<JobResult>();

    QApplication a(argc, argv);
    WorkerWidget workerWidget;

    workerWidget.show();
    return a.exec();
}
```

シグナル/スロットの仕組みで使われているので、まずはJobResultをqRegisterMetaTypeに登録します。その後、WorkerWidgetのレイアウトをインスタンス化して表示します。

***

**[戻る](../index.html)**
