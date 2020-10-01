# コードのベンチマーク

Qt Test は、コードの実行速度をベンチマークするための非常に使いやすいセマンティックも提供しています。実際にそれを見るために、JSON形式でトラックを保存するのにかかる時間をベンチマークしてみましょう。トラックの長さ（SoundEventsの数）にもよりますが、シリアル化にかかる時間は多かれ少なかれかかるはずです。

もちろん、この機能をトラックの長さを変えてベンチマークし、時間の節約が直線的であるかどうかを確認する方が面白いです。データセットの登場!期待される入力と出力で同じ関数を実行するだけでなく、異なるパラメータで同じ関数を実行することも有用です。

まずはTestJsonSerializerでデータセット関数を作成します。

```C++
class TestJsonSerializer : public QObject
{
    ...

private slots:
    void cleanup();
    void saveDummy();
    void loadDummy();
    void saveTrack_data();
    ...
};

void TestJsonSerializer::saveTrack_data()
{
    QTest::addColumn<int>("soundEventCount");
    QTest::newRow("1") << 1;
    QTest::newRow("100") << 100;
    QTest::newRow("1000") << 1000;
}
```

saveTrack_data() 関数は、単純に Track クラスに追加する SoundEvent の数を保存する前に保存します。1"、"100"、"1000 "の文字列は、テスト実行出力に明確なラベルをつけるためにここにあります。これらの文字列は、saveTrack() の各実行で表示されます。これらの数値は自由に微調整してください

さて、ベンチマークコールを使った実際のテストです。

```C++
class TestJsonSerializer : public QObject
{
    ...
    void saveTrack_data();
    void saveTrack();
    ...
};

void TestJsonSerializer::saveTrack()
{
    QFETCH(int, soundEventCount);
    Track track;
    track.record();
    for (int i = 0; i < soundEventCount; ++i) {
        track.addSoundEvent(i % 4);
    }
    track.stop();

    QBENCHMARK {
        mSerializer.save(track, FILENAME);
    }
}
```

saveTrack() 関数は、データセットから soundEventCount カラムを取得することから始まります。その後、正しい数のsoundEventを（適切なrecord()状態で！）追加し、最後にJSON形式でのシリアライズをベンチマークします。

ベンチマーク自体が単純にこんな感じのマクロになっていることがわかります。

```C++
QBENCHMARK {
    // instructions to benchmark
}
```

QBENCHMARKマクロで囲んだ命令は自動的に計測されます。更新されたTestJsonSerializerクラスでテストを実行すると、このような出力が表示されるはずです。

```shell
PASS : TestJsonSerializer::saveTrack(1)
RESULT : TestJsonSerializer::saveTrack():"1":
    0.041 msecs per iteration (total: 84, iterations: 2048)
PASS : TestJsonSerializer::saveTrack(100)
RESULT : TestJsonSerializer::saveTrack():"100":
    0.23 msecs per iteration (total: 59, iterations: 256)
PASS : TestJsonSerializer::saveTrack(1000)
RESULT : TestJsonSerializer::saveTrack():"1000":
    2.0 msecs per iteration (total: 66, iterations: 32)
```

ご覧の通り、QBENCHMARKマクロを使うとQt Testの出力が非常に面白いデータになります。1つのSoundEventでTrackクラスを保存するのに0.041ミリ秒かかりました。Qt Testはこのテストを2048回繰り返し、合計84ミリ秒かかりました。

QBENCHMARKマクロの威力は、次のテストから見えてきます。ここでは、saveTrack()関数が100個のSoundEventsを持つTrackクラスを保存しようとしました。これには0.23ミリ秒かかり、命令を256回繰り返しました。これは、Qt Testベンチマークが、1回の反復にかかる平均時間に基づいて反復回数を自動的に調整していることを示しています。

QBENCHMARKマクロは、メトリックが複数回繰り返されると（外部ノイズの可能性を避けるために）より正確になる傾向があるため、このような動作をします。

***

## Tips

複数回の反復を行わずにテストをベンチマークしたい場合は、QBENCHMARK_ONCEを使用します。

***

コマンドラインを使ってテストを実行すると、QBENCHMARKに追加のメトリクスを提供することができます。以下に、利用可能なオプションをまとめた表を示します。

| 名前 | コマンドライン引数 | 可用性 |
|---|---|---|
| Walltime | (デフォルト) | すべてのプラットフォーム |
| CPUティックカウンター | -tickcounter | Windows, OS X, Linux, 多くのUNIXライクなシステム |
| イベントカウンター | -eventcounter | すべてのプラットフォーム |
| Valgrind Callgrind | -callgrind | Linux (インストールされていれば) |
| Linux Perf | -perf | Linux |

これらのオプションのそれぞれは、ベンチマークされたコードの実行時間を測定するために使用される選択されたバックエンドを置き換えます。例えば、-tickcounter 引数を指定して drum-machine-test を実行した場合。

```shell
    $ ./drum-machine-test -tickcounter
    ...
    RESULT : TestJsonSerializer::saveTrack():"1":
    88,062 CPU cycles per iteration (total: 88,062, iterations: 1)
    PASS : TestJsonSerializer::saveTrack(100)
    RESULT : TestJsonSerializer::saveTrack():"100":
    868,706 CPU cycles per iteration (total: 868,706, iterations: 1)
    PASS : TestJsonSerializer::saveTrack(1000)
    RESULT : TestJsonSerializer::saveTrack():"1000":
    7,839,871 CPU cycles per iteration (total: 7,839,871, iterations:
1)
    ...
```

ミリ秒単位で測定された壁時間が、各イテレーションで完了したCPUサイクル数で置き換えられていることがわかります。

これは、対応するターゲットに送信される前にイベントループで受信した数を測定します。これは、あなたのコードが適切な数のシグナルを発しているかどうかをチェックするための興味深い方法かもしれません。

***

**[戻る](../index.html)**
