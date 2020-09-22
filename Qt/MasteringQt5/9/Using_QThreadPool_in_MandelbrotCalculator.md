# MandelbrotCalculatorでのQThreadPoolの使用

これで、Jobクラスの準備ができましたので、ジョブを管理するクラスを作成する必要があります。新しいクラス、MandelbrotCalculatorを作成してください。MandelbrotCalculator.hというファイルの中に必要なものを見てみましょう。

```C++
#include <QObject>
#include <QSize>
#include <QPointF>
#include <QElapsedTimer>
#include <QList>

#include "JobResult.h"

class Job;
class MandelbrotCalculator : public QObject
{
    Q_OBJECT

public:
    explicit MandelbrotCalculator(QObject *parent = 0);
    void init(QSize imageSize);

private:
    QPointF mMoveOffset;
    double mScaleFactor;
    QSize mAreaSize;
    int mIterationMax;
    int mReceivedJobResults;
    QList<JobResult> mJobResults;
    QElapsedTimer mTimer;
};
```

前のセクションでは、すでに mMoveOffset、mScaleFactor、mAreaSize、および mIterationMax について説明しました。また、いくつかの新しい変数もあります。

* mReceivedJobResults変数は、ジョブが送信したJobResultを受信した数を表します。
* mJobResults変数は、受信したJobResultを含むリストです。
* mTimer変数は、要求された画像のすべてのジョブを実行するまでの経過時間を計算します。

これで、すべてのメンバ変数のイメージがつかめたので、シグナル、スロット、プライベートメソッドを追加します。MandelbrotCalculator.hファイルを更新してください。

```C++
...
class MandelbrotCalculator : public QObject
{
    Q_OBJECT

public:
    explicit MandelbrotCalculator(QObject *parent = 0);
    void init(QSize imageSize);

signals:
    void pictureLinesGenerated(QList<JobResult> jobResults);
    void abortAllJobs();

public slots:
    void generatePicture(QSize areaSize, QPointF moveOffset,
    double scaleFactor, int iterationMax);
    void process(JobResult jobResult);

private:
    Job* createJob(int pixelPositionY);
    void clearJobs();

private:
    ...
};
```

これらの役割をご紹介します。

* generatePicture(): このスロットは、呼び出し元が新しいマンデルブロ画像を要求するために使用されます。この関数は、ジョブの準備と開始を行います。
* process(): このスロットは、ジョブによって生成された結果を処理します。
* pictureLinesGenerated(): このシグナルは、結果をディスパッチするために定期的にトリガされます。
* abortAllJobs(): このシグナルは、すべてのアクティブなジョブを中止するために使用されます。
* createJob(): これは、新しいジョブを作成して設定するためのヘルパー関数です。
* clearJobs(): このスロットは、キューに入っているジョブを削除し、アクティブなジョブをアボートします。

ヘッダーファイルが完成したので、実装を行います。ここからがMandelbrotCalculator.cppの実装の始まりです。

```C++
#include <QDebug>
#include <QThreadPool>

#include "Job.h"

const int JOB_RESULT_THRESHOLD = 10;

MandelbrotCalculator::MandelbrotCalculator(QObject *parent)
    : QObject(parent),
    mMoveOffset(0.0, 0.0),
    mScaleFactor(0.005),
    mAreaSize(0, 0),
    mIterationMax(10),
    mReceivedJobResults(0),
    mJobResults(),
    mTimer()
{
}
```

いつものように、メンバー変数のデフォルト値を持つイニシャライザリストを使用しています。JOB_RESULT_THRESHOLDの役割については、すぐに説明します。ここでは、generatePicture()スロットを使用しています。

```C++
void MandelbrotCalculator::generatePicture(QSize areaSize, QPointF
    moveOffset, double scaleFactor, int iterationMax)
{
    if (areaSize.isEmpty()) {
        return;
    }

    mTimer.start();
    clearJobs();

    mAreaSize = areaSize;
    mMoveOffset = moveOffset;
    mScaleFactor = scaleFactor;
    mIterationMax = iterationMax;

    for(int pixelPositionY = 0;
        pixelPositionY < mAreaSize.height(); pixelPositionY++) {
        QThreadPool::globalInstance()->
            start(createJob(pixelPositionY));
    }
}
```

areaSize次元が0x0であれば、何もすることはありません。リクエストが有効であれば、mTimerを起動して生成期間全体を追跡することができます。新しい画像生成のたびに、まず clearJobs() を呼び出して既存のジョブをキャンセルします。次に、提供されたものでメンバー変数を設定します。最後に、各垂直方向のピクチャラインに新しいJobクラスを作成します。Job*値を返すcreateJob()関数については、すぐに説明します。

