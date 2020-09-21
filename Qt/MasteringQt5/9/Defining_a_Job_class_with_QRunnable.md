# QRunnableでジョブクラスを定義する

プロジェクトの核心部分に飛び込んでみましょう。マンデルブロ画像生成を高速化するために、計算全体を複数のジョブに分割します。Jobとは、タスクのリクエストです。CPUアーキテクチャにもよりますが、複数のジョブが同時に実行されます。Jobクラスは、結果値を含むJobResult関数を生成します。このプロジェクトでは、Jobクラスは、画像全体の1行分の値を生成します。例えば、800 x 600 の画像解像度の場合、600 個のジョブが必要で、それぞれが 800 個の値を生成します。

JobResult.hというC++ヘッダーファイルを作成してください。

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
```

この構造体には 2 つの部分があります。

* 入力データ（areaSize, pixelPositionY, ...)
* 結果 Job クラスによって生成された values です。

これで Job クラス自体を作成することができます。次のJob.hのスニペットを内容に使用して、C++クラスJobを作成します。

```C++
#include <QObject>
#include <QRunnable>

#include "JobResult.h"

class Job : public QObject, public QRunnable
{
    Q_OBJECT

public:
    Job(QObject *parent = 0);
    void run() override;
};
```

このJobクラスはQRunnableなので、run()をオーバーライドしてマンデルブロ・ピクチャーアルゴリズムを実装することができます。ご覧のように、Job は QObject を継承しているので、Qt の signal/slot 機能を使用することができます。アルゴリズムはいくつかの入力データを必要とします。Job.h をこのように更新してください。

```C++
#include <QObject>
#include <QRunnable>
#include <QPointF>
#include <QSize>
#include <QAtomicInteger>

class Job : public QObject, public QRunnable
{
    Q_OBJECT

public:
    Job(QObject *parent = 0);
    void run() override;
    void setPixelPositionY(int value);
    void setMoveOffset(const QPointF& value);
    void setScaleFactor(double value);
    void setAreaSize(const QSize& value);
    void setIterationMax(int value);

private:
    int mPixelPositionY;
    QPointF mMoveOffset;
    double mScaleFactor;
    QSize mAreaSize;
    int mIterationMax;
};
```

これらの変数の話をしましょう。

* mPixelPositionY変数はピクチャの高さインデックスです。それぞれのJobは1つのピクチャ行だけのデータを生成するので、この情報が必要です。
* mMoveOffset 変数はマンデルブロ原点オフセットです。ユーザはピクチャをパンすることができるので、原点は常に(0, 0)ではありません。
* mScaleFactor変数はマンデルブロのスケール値です。ユーザは画像をズームインすることもできます。
* mAreaSize変数は、ピクセル単位の最終的な画像サイズです。 
* mIterationMax変数は、1ピクセルのマンデルブロー結果を決定するために許可された反復回数です。

これで、Job.hにシグナル、jobCompleted()、アボート機能を追加することができます。

```C++
#include <QObject>
#include <QRunnable>
#include <QPointF>
#include <QSize>
#include <QAtomicInteger>

#include "JobResult.h"

class Job : public QObject, public QRunnable
{
    Q_OBJECT

public:
    ...

signals:
    void jobCompleted(JobResult jobResult);

public slots:
    void abort();

private:
    QAtomicInteger<bool> mAbort;
    ...
};
```

アルゴリズムが終了すると、jobCompleted() シグナルが出力されます。jobResult パラメータには結果値が格納されます。abort() スロットは、mIsAbort フラグの値を更新しているジョブを停止させることができます。mAbortは古典的なboolではなく、QAtomicInteger\<bool\>であることに注意してください。この Qt のクロスプラットフォーム型により、アトミックな操作を中断することなく実行することができます。mutex や他の同期機構を使用してもよいのですが、アトミック変数を使用することで、異なるスレッドから安全に変数を更新したりアクセスしたりするための高速な方法です。

Job.cppで実装部分に切り替えましょう。ここに Job クラスのコンストラクタがあります。

```C++
#include "Job.h"

