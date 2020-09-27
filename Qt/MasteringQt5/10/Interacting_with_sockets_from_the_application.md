# アプリケーションからのソケットとのインタラクション

次のプロジェクトは mandelbrot-app です。これには、作業員と対話するQTcpServerとマンデルブロ集合の絵が含まれます。忘れないように、mandelbrot-appのアーキテクチャの図はここに示されています。

![image](img/9.png)

このアプリケーションを一から構築していきます。特定のWorkerとの接続を維持するためのクラスから始めましょう。WorkerClientです。このクラスは、特定の QThread 内に存在し、前節で説明した QTcpSocket/QDataStream メカニズムと同じ QTcpSocket/QDataStream メカニズムを使用して Worker クラスと対話します。

mandelbrot-appで、WorkerClientという名前の新しいC++クラスを作成し、WorkerClient.hを以下のように更新します。

```C++
#include <QTcpSocket>
#include <QList>
#include <QDataStream>

#include "JobRequest.h"
#include "JobResult.h"
#include "Message.h"

class WorkerClient : public QObject
{
    Q_OBJECT

public:
    WorkerClient(int socketDescriptor);

private:
    int mSocketDescriptor;
    int mCpuCoreCount;
    QTcpSocket mSocket;
    QDataStream mSocketReader;
};

Q_DECLARE_METATYPE(WorkerClient*)
```

Workerと非常に似ています。しかし、ライフサイクルの観点からは異なる挙動をするかもしれません。新しいWorkerがQTcpServerに接続するたびに、関連するQThreadと一緒に新しいWorkerClientが生成されます。WorkerClientオブジェクトは、mSocketを介してWorkerクラスと対話する責任を負います。

Workerが切断された場合、WorkerClientオブジェクトは削除され、QTcpServerクラスから削除されます。

メンバーから始めて、このヘッダーの内容を見直してみましょう。

* mSocketDescriptor: これは、システムがソケットと対話するために割り当てた一意の整数です。stdin, stdout, stderrもまた、アプリケーションの特定のストリームを指すディスクリプタです。与えられたソケットに対して、この値は QTcpServer で取得されます。これについては後述します。
* mCpuCoreCount: 接続されているワーカーのCPUコア数です。このフィールドは、WorkerがWORKER_REGISTERメッセージを送信したときに初期化されます。
* mSocket: これはWorkerクラスと対話するために使用されるQTcpSocketです。
* mSocketReader: これはWorkerでの役割と同じで、mSocketの内容を読み込みます。

これで、WorkerClient.hに関数を追加することができます。

```C++
class WorkerClient : public QObject
{
    Q_OBJECT

public:
    WorkerClient(int socketDescriptor);
    int cpuCoreCount() const;

signals:
    void unregistered(WorkerClient* workerClient);
    void jobCompleted(WorkerClient* workerClient,
                      JobResult jobResult);
    void sendJobRequests(QList<JobRequest> requests);

public slots:
    void start();
    void abortJob();

private slots:
    void readMessages();
    void doSendJobRequests(QList<JobRequest> requests);

private:
    void handleWorkerRegistered(Message& message);
    void handleWorkerUnregistered(Message& message);
    void handleJobResult(Message& message);
    ...
};
```

それぞれの関数が何をするのか見てみましょう。

* WorkerClient(): この関数は、パラメータとしてsocketDescriptorを期待しています。その結果、有効なソケットがないとWorkerClient関数を初期化することができません。
* cpuCoreCount(): この関数は、WorkerClientのオーナーにWorkerのコア数を知らせるシンプルなゲッターです。

このクラスには3つのシグナルがあります。

* unregister(): これは、WorkerClientがWORKER_UNREGISTERメッセージを受信したときに送られるシグナルです。
* jobCompleted(): これは、WorkerClientがJOB_RESULTメッセージを受信したときに送られるシグナルです。デシリアライズされたJobResultをコピーして渡します。
* sendJobRequests(): これは、キューに入れた接続のJobRequestsを適切なスロットに渡すためにWorkerClientのオーナーから発行されます: doSendJobRequests()。

スロットの詳細をご紹介します。

* start(): このスロットは、WorkerClientがプロセスを開始できるときに呼び出されます。通常、WorkerClientに関連付けられたQThreadのstartシグナルに接続されます。
* abortJob(): このスロットは、ALL_JOBS_ABORTメッセージのワーカーへの出荷をトリガします。
* readMessages(): このスロットは、ソケットに何かを読み込むたびに呼び出されます。
* doSendJobRequests(): このスロットは、WorkerへのJobRequestsの出荷をトリガーとする実際のスロットです。

そして最後に、プライベート機能の詳細です。

