# 独自の QTcpServer を構築する

すべてのソケットで読み書きができるようになりました。しかし、これらすべてのインスタンスをオーケストレーションするためのサーバがまだ必要です。そのためには、第9章「マルチスレッドで正気を保つ」で説明したMandelbrotCalculatorクラスの修正版を開発します。

アイデアは、マンデルブロ画像生成が異なるプロセス/マシン上でデポートされるという事実に気づかないようにマンデルブロWidgetを持つために、同じインターフェイスを尊重することです。

古いMandelbrotCalculatorと新しいMandelbrotCalculatorの主な違いは、QThreadPoolクラスをQTcpServerに置き換えたことです。MandelbrotCalculatorクラスは、WorkerにJobRequestsをディスパッチして結果を集計するだけの責任を持つようになりましたが、QThreadPoolクラスとのやりとりはなくなりました。

MandelbrotCalculator.cppという名前の新しいC++クラスを作成し、MandelbrotCalculator.hをこれに合わせて更新します。

```C++
#include <memory>
#include <vector>
#include <QTcpServer>
#include <QList>
#include <QThread>
#include <QMap>
#include <QElapsedTimer>

#include "WorkerClient.h"
#include "JobResult.h"
#include "JobRequest.h"

class MandelbrotCalculator : public QTcpServer
{
    Q_OBJECT

public:
    MandelbrotCalculator(QObject* parent = 0);
    ~MandelbrotCalculator();

signals:
    void pictureLinesGenerated(QList<JobResult> jobResults);
    void abortAllJobs();

public slots:
    void generatePicture(QSize areaSize, QPointF moveOffset,
    double scaleFactor, int iterationMax);

private slots:
    void process(WorkerClient* workerClient, JobResult jobResult);
    void removeWorkerClient(WorkerClient* workerClient);

protected:
    void incomingConnection(qintptr socketDescriptor) override;

private:
    std::unique_ptr<JobRequest> createJobRequest(
    int pixelPositionY);
    void sendJobRequests(WorkerClient& client,
    int jobRequestCount = 1);
    void clearJobs();

private:
    QPointF mMoveOffset;
    double mScaleFactor;
    QSize mAreaSize;
    int mIterationMax;
    int mReceivedJobResults;
    QList<JobResult> mJobResults;
    QMap<WorkerClient*, QThread*> mWorkerClients;
    std::vector<std::unique_ptr<JobRequest>> mJobRequests;
    QElapsedTimer mTimer;
};
```

変更された(または新しい)データがハイライトされています。まず、クラスがQObjectではなくQTcpServerを継承するようになったことに注意してください。MandelbrotCalculatorクラスはQTcpServerになり、接続を受け付けて管理できるようになりました。このトピックを掘り下げる前に、新しいメンバーのおさらいをしておきましょう。

* mWorkerClients: これは、WorkerClient と QThread のペアを格納する QMap です。WorkerClient が作成されるたびに、関連する QThread も生成され、その両方が mWorkerClients に格納されます。
* mJobRequests: これは、現在のピクチャのジョブリクエストのリストです。ピクチャ生成が要求されるたびに、ジョブリクエストの完全なリストが生成され、WorkerClients (ソケットの反対側のWorker) にディスパッチされる準備ができています。

そして機能は

* process(): この関数は、第9章「マルチスレッドで正気を保つ」で見たものを少し修正したものです。この関数は、pictureLinesGenerated()シグナルで送信する前にJobResultsを集約するだけでなく、渡されたWorkerClientにJobRequestをディスパッチしてビジー状態を保つようにしています。
* removeWorkerClient(): この関数は、指定された WorkerClient を mWorkerClients から削除します。
* incomingConnection(): この関数は、QTcpServerからのオーバーロード関数です。新しいクライアントがMandelbrotCalculatorに接続しようとするたびに呼び出されます。
* createJobRequest(): mJobRequestsに追加されるJobRequestを作成するヘルパー関数です。
* sendJobRequests(): この関数は、指定されたWorkerClientにJobRequestsのリストを送信することを担当します。