Job::Job(QObject* parent) :
    QObject(parent),
    mAbort(false),
    mPixelPositionY(0),
    mMoveOffset(0.0, 0.0),
    mScaleFactor(0.0),
    mAreaSize(0, 0),
    mIterationMax(1)
{
}
```

これは古典的な初期化です。QObjectのコンストラクタを呼び出すことを忘れないでください。

これで、run()関数を実装できるようになりました。

```C++
void Job::run()
{
    JobResult jobResult(mAreaSize.width());
    jobResult.areaSize = mAreaSize;
    jobResult.pixelPositionY = mPixelPositionY;
    jobResult.moveOffset = mMoveOffset;
    jobResult.scaleFactor = mScaleFactor;
    ...
}
```

この最初の部分では、JobResultの変数を初期化します。領域サイズの幅を利用して、JobResult::valueを正しい初期サイズのQVectorとして構築します。その他の入力データは、JobからJobResultにコピーして、JobResultの受信側にコンテキスト入力データで結果を取得させるようにしています。

そして、run()関数をマンデルブロアルゴリズムで更新することができます。

```C++
void Job::run()
{
    ...
    double imageHalfWidth = mAreaSize.width() / 2.0;
    double imageHalfHeight = mAreaSize.height() / 2.0;
    for (int imageX = 0; imageX < mAreaSize.width(); ++imageX) {
        int iteration = 0;
        double x0 = (imageX - imageHalfWidth)
            * mScaleFactor + mMoveOffset.x();
        double y0 = (mPixelPositionY - imageHalfHeight)
            * mScaleFactor - mMoveOffset.y();
        double x = 0.0;
        double y = 0.0;
        do {
            if (mAbort.load()) {
                return;
            }

            double nextX = (x * x) - (y * y) + x0;
            y = 2.0 * x * y + y0;
            x = nextX;
            iteration++;

        } while(iteration < mIterationMax
            && (x * x) + (y * y) < 4.0);

        jobResult.values[imageX] = iteration;
    }

    emit jobCompleted(jobResult);
}
```

マンデルブロアルゴリズム自体は本書の範囲を超えています。しかし、このrun()関数の主な目的を理解する必要があります。それを分解してみましょう。

* for ループは、1 本の線上のピクセルのすべての x 位置を繰り返し処理します。
* ピクセル位置は複素平面座標に変換されます。
* 試行回数が許可された最大反復回数を超えた場合、アルゴリズムはmIterationMax値に対するiterationで終了します。
* マンデルブロチェック条件が真の場合、アルゴリズムは iteration < mIterationMax で終了します。
* いずれにしても、各ピクセルについて、反復回数は JobResult の values に格納されます。
* 最後に、このアルゴリズムの結果値とともに jobCompleted() シグナルが放出されます。
* mAbort.load()でアトミックリードを実行します。戻り値がtrueの場合、アルゴリズムは中止され、何も出力されないことに注意してください。

最後の関数はabort()スロットです。

```C++
void Job::abort()
{
    mAbort.store(true);
}
```

このメソッドは、値 true のアトミック書き込みを実行します。アトミックなメカニズムにより、run() 関数での mAbort の読み込みを中断することなく、複数のスレッドから abort() を呼び出すことができます。

この場合、run()はQThreadPoolの影響を受けるスレッド内に存在します（これについては近日中に説明します）が、abort()スロットはMandelbrotCalculatorのスレッドコンテキスト内で呼び出されます。

mAbort上の操作をQMutexで確保した方がいいかもしれません。しかし、ミューテックスのロックとアンロックを頻繁に行うと、コストのかかる操作になることを覚えておいてください。ここでQAtomicIntegerクラスを使用することで、利点だけが得られます。mAbortへのアクセスはスレッドセーフであり、高価なロックを避けることができます。

Jobの実装の最後にはセッター関数のみが含まれています。疑問がある場合は、完全なソースコードを参照してください。

***

**[戻る](../index.html)**
