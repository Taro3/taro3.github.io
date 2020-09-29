# PlaybackWorkerに命を吹き込む

ユーザーはマウスのクリックやキーボードのキーで音を生演奏することができます。しかし、素晴らしいビートを録音したときには、アプリケーションはPlaybackWorkerクラスを使ってそれを再生できるようにしなければなりません。MainWindowがこのワーカーをどのように使っているか見てみましょう。以下は、PlaybackWorkerクラスに関連するMainWindow.hです。

```C++
class MainWindow : public QMainWindow
{
...
private slots:
    void playSoundEffect(int soundId);
    void clearPlayback();
    void stopPlayback();
    ...

private:
    void startPlayback();
    ...

private:
    PlaybackWorker* mPlaybackWorker;
    QThread* mPlaybackThread;
    ...
};
```

ご覧の通り、MainWindowにはPlaybackWorkerとQThreadメンバ変数があります。startPlayback()の実装を見てみましょう。

```C++
void MainWindow::startPlayback()
{
    clearPlayback();

    mPlaybackThread = new QThread();

    mPlaybackWorker = new PlaybackWorker(mTrack);
    mPlaybackWorker->moveToThread(mPlaybackThread);

    connect(mPlaybackThread, &QThread::started,
        mPlaybackWorker, &PlaybackWorker::play);
    connect(mPlaybackThread, &QThread::finished,
        mPlaybackWorker, &QObject::deleteLater);

    connect(mPlaybackWorker, &PlaybackWorker::playSound,
        this, &MainWindow::playSoundEffect);

    connect(mPlaybackWorker, &PlaybackWorker::trackFinished,
        &mTrack, &Track::stop);

    mPlaybackThread->start(QThread::HighPriority);
}
```

全てのステップを分析してみましょう。

1. clearPlayback()関数を使って現在の再生をクリアします。
2. 新しいQThreadとPlaybackWorkerを構築します。現在のトラックは、この時点でワーカーに与えられています。いつものように、ワーカーは専用スレッドに移動されます。
3. できるだけ早くトラックを再生したい。そこで、QThreadがstart()シグナルを出すと、PlaybackWorker::play()スロットが呼び出されます。
4. PlaybackWorkerのメモリを気にしないようにしたい。そこで、QThread が終了して finished() シグナルを送信したら、QObject::deleteLater() スロットを呼び出して、ワーカーを削除するようにスケジューリングしています。
5. PlaybackWorker クラスがサウンドを再生する必要がある場合、 playSound() シグナルが放出され、MainWindow:playSoundEffect() スロットが呼び出されます。
6. 最後の接続は、PlaybackWorkerクラスがトラック全体の再生を終了するときに行います。trackFinished()シグナルが出て、Track::Stop()スロットを呼び出します。
7. 最後に、優先度の高いスレッドが開始されます。オペレーティングシステムの中には、スレッドの優先順位をサポートしていないものがあることに注意してください。

これでstopPlayback()の本体が見えてきました。

```C++
void MainWindow::stopPlayback()
{
    mPlaybackWorker->stop();
    clearPlayback();
}
```

PlaybackWorkerのstop()関数をスレッドから呼び出しています。stop() で QAtomicInteger を使用しているので、この関数はスレッドセーフで直接呼び出すことができます。最後に、ヘルパー関数 clearPlayback() を呼び出します。clearPlayback()を使うのは2回目なので、実装してみましょう。

```C++
void MainWindow::clearPlayback()
{
    if (mPlaybackThread) {
        mPlaybackThread->quit();
        mPlaybackThread->wait(1000);
        mPlaybackThread = nullptr;
        mPlaybackWorker = nullptr;
    }
}
```

ここでのサプライズはありません。スレッドが有効であれば、スレッドを終了させて1秒待ちます。そして、スレッドとワーカーをnulllptrに設定します。

PlaybackWorker::PlaySoundシグナルはMainWindow::playSoundEffect()に接続されています。ここでは、その実装を紹介します。

```C++
void MainWindow::playSoundEffect(int soundId)
{
    mSoundEffectWidgets[soundId]->triggerPlayButton();
}
```

このスロットは soundId に対応する SoundEffectWidget クラスを取得します。そして、キーボードのトリガーキーを押したときに呼び出されるのと同じメソッド、triggerPlayButton()を呼び出します。

ボタンをクリックしたり、キーを押したり、PlaybackWorker クラスがサウンドの再生を要求すると、SoundEffectWidget の QPushButton が clicked() というシグナルを発します。このシグナルは SoundEffectWidget::play() スロットに接続されています。次のスニペットでは、このスロットについて説明します。

```C++
void SoundEffectWidget::play()
{
    mSoundEffect.play();
    emit soundPlayed(mId);
}
```

ここには何もありません。すでに説明したQSoundEffectのplay()関数を呼び出します。最後に、録音中の状態であれば、トラックが新しいサウンドイベントを追加するために使用する soundPlayed()シグナルを発します。

***

**[戻る](../index.html)**