コンストラクタを使ってMandelbrotCalculator.cppを見てみましょう。

```C++
#include <QDebug>
#include <QThread>

using namespace std;

const int JOB_RESULT_THRESHOLD = 10;

MandelbrotCalculator::MandelbrotCalculator(QObject* parent) :
    QTcpServer(parent),
    mMoveOffset(),
    mScaleFactor(),
    mAreaSize(),
    mIterationMax(),
    mReceivedJobResults(0),
    mWorkerClients(),
    mJobRequests(),
    mTimer()
{
    listen(QHostAddress::Any, 5000);
}
```

これが本体のlisten()命令と共通の初期化リストです。QTcpServerをサブクラス化しているので、自分自身でlistenを呼び出すことができます。QHostAddress::AnyはIPv4とIPv6のどちらでも動作することに注意してください。

オーバーロードされた関数、incomingConnection()を見てみましょう。

```C++
void MandelbrotCalculator::incomingConnection(
                                        qintptr socketDescriptor)
{
    qDebug() << "Connected workerClient";
    QThread* thread = new QThread(this);
    WorkerClient* client = new WorkerClient(socketDescriptor);
    int workerClientsCount = mWorkerClients.keys().size();
    mWorkerClients.insert(client, thread);
    client->moveToThread(thread);

    connect(this, &MandelbrotCalculator::abortAllJobs,
            client, &WorkerClient::abortJob);

    connect(client, &WorkerClient::unregistered,
            this, &MandelbrotCalculator::removeWorkerClient);
    connect(client, &WorkerClient::jobCompleted,
            this, &MandelbrotCalculator::process);

    connect(thread, &QThread::started,
            client, &WorkerClient::start);
    thread->start();

    if(workerClientsCount == 0 &&
        mWorkerClients.size() == 1) {
        generatePicture(mAreaSize, mMoveOffset,
                        mScaleFactor, mIterationMax);
    }
}
```

listen() が呼ばれると、誰かが ip/port ペアに接続するたびに、パラメータとして socketDescriptor が渡され、 incomingConnection() がトリガされます。

***

## Tips

これは、単純な telnet 127.0.0.1 5000 コマンドを使ってマシン接続でテストできます。mandelbrot-appにConnected workerClientのログが表示されるはずです。

***

まず、QThreadクラスとWorkerClientを作成します。このペアはすぐに mWorkerClients マップに挿入され、clientスレッドの親和性はthreadに変更されます。

その後、クライアントを管理するための全てのコネクション(abortJob、unregister、jobCompleted)を行います。続けて、QThread::start()シグナルをWorkerClient::start()スロットに接続し、最後にthreadを起動します。

この関数の最後の部分は、最初のclient接続時に画像生成をトリガーするために使用されます。これをしなければ、パンやズームをするまで画面が黒いままでした。

ここまでWorkerClientの作成について説明してきましたが、 removeWorkerClient()でそのライフサイクルを破棄して終了させましょう。

```C++
void MandelbrotCalculator::removeWorkerClient(WorkerClient* workerClient)
{
    qDebug() << "Removing workerClient";
    QThread* thread = mWorkerClients.take(workerClient);
    thread->quit();
    thread->wait(1000);
    delete thread;
    delete workerClient;
}
```

workerClient/thread のペアが mWorkerClients から削除され、きれいに削除されます。この関数は、WorkerClient::unregisteredシグナルまたはMandelbrotCalculatorデストラクタから呼び出すことができることに注意してください。

```C++
MandelbrotCalculator::~MandelbrotCalculator()
{
    while (!mWorkerClients.empty()) {
        removeWorkerClient(mWorkerClients.firstKey());
    }
}
```

MandelbrotCalculatorが削除された場合、mWorkerClientsは適切に空にしなければなりません。イテレータスタイルのwhileループは、removeWorkerClient()を呼び出すのに良い仕事をしています。

この新しいバージョンのMandelbrotCalculatorでは、generatePicture()の動作は全く同じではありません。

