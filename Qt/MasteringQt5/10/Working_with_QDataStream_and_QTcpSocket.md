# QDataStreamとQTcpSocketを使った作業

SDK の欠けている部分は MesssageUtils です。シリアライゼーションと QDataStream トランザクションという 2 つの主要なトピックをカバーしているため、専用のセクションが必要です。

まずはシリアライズから始めましょう。既に、Messageは不透明なQByteArrayデータメンバーのみを格納していることを知っています。結果として、必要なデータは、Messageに渡される前にQByteArrayとしてシリアライズされなければなりません。

JobRequestオブジェクトを例にとると、直接送信されるわけではありません。まず、適切なMessage型を持つ一般的なMessageオブジェクトを入れます。次の図は、行われるべき一連の動作をまとめたものです。

![image](img/7.png)

JobRequestオブジェクトは最初にQByteArrayクラスにシリアライズされます。その後、Messageインスタンスに渡され、そのインスタンスは最終的なQByteArrayにシリアライズされます。デシリアライズ処理は、このシーケンスの完全なミラーです（右から左へ）。

データをシリアライズすると、多くの疑問が出てきます。どのようにして一般的な方法でそれを行うことができるのでしょうか？CPUアーキテクチャの可能性のあるエンディアン性をどのように扱うか？どのようにしてデータの長さを指定して適切にデシリアライズするのか？

繰り返しになりますが、Qtの皆さんは素晴らしい仕事をしてくれて、これらの問題に対処するための素晴らしいツールを提供してくれました。QDataStreamです。

QDataStreamクラスは、任意のQIODevice（QAbstractSocket、QProcess、QFileDevice、QSerialPortなど）にバイナリデータをシリアライズすることを可能にします。QDataStreamの大きな利点は、プラットフォームに依存しないフォーマットで情報をエンコードすることです。バイトオーダー、オペレーティングシステム、CPUを気にする必要はありません。

QDataStreamクラスは、C++のプリミティブ型といくつかのQt型(QBrush, QColor, QStringなど)のシリアライズを実装しています。以下に基本的な書き方の例を示します。

```C++
QFile file("myfile");
file.open(QIODevice::WriteOnly);
QDataStream out(&file);
out << QString("QDataStream saved my day");
out << (qint32)42;
```

ご覧のように、QDataStreamはデータを書き込むために<演算子のオーバーロードに依存しています。情報を読み込むには、正しいモードでファイルを開き、 >> 演算子で読み込みます。

話を戻して、JobRequestのようなカスタムクラスをシリアライズしたいと思います。そのためには、JobRequestの<<演算子をオーバーロードしなければなりません。関数のシグネチャは以下のようになります。

```C++
QDataStream& operator<<(QDataStream& out,
    const JobRequest& jobRequest)
```

ここで書いていることは、out << jobRequestオペレータコールをカスタムバージョンでオーバーロードしたいということです。そうすることで、outオブジェクトをjobRequestの内容で埋めようとしています。QDataStreamはすでにプリミティブ型のシリアライズをサポートしているので、あとはシリアライズするだけです。

JobRequest.hのアップデート版です。

```C++
#include <QSize>
#include <QPointF>
#include <QDataStream>

struct JobRequest
{
    ...
};

inline QDataStream& operator<<(QDataStream& out,
    const JobRequest& jobRequest)
{
    out << jobRequest.pixelPositionY
        << jobRequest.moveOffset
        << jobRequest.scaleFactor
        << jobRequest.areaSize
        << jobRequest.iterationMax;
    return out;
}

inline QDataStream& operator>>(QDataStream& in,
    JobRequest& jobRequest)
{
    in >> jobRequest.pixelPositionY;
    in >> jobRequest.moveOffset;
    in >> jobRequest.scaleFactor;
    in >> jobRequest.areaSize;
    in >> jobRequest.iterationMax;
    return in;
}
```

QDataStreamをインクルードし、<<を非常に簡単にオーバーロードしています。返されたoutは、渡されたjobRequestのプラットフォームに依存しない内容で更新されます。演算子のオーバーロードは同じパターンに従います。jobRequestパラメータを変数inの内容で埋めます。裏では、QDataStreamはシリアライズされたデータの中に変数のサイズを格納し、後から読み込めるようにしています。

メンバーのシリアライズとデシリアライズを同じ順番で行うように注意してください。この点に注意を払わないと、JobRequestの中で非常に奇妙な値に遭遇するかもしれません。

JobResult演算子のオーバーロードは同じパターンに従っており、この章に含めるに値しません。実装に疑問がある場合は、プロジェクトのソースコードを見てください。

一方、Message演算子のオーバーロードはカバーする必要があります。

