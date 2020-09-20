# Qt のマルチスレッド技術の上を飛ぶ

QThread をベースに構築された Qt では、いくつかのスレッディング技術が利用可能です。まず、スレッドを同期させるために、通常のアプローチは、与えられたリソースに対して相互排他（ミューテックス）を使用して相互排他を持つことです。Qt は QMutex クラスを使ってそれを提供しています。使い方は簡単です。

```C++
QMutex mutex;
int number = 1;

mutex.lock();
number *= 2;
mutex.unlock();
```

mutex.lock()命令から、mutexをロックしようとする他のスレッドは、mutex.unlock()が呼ばれるまで待ちます。

ロック/アンロック機構は複雑なコードではエラーが発生しやすいです。特定の終了条件でミューテックスのロックを解除するのを忘れてしまい、デッドロックが発生してしまうことがあります。この状況を単純化するために、QtはQMutexをロックする必要がある場合に使用するQMutexLockerを提供しています。

```C++
QMutex mutex;
QMutexLocker locker(&mutex);

int number = 1;
number *= 2;
if (overlyComplicatedCondition) {
    return;
} else if (notSoSimple) {
    return;
}
```

mutexは、lockerオブジェクトが作成された時にロックされ、lockerオブジェクトが破壊された時、例えばスコープ外になった時にロックが解除されます。これは、return文が出現するところで述べたすべての条件に当てはまります。これにより、コードがよりシンプルになり、より読みやすくなります。

QThreadインスタンスを手作業で管理するのは面倒になるため、頻繁にスレッドを作成したり破棄したりする必要があるかもしれません。そのためには、再利用可能な QThread のプールを管理する QThreadPool クラスを使用することができます。

QThreadPool クラスによって管理されているスレッド内でコードを実行するには、先ほど説明したワーカーに非常に近いパターンを使用します。主な違いは、処理クラスが QRunnable クラスを拡張しなければならないことです。次のようになります。

```C++
class Job : public QRunnable
{
    void run()
    {
        // long running operation
    }
}

Job* job = new Job();
QThreadPool::globalInstance()->start(job);
```

run() 関数をオーバーライドして、QThreadPool に別のスレッドでジョブを実行するように依頼するだけです。QThreadPool::globalInstance() は静的なヘルパー関数で、アプリケーションのグローバルインスタンスにアクセスすることができます。QThreadPool のライフサイクルをより細かく制御したい場合は、独自の QThreadPool を作成することができます。

QThreadPool::start() 関数は job の所有権を取得し、run() が終了すると自動的に削除されることに注意してください。注意してください、これは QObject::moveToThread() がワーカーに対して行うようなスレッドの親和性を変更するものではありません。QRunnable クラスは再利用できません。

複数のジョブを起動した場合、QThreadPool は CPU のコア数に基づいて理想的なスレッド数を自動的に割り当てます。QThreadPool クラスが起動できる最大スレッド数は、QThreadPool::maxThreadCount() で取得できます。

***

## Tips

手作業でスレッドを管理する必要があるが、CPUのコア数を基準にしたい場合は、QThreadPool::idealThreadCount()という便利な静的関数を使うことができます。

***

マルチスレッド開発のもう一つのアプローチとして、Qt Concurrent フレームワークがあります。これは、ミューテックス/ロック/ウェイト条件の使用を避け、CPUコア間での処理の分散を促進する高レベルAPIです。

Qt Concurrent は QFuture クラスに依存して関数を実行し、後で結果を期待します。

```C++
void longRunningFunction();
QFuture<void> future = QtConcurrent::run(longRunningFunction);
```

longRunningFunction() 関数は、デフォルトの QThreadPool クラスから取得した分離スレッドで実行されます。

QFutureクラスにパラメータを渡して操作結果を取得するには、以下のコードを使用します。

```C++
QImage processGrayscale(QImage& image);
QImage lenna;

QFuture<QImage> future = QtConcurrent::run(processGrayscale,
    lenna);
QImage grayscaleLenna = future.result();
```

ここでは、processGrayscale() 関数のパラメータとして lenna を渡しています。結果として QImage が必要なので、QFuture クラスを QImage というテンプレート型で宣言します。その後、future.result() は現在のスレッドをブロックし、操作が完了するのを待って最終的な QImage を返します。

ブロッキングを避けるために、QFutureWatcherが助けに来る。

```C++
QFutureWatcher<QImage> watcher;
connect(&watcher, &QFutureWatcher::finished,
    this, &QObject::handleGrayscale);

QImage processGrayscale(QImage& image);
QImage lenna;
QFuture<QImage> future = QtConcurrent::run(processImage, lenna);
watcher.setFuture(future);
```

QFutureWatcherクラスをQFutureで使われているものと同じテンプレート引数で宣言することから始めます。そして、操作が完了したときに呼び出されたいスロットにQFutureWatcher::finishedシグナルを接続するだけです。

最後のステップは、watcher.setFuture(future)で未来のオブジェクトを見るようにwatcherオブジェクトに指示します。この文は、ほとんどSF映画から出てきたように見えます。

Qt ConcurrentではMapReduceとFilterReduceの実装も提供されています。MapReduceは基本的に2つのことを行うプログラミングモデルです。

* データセットの処理をCPUの複数のコアにマップしたり、分散したりします。
* 発信者に提供するために結果を縮小または集約します。

この技術は、CPUのクラスタ内で巨大なデータセットを処理できるようにするために、Googleが最初に推進したものです。

簡単なマップ操作の例を紹介します。

```C++
QList images = ...;

QImage processGrayscale(QImage& image);
QFuture<void> future = QtConcurrent::mapped(
    images, processGrayscale);
```

QtConcurrent::run()の代わりに、リストを取るマッピング関数と、毎回違うスレッドで各要素に適用する関数を使っています。imagesリストはその場で修正されるので、テンプレート型でQFutureを宣言する必要はありません。

QtConcurrent::mapped()の代わりにQtConcurrent::blockingMapped()を使用することで、操作をブロック化することができます。

最後に、MapReduceの操作は次のようになります。

```C++
QList images = ...;

QImage processGrayscale(QImage& image);
void combineImage(QImage& finalImage, const QImage& inputImage);

QFuture<void> future = QtConcurrent::mappedReduced(
    images,
    processGrayscale,
    combineImage);
```

ここでは，マップ関数 processGrayscale() が返す各結果に対して呼び出される combineImage() 関数を追加しました．この関数は、中間データである inputImage を finalImage にマージします。この関数はスレッドごとに一度に一度だけ呼び出されるので、結果変数をロックするためにミューテックスを使用する必要はありません。

FilterReduceは全く同じパターンに従います。filter関数は、入力リストを変換する代わりにフィルタリングすることができます。

***

**[戻る](../index.html)**