```C++
void MandelbrotCalculator::generatePicture(
                                    QSize areaSize, QPointF moveOffset,
                                    double scaleFactor, int iterationMax)
{
    // sanity check & members initization
    ...

    for(int pixelPositionY = mAreaSize.height() - 1;
        pixelPositionY >= 0; pixelPositionY--) {
        mJobRequests.push_back(move(
            createJobRequest(pixelPositionY)));
    }

    for(WorkerClient* client : mWorkerClients.keys()) {
        sendJobRequests(*client, client->cpuCoreCount() * 2);
    }
}
```

始まりは同じです。しかし、終わり方は全く異なります。Jobsを作成してQThreadPoolに渡すのではなく、今はMandelbrotCalculatorがあります。

* 全体像を生成するためのJobRequestsを作成します。逆の順番で作成されることに注意してください。その理由はすぐにわかります。
* 所有している各WorkerClientに多数のJobRequestsをディスパッチします。

2 番目のポイントは、強く強調する必要があります。システムのスピードを最大化したいのであれば、複数のワーカーを使用し、それぞれが複数のコアを持ち、同時に複数のジョブを処理する必要があります。

Workerクラスが複数のジョブを同時に処理することができても、(WorkerClient::jobCompletedを介して)一つずつしかJobResultsを送ることができません。WorkerClientからJobResultオブジェクトを処理するたびに、1つのJobRequestをWorkerClientにディスパッチします。

Workerクラスが8つのコアを持っているとします。JobRequestsを1つずつ送信すると、Workerは常に7コアのアイドル状態になります。私たちはあなたのCPUを暖めるためにここに来たのであって、ビーチでモヒートを飲ませるために来たのではありません!

これを緩和するために、ワーカーに送信する最初のバッチの JobResults は、その coreCount() よりも高くなければなりません。そうすることで、全体像を生成するまでの間、常に処理すべき JobRequests のキューをワーカーに持たせておくことができます。これが、client->cpuCoreCount() * two 初期のJobRequestsを送信する理由です。この値を弄ってみるとわかると思います。

* jobCount < cpuCoreCount() の場合、Worker のいくつかのコアがアイドル状態になり、その CPU のパワーをフルに活用することができません。
* jobCount > cpuCoreCount() の値が大きすぎると、ワーカーのキューをオーバーロードする可能性があります。

複数の作業員がいても柔軟に対応できることを覚えておきましょう。RaspberryPIと16コアのx86を持っている場合、RaspberryPIはx86のCPUに遅れをとってしまいます。初期のJobRequestsを与えすぎることで、x86 CPUがすでにすべてのジョブを終了している間にRaspberryPIが全体の画像生成を阻害してしまいます。

マンデルブロートCalculatorの残りの関数を、createJobRequest()から説明していきましょう。

```C++
std::unique_ptr<JobRequest> MandelbrotCalculator::createJobRequest(int
pixelPositionY)
{
    auto jobRequest = make_unique<JobRequest>();
    jobRequest->pixelPositionY = pixelPositionY;
    jobRequest->moveOffset = mMoveOffset;
    jobRequest->scaleFactor = mScaleFactor;
    jobRequest->areaSize = mAreaSize;
    jobRequest->iterationMax = mIterationMax;
    return jobRequest;
}
```

これは、MandelbrotCalculatorのメンバフィールドを持つjobRequestを簡単に作成したものです。ここでも、jobRequestの所有権を明示的に示し、メモリリークを避けるためにunique_ptrを使用しています。

次に、sendJobRequests()で。

```C++
void MandelbrotCalculator::sendJobRequests(WorkerClient& client, int
    jobRequestCount)
{
    QList<JobRequest> listJobRequest;
    for (int i = 0; i < jobRequestCount; ++i) {
        if (mJobRequests.empty()) {
            break;
        }

        auto jobRequest = move(mJobRequests.back());
        mJobRequests.pop_back();
        listJobRequest.append(*jobRequest);
    }

    if (!listJobRequest.empty()) {
        emit client.sendJobRequests(listJobRequest);
    }
}
```

