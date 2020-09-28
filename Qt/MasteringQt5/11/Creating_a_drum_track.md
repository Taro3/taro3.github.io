# ドラムトラックの作成

それでは、このプロジェクトに参加してみましょう。ch11-drum-machineという名前の新しいQtウィジェットアプリケーションプロジェクトを作成します。いつものように、ch11-drummachine.proにCONFIG += c++14を追加します。

では、SoundEventという名前の新しいC++クラスを作成します。ここでは、SoundEvent.hを関数から剥ぎ取っています。

```C++
#include <QtGlobal>

class SoundEvent
{
public:
    SoundEvent(qint64 timestamp = 0, int soundId = 0);
    ~SoundEvent();
    qint64 timestamp;
    int soundId;
};
```

このクラスには2つのパブリック・メンバしか含まれていません。

* timestamp: トラックの開始からのSoundEventの現在時刻をミリ秒単位で含むqint64 (long long型)
* soundId: 再生された音のID

録音モードでは、ユーザーが音を再生するたびに、適切なデータでSoundEventが作成されます。SoundEvent.cppファイルがつまらないので、ここでは煽りません。

次に作るクラスはTrackです。ここでも新しいC++クラスを作成します。Track.hをメンバーだけで復習してみましょう。

```C++
#include <QObject>
#include <QVector>
#include <QElapsedTimer>

#include "SoundEvent.h"

class Track : public QObject
{
    Q_OBJECT

public:
    enum class State {
        STOPPED,
        PLAYING,
        RECORDING,
    };

    explicit Track(QObject *parent = 0);
    ~Track();

private:
    qint64 mDuration;
    std::vector<std::unique_ptr<SoundEvent>> mSoundEvents;
    QElapsedTimer mTimer;
    State mState;
    State mPreviousState;
};
```

これで、彼らについて詳しく知ることができるようになりました。

* mDuration: この変数は、Track クラスの継続時間を保持します。このメンバは、録音が開始されると0にリセットされ、録音が停止されると更新されます。
* mSoundEvents: この変数は、与えられた Track のサウンドイベントのリストです。unique_ptr のセマンティックにあるように、Track がサウンドイベントの所有者である。
* mTimer: この変数は、トラックが再生または録音されるたびに開始されます。
* mState: この変数は現在のTrackクラスの状態を表し、3つの値を持つことができます。STOPPED, PLAYING, RECORDINGの3つの値があります。
* mPreviousState: この変数は、トラックの前の状態を表します。これは、新しいSTOPPEDStateでどのアクションをすればいいのかを知りたいときに便利です。mPreviousStateがPLAYING状態の場合は再生を停止しなければなりません。

Track クラスは、プロジェクトのビジネスロジックの支点となるクラスです。これはアプリケーション全体の状態を表す mState を保持しています。その内容は、あなたの素晴らしい演奏の再生中に読み込まれ、ファイルにもシリアライズされます。

Track.hに関数を追加してみましょう。

```C++
class Track : public QObject
{
    Q_OBJECT

public:
    ...
    qint64 duration() const;
    State state() const;
    State previousState() const;
    quint64 elapsedTime() const;
    const std::vector<std::unique_ptr<SoundEvent>>& soundEvents() const;

signals:
    void stateChanged(State state);

public slots:
    void play();
    void record();
    void stop();
    void addSoundEvent(int soundEventId);

private:
    void clear();
    void setState(State state);

private:
    ...
};
```

簡単なゲッターは飛ばして、重要な機能に集中します。