```C++
#include <QByteArray>
#include <QDataStream>
#include <QByteArray>
#include <QDataStream>

struct Message {
    ...
};

inline QDataStream &operator<<(QDataStream &out, const Message &message)
{
    out << static_cast<qint8>(message.type)
        << message.data;
    return out;
}

inline QDataStream &operator>>(QDataStream &in, Message &message)
{
    qint8 type;
    in >> type;
    in >> message.data;

    message.type = static_cast<Message::Type>(type);
    return in;
}
```

Message::Type enum class シグナルには int への暗黙の変換がないため、シリアライズするには明示的に変換する必要があります。メッセージの型は255を超えないことがわかっているので、安全にqint8型にキャストすることができます。

読み込みの部分も同じです。まず、in >> typeで埋められるqint8 type変数を宣言し、type変数をmessageのMessage::Typeにキャストします。

私たちのSDKクラスは、シリアライズとデシリアライズの準備ができています。MessageUtils で、メッセージのシリアライズと QTcpSocket クラスへの書き込みを実際に見てみましょう。

常に sdk ディレクトリに、以下の内容の MessageUtils.h ヘッダを作成します。

```C++
#include <QByteArray>
#include <QTcpSocket>
#include <QDataStream>

#include "Message.h"

namespace MessageUtils {

inline void sendMessage(QTcpSocket& socket,
    Message::Type messageType,
    QByteArray& data,
    bool forceFlush = false)
{
    Message message(messageType, data);

    QByteArray byteArray;
    QDataStream stream(&byteArray, QIODevice::WriteOnly);
    stream << message;
    socket.write(byteArray);
    if (forceFlush) {
        socket.flush();
    }
}
```

MessageUtils クラスは状態を保持しないので、インスタンスを作成する必要はありません。ここでは、名前の衝突から関数を保護するために MessageUtils 名前空間を使用しています。

スニペットの本質はsendMessage()にあります。パラメータを見てみましょう。

* socket: これはメッセージが送信されるQTcpSocketクラスです。メッセージが適切に開かれるようにするのは呼び出し側の責任です。
* messageType: これは、送信するメッセージのタイプです。
* data: これは、メッセージに含まれるシリアル化されたデータです。これはQByteArrayクラスであり、呼び出し元がそのカスタムクラスやデータをすでにシリアライズしていることを意味します。
* forceFlush: これは、メッセージの送信時にソケットを強制的にフラッシュするためのフラグです。OS はソケットバッファを保持しており、そのバッファが満たされるのを待ってから送信します。メッセージの中には、すべてのジョブを中止するメッセージのように、すぐに配信しなければならないものもあります。

関数自体では、まず渡されたパラメータでメッセージを作成します。そして、QByteArrayクラスを作成します。このbyteArrayがシリアル化されたデータの受け皿となります。

実は、QIODevice::WriteOnlyモードで開いているQDataStreamストリームのコンストラクタにはbyteArrayが渡されています。つまり、ストリームがそのデータをbyteArrayに出力するということです。

その後、メッセージは stream << message で優雅にストリームにシリアライズされ、修正されたbyteArrayはsocket.write(byteArray)でソケットに書き込まれます。

最後に、forceFlushフラグがtrueに設定されている場合、 socket.flush()でソケットをフラッシュします。

メッセージの中には、関連するペイロードを持たないものもあります。このため、この目的のために小さなヘルパー関数を追加します。

```C++
inline void sendMessage(QTcpSocket& socket,
                        Message::Type messageType,
                        bool forceFlush = false) {
    QByteArray data;
    sendMessage(socket, messageType, data, forceFlush);
}
```

sendMessage() が完了したので、次は readMessages() に移りましょう。IPC、特にソケットを使って作業をしているので、メッセージを読み込んだり解析したりするときに興味深い問題が発生します。

ソケットに何かが読み込まれる準備ができたとき、信号が通知してくれます。しかし、どのくらいの量を読み込めばいいのか、どうやって知ることができるのでしょうか？WORKER_DISCONNECTメッセージの場合、ペイロードはありません。一方、JOB_RESULTメッセージの場合、非常に重くなることがあります。さらに悪いことに、いくつかのJOB_RESULTメッセージがソケットに並んで、読まれるのを待っていることがあります。

物事をより困難にするためには、私たちがネットワークを使って作業しているという事実を認めなければなりません。パケットは紛失したり、再送されたり、不完全だったり、何でもあります。確かに、TCPは最終的にすべての情報を確実に得ることができますが、それが遅れることもあります。

もし自分たちでそれをしなければならないとしたら、各メッセージのペイロードサイズとフッターを持つカスタムのメッセージヘッダーが必要になるでしょう。