複数のJobRequestsを同時に送信することができるので、mJobRequestsの最後のJobRequestを取り出してlistJobRequestに追加することでJobRequestCountをループさせています。これがmJobRequestsを逆順に埋めていた理由です。

最後に、client.sendJobRequests()シグナルが発行され、WorkerClient::doSendJobRequests()スロットがトリガーされます。

これから、process()の修正版を見ていきます。

```C++
void MandelbrotCalculator::process(WorkerClient* workerClient,
                                   JobResult jobResult)
{
    // Sanity check and JobResult aggregation

    if (mReceivedJobResults < mAreaSize.height()) {
        sendJobRequests(*workerClient);
    } else {
        qDebug() << "Generated in" << mTimer.elapsed() << "ms";
    }
}
```

このバージョンでは、workerClientをパラメータとして渡しています。これは関数の最後に使用され、与えられたワーカークライアントに新しいJobRequestをディスパッチできるようにします。

最後に、abortAllJobs()のアップデート版です。

```C++
void MandelbrotCalculator::clearJobs()
{
    mReceivedJobResults = 0;
    mJobRequests.clear();
    emit abortAllJobs();
}
```

これは単に QThreadPool を空にする代わりに mJobRequests をクリアするだけです。

MandelbrotCalculatorクラスが完成しました。第9章「マルチスレッドで正気を保つ」からMandelBrotWidgetとMainWindow(.uiファイルを含む)をコピペします。誰が画像を生成しているのかわからないように、プラグアンドプレイできるように設計しました。QRunnable を持つローカルの QThreadPool や、IPC メカニズムを利用したミニオンなどです。

main.cppにはほんのわずかな違いしかありません。

```C++
#include <QApplication>
#include <QList>

#include "JobRequest.h"
#include "JobResult.h"
#include "WorkerClient.h"

int main(int argc, char *argv[])
{
    qRegisterMetaType<QList<JobRequest>>();
    qRegisterMetaType<QList<JobResult>>();
    qRegisterMetaType<WorkerClient*>();

    QApplication a(argc, argv);
    MainWindow w;
    w.show();

    return a.exec();
}
```

これで、mandelbrot-appを起動し、その後、アプリケーションに接続する1つまたは多くのmandelbrotworkerプログラムを起動することができます。自動的にピクチャ生成が起動するはずです。これでマンデルブロの画像生成は複数のプロセスにまたがって動作するようになりました!ソケットを使用することにしたので、異なる物理マシン上でアプリケーションとワーカーを起動することができます。

***

## Tips

IPv6を使用すると、非常に簡単に異なる場所でのアプリ/作業者の接続をテストすることができます。高速インターネット接続をしていない場合は、ネットワークがいかに画像生成を妨げているかがわかります。

***

複数のマシンにアプリケーションをデプロイして、このクラスタがどのように動作するかを確認するのに時間がかかるかもしれません。テストでは、非常に異質なマシン (PC、ラップトップ、Macbook など) を使用して、クラスタを 18 コアまでアップグレードしました。

***

まとめ

IPC は、コンピュータサイエンスの基本的なメカニズムです。この章では、IPC を行うために Qt が提供する様々なテクニックと、ソケットを使用して対話、コマンドの送受信を行うアプリケーションを作成する方法を学びました。マシンのクラスタ上で画像を生成できるようにすることで、オリジナルの mandelbrot-threadpool アプリケーションを次のレベルに引き上げました。

マルチスレッドアプリケーションの上にIPCを追加すると、いくつかの問題が発生します。ボトルネックになる可能性が高く、メモリリークの可能性があり、非効率的な計算をしなければなりません。Qtには、IPCを行うための複数のメカニズムが用意されています。Qt 5.7では、トランザクションを追加することで、シリアライズ/デシリアライズの部分が非常に簡単になりました。

次の章では、Qt マルチメディアフレームワークと、ファイルから C++ オブジェクトを保存してロードする方法について説明します。プロジェクトの例は、仮想ドラムマシンになります。トラックを保存してロードすることができるようになります。

***

**[戻る](../index.html)**
