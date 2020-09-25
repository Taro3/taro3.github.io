# SDK を使って基礎を固める

最初のステップは、アプリケーションとワーカーの間で共有されるクラスを実装することです。そのためには、カスタムSDKに頼ることになります。このテクニックについての記憶を新たにする必要がある場合は、第8章「Animations-It's Alive, Alive!

備忘録として、SDKを説明する図を示します。

![image](img/6.png)

それぞれの部品の仕事内容を説明してみましょう。

* コンポーネントは、アプリケーションとワーカーの間で交換される情報の一部をカプセル化します。
* JobRequestコンポーネントには、適切なジョブをワーカーに派遣するために必要な情報が含まれています。
* JobResultコンポーネントには、与えられた行のマンデルブロ集合の計算結果が含まれます。
* MessageUtils コンポーネントには、TCP ソケットを介してデータをシリアライズ/デシリアライズするためのヘルパー関数が含まれています。

これらのファイルはすべて、IPC メカニズムの各サイド（アプリケーションとワーカー）からアクセス可能でなければなりません。SDKにはヘッダファイルのみが含まれていることに注意してください。これは、SDK の使用法を単純化するために行ったものです。

まずはsdkディレクトリにあるMessageの実装から始めてみましょう。以下の内容のMessage.hファイルを作成します。

```C++
#include <QByteArray>

struct Message {
    enum class Type {
        WORKER_REGISTER,
        WORKER_UNREGISTER,
        ALL_JOBS_ABORT,
        JOB_REQUEST,
        JOB_RESULT,
    };

    Message(const Type type = Type::WORKER_REGISTER,
        const QByteArray& data = QByteArray()) :
        type(type),
        data(data)
    {
    }

    ~Message() {}

    Type type;
    QByteArray data;
};
```

最初に注意すべきなのは、すべての可能なメッセージタイプを詳細に記述した enum class Type です。

* WORKER_REGISTER: ワーカーが最初にアプリケーションに接続したときに送信するメッセージです。メッセージの内容は、ワーカーのCPUのコア数だけです。これがなぜ便利なのかは、すぐに見ていきましょう。
* WORKER_UNREGISTER: これは、ワーカーが切断されたときに送られるメッセージです。これは、アプリケーションがこのワーカーをリストから削除し、メッセージの送信を停止すべきであることを知らせます。
* ALL_JOBS_ABORT: これは、ピクチャ生成がキャンセルされるたびにアプリケーションから送信されるメッセージです。その後、ワーカーは現在のローカルスレッドをすべてキャンセルする責任があります。
* JOB_REQUEST: これは、所望のピクチャの特定のラインを計算するためにアプリケーションによって送信されるメッセージである。
* JOB_RESULT: これは、JOB_REQUEST入力から計算された結果をワーカーが送信するメッセージです。

C++11 で追加された enum クラス型について簡単に説明します。これは enum のより安全なバージョンです（最初からあるべき姿が enum であると言う人もいるかもしれません）。

* 値のスコープはローカルです。この例では、Message::Type::WORKER_REGISTERという構文でしか列挙型の値を参照できません。この制限の良いところは、念のためにenum値の前にMESSAGE_TYPE_を付ける必要がないことです。名前が他の何かと競合しないことを確認してください。
* 暗黙のうちにintに変換されることはありません。enumクラスは実在の型のように振る舞います。enumクラスをintにキャストするには、static_cast\<int\>(Message::Type::WORKER_REGISTER)と書かなければなりません。
* 順方向の宣言はありません。enum classがchar型であることを指定することはできますが（enum class Test : char { ... }という構文で）、コンパイラは前方宣言ではenumクラスのサイズを推論することができません。そのため、単純に禁止されています。

私たちは可能な限りenumクラスを使用する傾向がありますが、これはQtのenumの使用法と衝突しない場合を意味します。

ご覧のように、メッセージには2人のメンバーしかいません。

* type: これは先ほど説明したメッセージタイプです。
* data: これは、送信される情報のピースを含む不透明なタイプです。

シリアライズ/デシリアライズの責任をMessage呼び出し元に負わせるために、dataを非常に汎用的なものにすることにしました。メッセージのtypeに基づいて、彼らはメッセージの内容を読み書きする方法を知っていなければなりません。

このアプローチを使うことで、MessageRegisterやMessageUnregisterなどのもつれたクラス階層を避けることができます。新しい Message type を追加するのは、単に Type enum class に値を追加し、適切なシリアライズ/デシリアライズを data で行うだけです (これはいずれにせよ行わなければなりません)。

Qt Creatorで見るには、ch10-mandelbrot-ipc.proのMessage.hを忘れずに追加してください。

```QMake
OTHER_FILES += \
sdk/Message.h
```

次のヘッダはJobRequest.hです。

```C++
#include <QSize>
#include <QPointF>

struct JobRequest
{
    int pixelPositionY;
    QPointF moveOffset;
    double scaleFactor;
    QSize areaSize;
    int iterationMax;
};

Q_DECLARE_METATYPE(JobRequest)

// In ch10-mandelbrot-ipc
OTHER_FILES += \
    sdk/Message.h \
    sdk/JobRequest.h
```

この struct 要素には、ワーカーが対象のマンデルブロ画像の線を計算するために必要なすべてのデータが含まれています。アプリケーションとワーカーは異なるメモリ空間(あるいは異なる物理マシン)に存在するため、マンデルブロ集合を計算するためのパラメータは何らかの方法で送信されなければなりません。これがJobRequestの目的です。各フィールドの意味は、第9章「マルチスレッドで正気を保つ」のJobResultと同じです。

Q_DECLARE_METATYPE(JobRequest) マクロの存在に注意してください。このマクロは、QtメタオブジェクトシステムにJobRequestを知らせるために使用されます。これはQVariantと組み合わせてクラスを使えるようにするために必要です。QVariantを直接使用するのではなく、QVariantに依存するQDataStreamを使用します。

JobResultといえば、こちらが新しいJobResult.hです。

```C++
#include <QSize>
#include <QVector>
#include <QPointF>

struct JobResult
{
    JobResult(int valueCount = 1) :
        areaSize(0, 0),
        pixelPositionY(0),
        moveOffset(0, 0),
        scaleFactor(0.0),
        values(valueCount)
    {
    }

    QSize areaSize;
    int pixelPositionY;
    QPointF moveOffset;
    double scaleFactor;

    QVector<int> values;
};

Q_DECLARE_METATYPE(JobResult)

// In ch10-mandelbrot-ipc
OTHER_FILES += \
    sdk/Message.h \
    sdk/JobRequest.h \
    sdk/JobResult.h
```

新しいバージョンは、第9章「マルチスレッドで正気を保つ」のプロジェクト例の恥知らずなコピーペーストです (小さな Q_DECLARE_METATYPE を追加しています)。

***

**[戻る](../index.html)**