Qt 5.7で導入された機能が救いです。QDataStream トランザクションです。QIODeviceクラスの読み込みを開始するときに、どのくらいの量のデータを読み込まなければならないか（埋めたいオブジェクトのサイズに基づいて）はすでにわかっています。しかし、一度の読み込みではすべてのデータを取得できないかもしれません。

読み取りが完了していない場合、QDataStreamは既に読み込まれた内容を一時的なバッファに保存し、次の読み取り時に復元します。次の読み取りでは、すでに読み込まれた内容と新しい読み取りの内容が含まれます。これは、後で読み込んでもよいリードストリームのチェックポイントとして見ることができます。

この処理は、データが読み込まれるまで繰り返すことができます。公式ドキュメントには、十分にシンプルな例が記載されています。

```C++
in.startTransaction();
qint8 messageType;
QByteArray messageData;
in >> messageType >> messageData;

if (!in.commitTransaction())
    return;
```

読み込みたいQDataStreamクラスでは、in.startTransaction()がストリームのチェックポイントをマークします。そして、messageTypeとmessageDataをアトミックに読み込もうとします。それができない場合、in.commitTransaction() は false を返し、読み込んだデータは内部バッファにコピーされます。

このコードの次の呼び出し(読み込むデータが増える)では、in.startTransaction()が前のバッファを復元し、アトミックリードを終了させようとします。

readMessages() の状況では、複数のメッセージを一度に受け取ることができます。そのため、コードが少し複雑になっています。ここに MessageUtils の更新版があります。

```C++
#include <memory>
#include <vector>
#include <QByteArray>
#include <QTcpSocket>
#include <QDataStream>

#include "Message.h"

...

inline std::unique_ptr<std::vector<std::unique_ptr<Message>>>
readMessages(QDataStream& stream)
{
    auto messages =
        std::make_unique<std::vector<std::unique_ptr<Message>>>();
    bool commitTransaction = true;
    while (commitTransaction
            && stream.device()->bytesAvailable() > 0) {
        stream.startTransaction();
        auto message = std::make_unique<Message>();
        stream >> *message;
        commitTransaction = stream.commitTransaction();
        if (commitTransaction) {
            messages->push_back(std::move(message));
        }
    }
    return messages;
}

}
```

この関数では、パラメータはQDataStreamのみです。呼び出し元が stream.setDevice(socket) でソケットとストリームをリンクしたと仮定しています。

読み込まれる内容の長さがわからないので、いくつかのメッセージを読む準備をします。所有権を明示的に示し、メモリリークを避けるために、 vector\<unique_ptr\<Message\>\>を返します。このvectorは、ヒープ上に割り当てることができるように、また、関数のリターン中にコピーが発生しないようにするために、unique_ptrポインタでなければなりません。

関数自体では、まずvectorを宣言することから始めます。その後、whileループを実行します。ループ内に留まるための条件は2つです。

* commitTransaction == true: これは、実行されたストリーム内のアトミックリードです。完全なmessage が読み込まれました。
* stream.device().bytesAvailable() > 0: これは、ストリームにまだ読み込むデータがあることを示しています。

while ループでは、まずストリームを stream.startTransaction() でマークします。その後、*messageシグナルのアトミックリードを実行し、stream.commitTransaction()で結果を確認します。成功した場合は、新しいmessageがmessagesベクトルに追加されます。これは、 bytesAvailable() > 0 テストでストリームのすべての内容を読み込むまで繰り返されます。

何が起こるかを理解するために、ユースケースを勉強してみましょう。readMessages()で複数のメッセージを受信したとします。

* streamオブジェクトは、それをmessageに読み込もうとします。
* commitTransaction変数がtrueに設定され、最初のメッセージがmessagesに追加されます。
* もし、streamにまだ読み込むバイトがあれば、ステップ1から繰り返します。そうでなければ、ループを終了します。

まとめると、ソケットを扱うことは、それ自体に疑問を投げかけることになります。一方で、ソケットは非常に強力な IPC メカニズムであり、多くの柔軟性を持っています。一方で、ネットワーク自体の性質上、ソケットは多くの複雑さをもたらします。幸いなことに、Qt (そしてさらに Qt 5.7) は私たちを助けてくれる素晴らしいクラスを提供してくれています。

QDataStreamのシリアライズとトランザクションのオーバーヘッドは、我々の必要性に合っているため、大目に見ていることを覚えておいてください。制約のある組み込みプラットフォームで作業している場合、シリアル化のオーバーヘッドとバッファコピーについてはそれほど自由度が高くないかもしれません。しかし、受信バイトのために手でメッセージを再構築しなければならないことに変わりはありません。

***

**[戻る](../index.html)**