* handleWorkerRegistered(): WORKER_REGISTER メッセージを処理し、mCpuCoreCount を初期化します。
* handleWorkerUnregistered(): この関数は、WORKER_UNREGISTER メッセージを処理して unregistered() シグナルを発行します。
* handleJobResult(): この関数は、JOB_RESULT メッセージを処理し、jobCompleted() シグナルを通して内容をディスパッチします。

WorkerClient.cppでの実装はよく知っているはずです。ここにコンストラクタがあります。

```C++
#include "MessageUtils.h"

WorkerClient::WorkerClient(int socketDescriptor) :
    QObject(),
    mSocketDescriptor(socketDescriptor),
    mSocket(this),
    mSocketReader(&mSocket)
{
    connect(this, &WorkerClient::sendJobRequests,
            this, &WorkerClient::doSendJobRequests);
}
```

フィールドは初期化リストで初期化され、sendJobRequestsシグナルはプライベートスロットであるdoSendJobRequestsに接続されています。このトリックは、複数の関数の宣言を避けながらも、スレッド間でキュー接続を保持するために使用されます。

start()関数を進めていきます。

```C++
void WorkerClient::start()
{
    connect(&mSocket, &QTcpSocket::readyRead,
            this, &WorkerClient::readMessages);
            mSocket.setSocketDescriptor(mSocketDescriptor);
}
```

これは非常に短いです。最初にソケットからのREADYRead()シグナルをreadMessages()スロットに接続します。その後、mSocket は mSocketDescriptor で適切に設定されます。

接続は、WorkerClientに関連付けられたQThreadクラスで実行される必要があるため、start()で実行する必要があります。これにより、接続が直接接続され、mSocketがWorkerClientと対話するためにシグナルをキューイングする必要がないことがわかります。

関数の終了時には、関連する QThread は終了していないことに注意してください。それどころか、QThread::exec() でイベントループを実行しています。QThread クラスは、誰かが QThread::exit() を呼び出すまでイベントループを実行し続けます。

start()関数の唯一の目的は、適切なスレッドの親和性でmSocket接続の作業を行うことです。その後は、データを処理するためにQtのシグナル/スロット機構だけに頼っています。忙しいwhileループは必要ありません。

readMessages() クラスが待っています。

```C++
void WorkerClient::readMessages()
{
    auto messages = MessageUtils::readMessages(mSocketReader);
    for(auto& message : *messages) {
        switch (message->type) {
        case Message::Type::WORKER_REGISTER:
            handleWorkerRegistered(*message);
            break;

        case Message::Type::WORKER_UNREGISTER:
            handleWorkerUnregistered(*message);
            break;

        case Message::Type::JOB_RESULT:
            handleJobResult(*message);
            break;

        default:
            break;
        }
    }
}
```

ここでは何も驚くことはありません。Worker の場合と全く同じです。MessageUtils::readMessages() を使用して Messages をデシリアライズし、メッセージタイプごとに適切な関数を呼び出します。

以下、handleWorkerRegistered()から始まる各関数の内容です。

```C++
void WorkerClient::handleWorkerRegistered(Message& message)
{
    QDataStream in(&message.data, QIODevice::ReadOnly);
    in >> mCpuCoreCount;
}
```

WORKER_REGISTERメッセージの場合、Workerはmessage.dataでintをシリアライズしているだけなので、in >> mCpuCoreCountでその場でmCpuCoreCountを初期化することができます。

これで、handleWorkerUnregistered()の本体。

```C++
void WorkerClient::handleWorkerUnregistered(Message& /*message*/)
{
    emit unregistered(this);
}
```

これは、WorkerClientのオーナーに拾われるunregistered()シグナルを送信するための中継です。

最後の「読み込み」関数はhandleJobResult()です。

```C++
void WorkerClient::handleJobResult(Message& message)
{
    QDataStream in(&message.data, QIODevice::ReadOnly);
    JobResult jobResult;
    in >> jobResult;
    emit jobCompleted(this, jobResult);
}
```

これは見かけによらず短いです。これは message.data から jobResult コンポーネントをデシリアライズして jobCompleted() シグナルを出すだけです。

ソケットへの書き込み」関数は abortJob() と doSendJobRequest() です。

```C++
void WorkerClient::abortJob()
{
    MessageUtils::sendMessage(mSocket,
                              Message::Type::ALL_JOBS_ABORT,
                              true);
}

void WorkerClient::doSendJobRequests(QList<JobRequest> requests)
{
    QByteArray data;
    QDataStream stream(&data, QIODevice::WriteOnly);
    stream << requests;

    MessageUtils::sendMessage(mSocket,
                              Message::Type::JOB_REQUEST,
                              data);
}
```

abortJob() 関数は forceFlush フラグを true に設定して ALL_JOBS_ABORT メッセージを送信し、doSendJobRequests() はリクエストをストリームにシリアライズしてから MessageUtils::sendMessage() を使用して送信します。

***

**[戻る](../index.html)**
