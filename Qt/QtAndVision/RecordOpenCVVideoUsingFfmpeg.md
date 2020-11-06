# ffmpeg を使用して OpenCV の映像を保存する

環境: Linux Mint 20 + Qt 5.15.1

**[全ソースはここ](https://github.com/Taro3/RecordOpenCVVideoUsingFfmpeg)**

ffmpeg を経由して、 OpenCV の動画を保存してみます。もちろん Qt を使用します。^^;

方法としては、 ffmpeg を QProcess で起動して、 stdin 経由で動画を流して保存します。( ffmpeg を経由するので、ビットレートなどの設定も可能になります)

まず、 ffmpeg に渡す引数を定義します。

```C++
    QStringList args = {"-y", "-f", "image2pipe", "-vcodec", "mjpeg", "-r", "24", "-i", "-", "-vcodec", "h264"
                        , "-qscale", "5", "-r", "24", "video.mp4",};
```

ここでは、モーション jpeg を受け取って、 h264 で出力するように指定しています。

次に、この引数を使用して ffmpeg を QProcess で起動します。

```C++
    p.setProgram("ffmpeg");
    p.setArguments(args);
    p.start();
    p.waitForStarted();
```

ループ中では、 OpenCV で取得したフレームデータを、一旦 JPEG に変換してから ffmpeg のプロセスの stdin に渡します。

```C++
        cv::Mat outFrame;
        cv::cvtColor(frame, outFrame, cv::COLOR_BGR2RGB);
        QImage image(outFrame.data, outFrame.cols, outFrame.rows, outFrame.step, QImage::Format_RGB888);
        // write jpeg via sidin
        image.save(&p, "jpg");
```

これを繰り返すだけです。

終了するときに、QProcess もクローズします。

一旦 JPEG へ変換しないで直接 Mat データを送る場合は、 #if defined で無効にしてある方の処理を使います。

ffmpeg で rawvideo を受け取って h264 を出力するようにすれば OK です。

※私の環境では、Qt 5.15.1 では、デバッグビルドをして、ステップ実行を行うと、 QProcess の開始後に例外が発生しました。 Qt 6 beta3 では例外は発生しませんでした。謎です…^^;

***

**[戻る](../Qt.md)**