* elapsedTime(): この関数は、mTimer.elapsed()の値を返します。
* soundEvents(): この関数は少し複雑なゲッターです。トラッククラスはmSoundEventsの内容の所有者であり、それを強制したいのです。そのために、ゲッターはmSoundEventsにconst &を返します。
* stateChanged(): この関数は、mStateの値が更新されたときに発行されます。新しいStateはパラメータとして渡されます。
* play(): この機能は、トラックの再生を開始するスロットです。この再生は純粋に論理的なもので、実際の再生はPlaybackWorkerによってトリガーされます。
* record(): この関数は、Trackの記録状態を開始するスロットです。
* stop(): この関数は、現在の開始状態またはレコード状態を停止させるスロットです。
* addSoundEvent(): この関数は、指定された soundId を持つ新しい SoundEvent を作成し、それを mSoundEvents に追加します。
* clear(): この関数は、トラックの内容をリセットする：mSoundEvents をクリアし、mDuration を 0 に設定する。
* setState(): この関数は、mState, mPreviousStateを更新し、stateChanged()シグナルを発するプライベートヘルパー関数です。

ヘッダーがカバーされたので、Track.cppの面白いところを勉強していきましょう。

```C++
void Track::play()
{
    setState(State::PLAYING);
    mTimer.start();
}
```

Track.play()を呼び出すと、状態がPLAYINGに更新され、mTimerが起動します。TrackクラスはQt Multimedia APIに関連する何かを保持しているわけではなく、進化したデータホルダーに限定されています（ステートも管理しているので）。

今度はrecord（）について、多くの驚きをもたらします。

```C++
void Track::record()
{
    clearSoundEvents();
    setState(State::RECORDING);
    mTimer.start();
}
```

データをクリアすることから始まり、状態をRECORDINGに設定し、mTimerを起動します。ここで、少し変わったstop()を考えてみましょう。

```C++
void Track::stop()
{
    if (mState == State::RECORDING) {
        mDuration = mTimer.elapsed();
    }
    setState(State::STOPPED);
}
```

RECORDING状態で停止している場合は、mDurationが更新されます。ここでは、何も特別なことはしていません。setState()の呼び出しを、本体を見ずに3回見ました。

```C++
void Track::setState(Track::State state)
{
    mPreviousState = mState;
    mState = state;
    emit stateChanged(mState);
}
```

mStateの現在値は、更新される前にmPreviousStateに格納されます。最後に、新しい値で stateChanged() が発行されます。

Trackのステートシステムは完全にカバーしています。最後に足りないのはSoundEventsのインタラクションです。まずはaddSoundEvent()スニペットから。

```C++
void Track::addSoundEvent(int soundEventId)
{
    if (mState != State::RECORDING) {
        return;
    }
    mSoundEvents.push_back(make_unique<SoundEvent>(
                           mTimer.elapsed(),
                           soundEventId));
}
```

SoundEventは、RECORDING状態になっている場合にのみ作成されます。その後、mTimerの現在の経過時間と渡されたsoundEventIdでmSoundEventsにSoundEventを追加します。

さて、clear()関数です。

```C++
void Track::clear()
{
    mSoundEvents.clear();
    mDuration = 0;
}
```

mSoundEventsではunique_ptr\<SoundEvent\>を使用しているので、mSoundEvents.clear()関数でベクトルを空にして、各SoundEventも削除するだけで十分です。これでスマートポインタで悩むことが一つ減りました。

SoundEventとTrackは、未来のビートに関する情報を保持する基底クラスです。このデータを読み込んで再生する役割を担うクラスを見ていきます。PlaybackWorkerです。

新しいC++クラスを作成し、PlaybackWorker.hをこのように更新します。

```C++
#include <QObject>
#include <QAtomicInteger>

class Track;

class PlaybackWorker : public QObject
{
    Q_OBJECT

public:
    explicit PlaybackWorker(const Track& track, QObject *parent = 0);

signals:
    void playSound(int soundId);
    void trackFinished();

public slots:
    void play();
    void stop();

private:
    const Track& mTrack;
    QAtomicInteger<bool> mIsPlaying;
};
```

PlaybackWorker クラスは別のスレッドで実行されます。メモリをリフレッシュする必要がある場合は、第9章の「マルチスレッドで正気を保つ」を参照してください。PlaybackWorker クラスの役割は、Track クラスのコンテンツを反復処理してサウンドをトリガーすることです。このヘッダを分解してみましょう。