QThreadPool::globalInstance()は、CPUのコア数に応じて最適なグローバルスレッドプールを提供してくれる静的関数です。すべてのジョブクラスに対して start() を呼び出しても、最初のクラスだけがすぐに起動します。他のクラスは、利用可能なスレッドを待ってプールキューに追加されます。

では、createJob() 関数を使って Job クラスがどのように作成されるか見てみましょう。

```C++
Job* MandelbrotCalculator::createJob(int pixelPositionY)
{
    Job* job = new Job();

    job->setPixelPositionY(pixelPositionY);
    job->setMoveOffset(mMoveOffset);
    job->setScaleFactor(mScaleFactor);
    job->setAreaSize(mAreaSize);
    job->setIterationMax(mIterationMax);

    connect(this, &MandelbrotCalculator::abortAllJobs,
        job, &Job::abort);

    connect(job, &Job::jobCompleted,
        this, &MandelbrotCalculator::process);

    return job;
}
```

ご覧のように、ジョブはヒープ上に割り当てられています。この操作は、MandelbrotCalculatorスレッドで時間がかかります。しかし、結果はそれに見合うだけの価値があります。QThreadPool::start()を呼び出すと、スレッドプールがjobの所有権を取得することに注意してください。その結果、Job::run()が終了すると、スレッドプールによって削除されます。マンデルブロアルゴリズムで必要とされるJobクラスの入力データを設定します。

その後、2つのコネクションが実行されます。

* abortAllJobs() シグナルを出すと、すべてのジョブの abort() スロットが呼び出されます。
* Jobがタスクを完了するたびに process() スロットが実行されます。

最後に、 Jobポインターが呼び出し元（この例では、generatePicture（）スロット）に返されます。

最後のヘルパー関数は clearJobs() です。これをMandelbrotCalculator.cppに追加してください。

```C++
void MandelbrotCalculator::clearJobs()
{
    mReceivedJobResults = 0;
    emit abortAllJobs();
    QThreadPool::globalInstance()->clear();
}
```

受信したジョブ結果のカウンタがリセットされます。アクティブなジョブをすべて中止するためのシグナルを発します。最後に、スレッドプールで利用可能なスレッドを待っているキューに入っているジョブを削除します。

このクラスの最後の関数はprocess()で、おそらく最も重要な関数です。以下のスニペットでコードを更新してください。

```C++
void MandelbrotCalculator::process(JobResult jobResult)
{
    if (jobResult.areaSize != mAreaSize ||
            jobResult.moveOffset != mMoveOffset ||
            jobResult.scaleFactor != mScaleFactor) {
        return;
    }

    mReceivedJobResults++;
    mJobResults.append(jobResult);

    if (mJobResults.size() >= JOB_RESULT_THRESHOLD ||
            mReceivedJobResults == mAreaSize.height()) {
        emit pictureLinesGenerated(mJobResults);
        mJobResults.clear();
    }

    if (mReceivedJobResults == mAreaSize.height()) {
        qDebug() << "Generated in " << mTimer.elapsed() << " ms";
    }
}
```

このスロットは、ジョブがそのタスクを完了するたびに呼び出されます。最初にチェックするのは、現在の入力データで現在のJobResultがまだ有効であるかどうかです。新しい画像が要求されたら、ジョブキューをクリアしてアクティブなジョブを中止します。しかし、古いJobResultがまだこのprocess()スロットに送られている場合は無視しなければなりません。

その後、mReceivedJobResultsカウンタをインクリメントし、このJobResultsをメンバーキューであるmJobResultsに追加します。計算機は、pictureLinesGenerated()シグナルを発してそれらをディスパッチする前に、JOB_RESULT_THRESHOLD(つまり、10)結果を取得するのを待ちます。この値は注意して微調整してみてください。

* 低い値、例えば1は、電卓がデータを取得するとすぐにウィジェットにデータの各行をディスパッチします。しかし、ウィジェットは各行を処理するために電卓よりも遅くなります。さらに、ウィジェットのイベントループをフラッディングしてしまいます。
* 値を高くすると、ウィジェットのイベントループが緩和されます。しかし、ユーザは何かが起こるのを見るまで長く待つことになります。連続的な部分フレーム更新は、より良いユーザーエクスペリエンスを提供します。

また、イベントがディスパッチされると、ジョブ結果のあるQListクラスがコピーで送られてくることにも注目してください。しかし、QtはQListと暗黙の共有を行っているので、コストのかかる深いコピーではなく、浅いコピーを送るだけです。そして、電卓の現在のQListをクリアします。

最後に、処理されたJobResultが領域内の最後のものである場合、ユーザがgeneratePicture()を呼び出してからの経過時間を表示したデバッグメッセージを表示します。

***

## Qt Tip

QThreadPoolクラスが使用するスレッド数は、setMaxThreadCount(x)で設定できます。

***

**[戻る](../index.html)**