* mTrack: この関数は、PlaybackWorkerが作業しているTrackクラスへの参照です。コンストラクタでは const 参照として渡されます。この情報があれば、PlaybackWorkerがmTrackを変更できないことがわかります。
* mIsPlaying: この関数は、他のスレッドからワーカーを停止できるようにするために使用されるフラグです。変数へのアトミックアクセスを保証するためのQAtomicIntegerです。
* playSound(): この関数は、サウンドを再生するたびに PlaybackWorker から発行されます。
* trackFinished(): 最後まで再生した場合に発せられます。途中で停止している場合は、この信号は発せられません。
* play(): この機能はPlaybackWorkerのメイン機能です。この関数では、mTrackの内容を照会してサウンドをトリガします。
* stop(): この関数は、mIsPlaying フラグを更新して play() をループ終了させる関数です。

このクラスの本質は play() 関数にあります。

```C++
void PlaybackWorker::play()
{
    mIsPlaying.store(true);
    QElapsedTimer timer;
    size_t soundEventIndex = 0;
    const auto& soundEvents = mTrack.soundEvents();

    timer.start();
    while(timer.elapsed() <= mTrack.duration()
            && mIsPlaying.load()) {
        if (soundEventIndex < soundEvents.size()) {
            const auto& soundEvent =
                                soundEvents.at(soundEventIndex);

            if (timer.elapsed() >= soundEvent->timestamp) {
                emit playSound(soundEvent->soundId);
                soundEventIndex++;
            }
        }
        QThread::msleep(1);
    }

    if (soundEventIndex >= soundEvents.size()) {
        emit trackFinished();
    }
}
```

play()関数が最初に行うことは、読み込みの準備です。mIsPlayingがtrueに設定され、QElapsedTimerクラスが宣言され、soundEventIndexが初期化されます。timer.elapsed()が呼ばれるたびに、音が再生されるべきかどうかを知ることができます。

どの音を再生するかを知るために、 soundEventIndex を使用して soundEvents ベクトルのどこにいるかを知ることができます。

その直後に、timerオブジェクトが開始され、whileループに入ります。このwhileループには、継続するための条件が2つあります。

* timer.elapsed() <= mTrack.duration(): この状態では、トラックの演奏が終了しなかったことになります。
* mIsPlaying.load(): この条件は true を返します: 誰も PlaybackWorker の停止を要求しませんでした。

直感的には、while条件の中にsoundEventIndex < soundEvents.size()条件を追加したかもしれません。そうすることで、最後の音が再生されたらすぐにPlaybackWorkerを終了させることができます。技術的にはこれで動作しますが、これではユーザが録音した音が尊重されません。

複雑なビートを作成し（4つの音で何ができるか侮ってはいけません！）、曲の最後に5秒間の長い一時停止を決めたユーザーを考えてみましょう。彼が停止ボタンをクリックすると、時間表示は00:55（55秒間）と表示されます。しかし、彼の演奏を再生すると、最後の音は00:50で終了。再生は00:50で停止し、番組は彼が録音したものを尊重しない。

このため、soundEventIndex < size() テストは while ループの中に移動され、読み込んだ soundEvents のヒューズとしてのみ使用されます。

この条件の中で、現在の soundEvent への参照を取得します。次に、経過時間を soundEvent のタイムスタンプと比較します。timer.elapsed() が soundEvent->timestamp 以上の場合、シグナル playSound() が soundId と一緒に発せられます。

これはあくまでも音を再生するためのリクエストです。PlaybackWorker クラスは soundEvents を読み込んで、適切なタイミングで playSound() をトリガーすることに限定しています。実際の音は後からSoundEffectWidgetクラスで処理します。

whileループの各イテレーションでは、ビジーループを避けるためにQThread::msleep(1)が行われます。スリープを最小限にしているのは、元のスコアにできるだけ忠実に再生したいからです。スリープが長ければ長いほど、再生タイミングに不一致が生じる可能性が高くなります。

最後に、全てのサウンドイベントが処理されていれば、trackFinished信号が出力されます。

***

**[戻る](../index.html)**
