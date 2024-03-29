# 顔で楽しむ

第3章「ホームセキュリティアプリケーション」では、Gazerという新しいアプリケーションを作成し、パソコンに取り付けたウェブカメラからビデオを撮影したり、動きを検出したりすることができました。この章では、ウェブカメラを使った遊びを続け、動きを検出する代わりに、カメラを使って顔を検出する新しいアプリケーションを作成します。まず、Webカメラに写っている顔を検出します。次に、検出された顔から顔のランドマークを検出します。この顔のランドマークにより、検出された顔の目、鼻、口、頬の位置がわかるので、顔に面白いマスクを適用することができます。

この章では、以下のトピックを取り上げます。

* ウェブカメラからの写真撮影
* OpenCVを使った顔の検出
* OpenCV を用いた顔のランドマークの検出
* Qt ライブラリのリソースシステム
* 顔へのマスクの適用

***

## 技術的要件

前の章で見たように、ユーザは少なくとも Qt バージョン 5 をインストールし、C++ と Qt プログラミングの基本的な知識を持っていることが要求されます。また、OpenCVの最新版である4.0が正しくインストールされている必要があります。さらに、この章では、OpenCV の core モジュールと imgproc モジュールの他に、video モジュールと videoio モジュールも利用されます。もしあなたがこれまでの章を読んできたのであれば、これらの要件はすでに満たされていることでしょう。

顔や顔のランドマークを検出するために、OpenCVが提供するいくつかの事前学習済みの機械学習モデルを使用しますので、機械学習技術に関する基礎知識があればより良いでしょう。これらの機械学習モデルのいくつかは、OpenCVライブラリのエクストラモジュールのものなので、OpenCVのエクストラモジュールもコアモジュールと一緒にインストールされている必要があります。この章では、OpenCVの追加モジュールを段階的にインストールしてから使用するので、不安な方はご安心ください。

この章のすべてのコードは、私たちのコードリポジトリ（[https://github.com/PacktPublishing/Qt-5-and-OpenCV-4-Computer-Vision-Projects/tree/master/Chapter-04](https://github.com/PacktPublishing/Qt-5-and-OpenCV-4-Computer-Vision-Projects/tree/master/Chapter-04)）で見ることができます。

次のビデオでコードの動作を確認してください： [http://bit.ly/2FfQOmr](http://bit.ly/2FfQOmr)

***

## アプリケーション「Facetious」

この章で作成するアプリケーションは、検出された顔にリアルタイムで面白いマスクを適用することで、多くの楽しみを与えてくれますので、私はこのアプリケーションをFacetiousと名付けました。Facetiousアプリケーションで最初にできることは、ウェブカメラを開き、そこからビデオフィードを再生することです。これは、前章で私たちが作ったGazerアプリケーションで行っていた作業です。そこで、ここでは、Gazerアプリケーションの骨格を借りて、新しいアプリケーションのベースとします。計画としては、まず、Gazerのコピーを作り、Facetiousに名前を変え、動き検出に関する機能を削除し、ビデオ撮影機能を写真撮影の新機能に変更します。そうすることで、シンプルでクリーンなアプリケーションに、顔や顔の特徴を検出する新しい機能を追加することができます。

***

### GazerからFacetiousへ

まず、Gazerアプリケーションのソースをコピーすることから始めましょう。

```sh
     $ mkdir Chapter-04
     $ cp -r Chapter-03/Gazer Chapter-04/Facetious
     $ ls Chapter-04
     Facetious
     $ cd Chapter-04/Facetious
     $ make clean
     $ rm -f Gazer
     $ rm -f Makefile
```

これらのコマンドで、Chapter-03 ディレクトリの下にある Gazer ディレクトリを Chapter-04/Facetious にコピーします。そして、そのディレクトリに入り、make clean を実行して、コンパイル時に生成された中間ファイルをすべてクリーニングし、rm -f Gazer で古いターゲットの実行ファイルを削除しています。

では、プロジェクトファイルごとにリネームとクリーニングを行ってみましょう。

まず、Gazer.pro のプロジェクトファイルです。 Facetious.pro にリネームして、エディタで開き、内容を編集します。エディタで、TARGET キーの値を Gazer から Facetious に変更し、QT キーの値から、この新しいアプリケーションでは使わない Qt モジュール、network と concurrent を削除し、ファイルの最後にある GAZER_USE_QT_CAMERA の関連行を削除しています。Facetious.proの変更された行は以下の通りです。

```qmake
     TARGET = Facetious
     # ...
     QT += core gui multimedia
     # ...
     # the below lines are deleted in this update:
     # Using OpenCV or QCamera
     # DEFINES += GAZER_USE_QT_CAMERA=1
     # QT += multimediawidgets
```

次にmain.cppファイルです。 このファイルは、ウィンドウのタイトルをGazerからFacetiousに変えるだけなので、シンプルです。

```cpp
     window.setWindowTitle("Facetious");
```

次に、capture_thread.h ファイルです。このファイルでは、CaptureThreadクラスから多くのフィールドとメソッドを削除しています。削除されるフィールドは以下の通りです。

```cpp
         // FPS calculating
         bool fps_calculating;
         int fps;

         // video saving
         // int frame_width, frame_height; // notice: we keep this line
         VideoSavingStatus video_saving_status;
         QString saved_video_name;
         cv::VideoWriter *video_writer;

         // motion analysis
         bool motion_detecting_status;
         bool motion_detected;
         cv::Ptr<cv::BackgroundSubtractorMOG2> segmentor;
```

このクラスで削除されるメソッドは以下の通りです。

```cpp
         void startCalcFPS() {...};
         void setVideoSavingStatus(VideoSavingStatus status) {...};
         void setMotionDetectingStatus(bool status) {...};
     // ...
     signals:
         // ...
         void fpsChanged(int fps);
         void videoSaved(QString name);

     private:
         void calculateFPS(cv::VideoCapture &cap);
         void startSavingVideo(cv::Mat &firstFrame);
         void stopSavingVideo();
         void motionDetect(cv::Mat &frame);
```

enum VideoSavingStatus型も不要になったので、同様に削除します。

さて、capture_thread.hはきれいになりましたので、次にcapture_thread.cppに移りましょう。ヘッダファイルの変更点に従って、まず、以下のことを行いましょう。

* コンストラクタの中で，ヘッダファイルで削除したフィールドの初期化を削除する。
* ヘッダファイルで削除したメソッド（スロットを含む）の実装を削除する。
* runメソッドの実装において、動画保存、モーション検出、FPS（Frames per Second）計算に関するコードをすべて削除。

これで、キャプチャスレッドから、ビデオ保存、モーション検出、FPS計算に関するコードがすべて削除されました。この章では、ビデオキャプチャはOpenCVで行うので、まず、#ifdef GAZER_USE_QT_CAMERA から #endif 行の間のコードを削除してください。このようなコードのブロックが2つあるので、その両方を削除します。次に、多くのメソッドとフィールドを削除します。そのほとんどは、ビデオの保存、モーション検出、FPSの計算に関するものです。これらのフィールドとメソッドは次のとおりです。

```cpp
         void calculateFPS();
         void updateFPS(int);
         void recordingStartStop();
         void appendSavedVideo(QString name);
         void updateMonitorStatus(int status);

     private:
         // ...
         QAction *calcFPSAction;
         // ...
         QCheckBox *monitorCheckBox;
         QPushButton *recordButton;
```

appendSavedVideoメソッドとQPushButton *recordButtonフィールドは、実際には削除されないことに注意してください。それぞれ、appendSavedPhoto、QPushButton *shutterButtonに名前を変えています。

```cpp
         void appendSavedPhoto(QString name);
         // ...
         QPushButton *shutterButton;
```

これは、新しいアプリケーションで写真を撮るための準備です。これまで述べてきたように、Facetiousでは、ビデオは録画せず、写真だけを撮ります。

次に、mainwindow.cppのヘッダーファイルと同じように、#ifdef GAZER_USE_QT_CAMERA と #else の間のコードを削除してください。また、この種のブロックが2つありますので、それぞれのブロックの#endif行を削除することを忘れないでください。その後、削除された5つのメソッドの実装を削除します。

* void calculateFPS();
* void updateFPS(int);
* void recordingStartStop();
* void appendSavedVideo(QString name);
* void updateMonitorStatus(int status);

ほとんどの削除は終わりましたが、MainWindowクラスはまだやることがたくさんあります。まずは、ユーザーインターフェースから。MainWindow::initUIメソッドで、モニター状態のチェックボックス、記録ボタン、記録ボタンの横のプレースホルダーに関するコードを削除し、新たにシャッターボタンを作成します。

```cpp
         shutterButton = new QPushButton(this);
         shutterButton->setText("Take a Photo");
         tools_layout->addWidget(shutterButton, 0, 0, Qt::AlignHCenter);
```

先ほどのコードで、シャッターボタンを tools_layout の唯一の子ウィジェットにし、ボタンが中央に配置されていることを確認します。

そして、ステータスバーを作成した後、ステータスバーの起動メッセージを「Facetious is Ready」に変更します。

```cpp
         mainStatusLabel->setText("Facetious is Ready");
```

次に、MainWindow::createActionsメソッドです。このメソッドでは、calcFPSActionアクションの作成とシグナルスロットの接続に関するコードを削除します。

次に、MainWindow::openCameraメソッドで、FPSの計算と動画の保存に関するコードをすべて削除します（そのほとんどはシグナルスロットの接続と切断です）。このメソッドの最後にあるチェックボックスとプッシュボタンに関するコードも削除します。

最後に、新しく追加された appendSavedPhoto メソッドに空の実装を与え、populateSavedList メソッドのボディを空にすることです。次のサブセクションでは、これらのメソッドに新しい実装を追加します。

```cpp
     void MainWindow::populateSavedList()
     {
         // TODO
     }

     void MainWindow::appendSavedPhoto(QString name)
     {
         // TODO
     }
```

次は、utilities.h と utilities.cpp ファイルの番です。ヘッダーファイルでは、notifyMobileメソッドを削除し、newSavedVideoNameメソッドとgetSavedVideoPathメソッドをそれぞれnewPhotoNameとgetPhotoPathにリネームしています。

```cpp
     public:
         static QString getDataPath();
         static QString newPhotoName();
         static QString getPhotoPath(QString name, QString postfix);
```

utilities.cppでは、ヘッダーファイルの変更に合わせて名前の変更と削除を行うほか、getDataPathメソッドの実装を変更しています。

```cpp
     QString Utilities::getDataPath()
     {
         QString user_pictures_path = QStandardPaths::standardLocations(QStandardPaths::PicturesLocation)[0];
         QDir pictures_dir(user_pictures_path);
         pictures_dir.mkpath("Facetious");
         return pictures_dir.absoluteFilePath("Facetious");
     }
```

最も重要な変更点は、QStandardPaths::MoviesLocation の代わりに QStandardPaths::PicturesLocation を使用して、ビデオ用ではなく写真用の標準ディレクトリを取得するようになったことです。

さて、Gazer アプリケーションを簡略化することで、私たちの新しい Facetious アプリケーションの基礎を得ることに成功しました。それでは、コンパイルと実行を試してみましょう。

```sh
     $ qmake -makefile
     $ make
     g++ -c #...
     # output truncated
     $ export LD_LIBRARY_PATH=/home/kdr2/programs/opencv/lib
     $ ./Facetious
```

うまくいけば、Gazer のメイン・ウィンドウに非常によく似た空白のウィンドウが表示されます。このサブセクションのすべてのコードの変更は、[https://github.com/PacktPublishing/Qt-5-and-OpenCV-4-Computer-Vision-Projects/commit/0587c55e4e8e175b70f8046ddd4d67039b431b54](https://github.com/PacktPublishing/Qt-5-and-OpenCV-4-Computer-Vision-Projects/commit/0587c55e4e8e175b70f8046ddd4d67039b431b54) の単一の Git コミットで見つけることができます。もし、このサブセクションを完了するのに問題があれば、遠慮なくそのコミットを参照してください。

***

### 写真を撮る

前節では、Gazerから動画保存機能と動体検知機能を削除し、新しいアプリケーション「Facetious」の基盤を作りました。また、写真撮影のためのいくつかのスタブを新しいアプリケーションに配置しました。このサブセクションでは、写真撮影の機能を完成させます。

ビデオ撮影機能に比べ、写真撮影は非常にシンプルです。まず、capture_thread.h ヘッダーファイルで、CaptureThread クラスにフィールドと多くのメソッドを追加しています。

```cpp
     public:
         // ...
         void takePhoto() {taking_photo = true; }
         // ...
     signals:
         // ...
         void photoTaken(QString name);

     private:
         void takePhoto(cv::Mat &frame);

     private:
         // ...
         // take photos
         bool taking_photo;
```

bool taking_photo フィールドは、キャプチャスレッドが現在のフレームを写真としてハードディスクに保存すべきかどうかを示すフラグであり、void takePhoto() public inline メソッドはこのフラグを true に設定するためのメソッドである。このメソッドは、ユーザがメインウィンドウのシャッターボタンをクリックしたときにメインスレッドで呼び出すことにします。写真を撮るたびに void photoTaken(QString name) シグナルが発行され、Qt メタオブジェクトシステムがその実装を担当します。void takePhoto(cv::Mat &frame) private メソッドは，フレームを写真としてディスクに保存するためのメソッドで， capture_thread.cpp のソースファイルで実装する必要があるのはこのメソッドのみです． その実装を見てみましょう．

```cpp
     void CaptureThread::takePhoto(cv::Mat &frame)
     {
         QString photo_name = Utilities::newPhotoName();
         QString photo_path = Utilities::getPhotoPath(photo_name, "jpg");
         cv::imwrite(photo_path.toStdString(), frame);
         emit photoTaken(photo_name);
         taking_photo = false;
     }
```

メソッド本体では、Utilitiesクラスで書いた関数を使って新しい名前を生成し、生成した名前で保存する写真のパスを取得し、拡張子名をjpgとしています。そして、OpenCVのimgcodecsモジュールのimwrite関数を使って、フレームをJPEG画像ファイルとして、指定したパスのディスクに書き込んでいます。写真が保存された後、写真名とともに photoTaken シグナルを出力します。このシグナルに興味を持つ人がいれば、スロットを接続する必要があり、シグナルが発せられた瞬間にそのスロットが呼ばれます。メソッド本体の最後では、taking_photoフラグをfalseに戻しています。

CaptureThread::takePhoto(cv::Mat &frame) メソッドが実装されたので、 CaptureThread::run() メソッド内のキャプチャ無限ループでこれを呼び出してみましょう。

```cpp
             if(taking_photo) {
                 takePhoto(tmp_frame);
             }
```

このコード片では、写真を撮るべきかどうかを調べるために taking_photo フラグをチェックしています。もしこれが真であれば、 takePhoto(cv::Mat &frame) メソッドを呼び出して、現在のフレームを写真として保存します。このコード片は、 tmp_frame が空でないチェックを通過した後、そのフレームの色順序が BGR から RGB に変換される前に配置されなければならず、これにより、imwrite 関数に渡すことができる適切な色順序のフレームであることが確認されます。

CaptureThreadクラスで最後に行うことは、そのコンストラクタでtaking_photoフラグをfalseに初期化することです。

さて、ユーザーインターフェイスに移りましょう。まず、mainwindow.h で MainWindow クラスに新しいプライベートスロットを追加します。

```cpp
     private slots:
         // ...
         void takePhoto();
```

このスロットは、シャッターボタンのシグナルに接続されることになります。その実装をmainwindow.cppのソースファイルから見てみましょう。

```cpp
     void MainWindow::takePhoto()
     {
         if(capturer != nullptr) {
             capturer->takePhoto();
         }
     }
```

単純な作業です。このメソッドでは、CaptureThreadのインスタンスであるcapturerのvoid takePhoto()メソッドを呼び出して、nullでなければ写真を撮るように指示しています。そして、MainWindow::initUI()メソッドで、シャッターボタンを作成した後、このスロットをshutterButtonのclickedシグナルに接続します。

```cpp
         connect(shutterButton, SIGNAL(clicked(bool)), this, SLOT(takePhoto()));
```

これまでの作業で、キャプチャースレッドに写真を撮るように指示することができるようになりました。しかし、写真が撮影されたとき、メインウィンドウはどうやってそれを知るのでしょうか？CaptureThread::photoTaken シグナルと MainWindow::appendSavedPhoto スロットの間の接続によって実現されます。 この接続は、MainWindow::openCamera()メソッドでキャプチャスレッドのインスタンスが生成された後に行います。

```cpp
         connect(capturer, &CaptureThread::photoTaken, this, &MainWindow::appendSavedPhoto);
```

また、同じメソッドで閉じたキャプチャースレッドインスタンスの前に切断することを忘れないでください。

```cpp
             disconnect(capturer, &CaptureThread::photoTaken, this, &MainWindow::appendSavedPhoto);
```

さて、MainWindow::appendSavedPhoto(QString name) スロットが何をするのかを見てみましょう。前のサブセクションで、私たちはこのスロットに空のボディを与えました。さて、このスロットはその責任を負わなければなりません。

```cpp
     void MainWindow::appendSavedPhoto(QString name)
     {
         QString photo_path = Utilities::getPhotoPath(name, "jpg");
         QStandardItem *item = new QStandardItem();
         list_model->appendRow(item);
         QModelIndex index = list_model->indexFromItem(item);
         list_model->setData(index, QPixmap(photo_path).scaledToHeight(145), Qt::DecorationRole);
         list_model->setData(index, name, Qt::DisplayRole);
         saved_list->scrollTo(index);
     }
```

これは、第3章「ホームセキュリティの応用」でGazerアプリケーションの保存ビデオリストに新しく録画したビデオのカバー画像を追加したときに行ったことと非常によく似ています。したがって、ここではこのコードの一部を一行ずつ説明することはしません。

もうひとつ、アプリケーションの起動中に、撮影したすべての写真を保存済み写真リストに登録するためのメソッド、MainWindow::populateSavedList()を実装する必要があります。このメソッドも、Gazerアプリケーションで保存された動画をポピュレートするために使用したメソッドと非常によく似ているので、実装はおまかせします。もし困ったことがあれば、GitHubにある本書の付属コードリポジトリを参照してください。このサブセクションのすべての変更は、このコミットにあります: [https://github.com/PacktPublishing/Qt-5-and-OpenCV-4-Computer-Vision-Projects/commit/744d445ad4c834cd52660a85a224e48279ac2cf4](https://github.com/PacktPublishing/Qt-5-and-OpenCV-4-Computer-Vision-Projects/commit/744d445ad4c834cd52660a85a224e48279ac2cf4)

さて、もう一度アプリケーションをコンパイルして実行してみましょう。アプリケーションがメインウィンドウを表示した後、FileメニューのOpen Cameraアクションをクリックしてカメラを開き、シャッターボタンをクリックして写真を撮影します。これらのアクションの後、私のメインウィンドウは次のスクリーンショットのようになります。

![実行結果](img/641e3c74-b3f5-43f2-8c2e-699b73dc4957.png)

この章では、新しいアプリケーションの基本的な機能を設定します。次章では、OpenCVを用いて、撮影したフレームからリアルタイムに顔を検出します。

***

## カスケード分類器を用いた顔検出

前節では、カメラの映像を再生したり写真を撮ったりできる新しいアプリケーション、Facetiousを作成しました。本章では、このアプリケーションに新しい機能を追加し、OpenCVライブラリを使ってリアルタイムにビデオ内の顔を検出する方法を紹介します。

顔検出には、OpenCVが提供するカスケード分類器と呼ばれる機能を使用します。カスケード分類器は、顔の検出だけでなく、物体の検出にも使われます。分類器としては、画像中のある注目領域がある種の物体であるか否かを教えてくれます。この分類器は、いくつかの単純な分類器（段階）を含んでおり、その後、これらの単純な分類器を注目領域に適用します。そして、いずれかの簡易分類器が否定的な結果を出した場合、その注目領域には注目すべきオブジェクトが含まれないと判断します。一方、すべての段階が成功すれば、その領域から対象物を発見したと言う。これがカスケード（cascade）の意味です。

カスケード分類器を使用する前に、まず学習させる必要があります。学習過程では、ある種類のオブジェクトの多くのサンプルビュー（正の例と呼ばれる）と、その種類のオブジェクトを含まない多くの画像（負の例）を分類器に与える。例えば、カスケード分類器を人間の顔の検出に役立てたい場合、人間の顔を含む画像と人間の顔を含まない画像を多数用意して学習させる必要があります。カスケード分類器は、これらの正例と負例を用いて、画像中のある領域がある種の物体を含んでいるかどうかを学習します。

この学習プロセスは複雑ですが、幸いなことにOpenCVでは、事前に学習された多くのカスケード分類器がリリースされています。後ほど例に挙げる Haar 分類器を考えてみましょう。インストールされた OpenCV ライブラリのデータディレクトリを調べると、その中にたくさんの事前学習された分類器データがあることが分かります。

```sh
    # if you use a system provided OpenCV, the path is /usr/share/opencv/haarcascades
    $ ls /home/kdr2/programs/opencv/share/opencv4/haarcascades/
    haarcascade_eye_tree_eyeglasses.xml haarcascade_lefteye_2splits.xml
    haarcascade_eye.xml haarcascade_licence_plate_rus_16stages.xml
    haarcascade_frontalcatface_extended.xml haarcascade_lowerbody.xml
    haarcascade_frontalcatface.xml haarcascade_profileface.xml
    haarcascade_frontalface_alt2.xml haarcascade_righteye_2splits.xml
    haarcascade_frontalface_alt_tree.xml haarcascade_russian_plate_number.xml
    haarcascade_frontalface_alt.xml haarcascade_smile.xml
    haarcascade_frontalface_default.xml haarcascade_upperbody.xml
    haarcascade_fullbody.xml
```

学習済みデータファイルは、ファイル名で簡単に見分けることができます。frontalfaceを含むファイル名は、顔検出に必要なものです。

物体検出、特に顔検出に関する知識を得たところで、ビデオフィードから顔を検出するためのアプリケーションに戻りましょう。

まず、Facetious.pro のプロジェクトファイルを更新します。

```qmake
    # ...
    unix: !mac {
        INCLUDEPATH += /home/kdr2/programs/opencv/include/opencv4
        LIBS += -L/home/kdr2/programs/opencv/lib -lopencv_core -lopencv_imgproc -lopencv_imgcodecs -lopencv_video -lopencv_videoio -lopencv_objdetect
    }

    # ...
    DEFINES += OPENCV_DATA_DIR=\\\"/home/kdr2/programs/opencv/share/opencv4/\\\"
    #...
```

LIBS キーの設定では、これから使用する顔検出を含む物体検出の機能は、この OpenCV コアモジュールによって提供されるため、その値に opencv_objdetect OpenCV モジュールを追加しています。変更点の 2 つ目は、OpenCV のインストール先であるデータディレクトリを定義するマクロ定義です。このマクロを使用して、事前学習された分類器データをコードに読み込みます。

次に、capture_thread.h ヘッダファイルに移動します。このファイルでは、CaptureThread クラスに private メソッドと private メンバフィールドを追加しています。

```cpp
    #include "opencv2/objdetect.hpp"
    //...

    private:
        // ...
        void detectFaces(cv::Mat &frame);

    private:
        // ...

        // face detection
        cv::CascadeClassifier *classifier;
```

このメンバフィールドが顔検出に使うカスケード分類器であり、顔検出の作業は新しく追加されたdetectFacesメソッドで行われることは明らかです。

では、capture_thread.cppのソースファイルを見て、そこでカスケード分類器をどのように使うか見てみましょう。まず、CaptureThread::runメソッドの本体を更新します。

```cpp
        classifier = new cv::CascadeClassifier(OPENCV_DATA_DIR "haarcascades/haarcascade_frontalface_default.xml");

        while(running) {
            cap >> tmp_frame;
            if (tmp_frame.empty()) {
                break;
            }

            detectFaces(tmp_frame);
            // ...
        }
        cap.release();
        delete classifier;
        classifier = nullptr;
```

このメソッドを入力して Web カメラを開いた後、cv::CascadeClassifier のインスタンスを生成し、classifier メンバフィールドに代入しています。インスタンスを生成する際に、コンストラクタに事前に学習された分類器のデータパスを渡します。このパスは、プロジェクトファイルで定義されている OPENCV_DATA_DIR マクロを用いて構築されます。また、顔検出用の分類器を作成するために、OpenCV のデータディレクトリの下にある haarcascades/haarcascade_frontalface_default.xml ファイルを使用します。

runメソッド内の無限ループでは、新たに追加したdetectFacesメソッドを、開いたWebカメラからキャプチャしたばかりのフレームで呼び出しています。

無限ループ終了後、キャプチャスレッドが終了する前に、クリーニングを行い、開いたカメラを解放し、分類器を削除し、nullに設定します。

最後に、detectFacesメソッドの実装を見てみましょう。

```cpp
    void CaptureThread::detectFaces(cv::Mat &frame)
    {
        vector<cv::Rect> faces;
        cv::Mat gray_frame;
        cv::cvtColor(frame, gray_frame, cv::COLOR_BGR2GRAY);
        classifier->detectMultiScale(gray_frame, faces, 1.3, 5);

        cv::Scalar color = cv::Scalar(0, 0, 255); // red
        for(size_t i = 0; i < faces.size(); i++) {
            cv::rectangle(frame, faces[i], color, 1);
        }
    }
```

このメソッドの本体では、まず検出された顔の外接矩形を保存するための cv::Rect ベクトルを宣言します。次に、顔検出処理はRGBの色の特徴とは無関係であるため、入力フレームをグレースケール画像に変換します。そして、分類器のdetectMultiScaleメソッドを呼び出します。このメソッドの最初の引数は、顔を検出したいグレースケールの入力フレームです。第2引数は矩形ベクトルへの参照で、先ほど定義した検出された顔の外接矩形を保存するために使用します、第3引数は、画像スケールごとに画像サイズをどれだけ縮小するかを指定するためのもので、一方の顔がカメラに近いという理由だけで他方の顔より大きく見える場合に生じる大きさの誤認識を補償するために作られる。この検出アルゴリズムでは、移動ウィンドウを使用してオブジェクトを検出します。4番目の引数は、顔が検出されたと宣言する前に、現在のオブジェクトの近くでいくつのオブジェクトが検出されたかを定義するためのものです。

detectMultiScale メソッドが返った後、検出された顔の領域をすべて faces vector に取得し、キャプチャしたフレームにこれらの長方形を 1 ピクセルの赤い縁取りで描画します。

さて、顔検出のための機能はこれで完了です。では、プロジェクトをコンパイルして実行してみましょう。開いたウェブカメラの目線の先に誰かが入ってくると、その人の顔の周りに赤い矩形が表示されます。

![実行結果](img/03184846-d6cf-47d8-807f-3f1e10720b22.png)

今回のコードでは、カスケード分類器を作成する際に、haarcascade_frontalface_default.xml ファイルを使用しました。しかし、OpenCVのデータディレクトリを見ると、haarcascade_frontalface_alt.xml, haarcascade_frontalface_alt2.xml, haarcascade_frontalface_alt_tree.xml など、前顔検出のためのファイルであることがわかるファイルが複数あることに気づかれるかもしれません。なぜhaarcascade_frontalface.xmlを選び、他を選ばないのでしょうか？この事前学習済みモデルデータは、異なるデータセット、または異なる設定で学習されます。詳細はこれらのXMLファイルの冒頭にコメントとして記述されていますので、必要であればそちらのドキュメントを参照してください。モデルファイルを選択するもう一つの簡単な方法は、すべてのモデルファイルを試し、あなたのデータセットでテストし、精度と再現率を計算し、あなたのケースに最適なものを選択することです。

Haarカスケード分類器以外にも、Local binary pattern (LBP) カスケード分類器という名前のカスケード分類器があり、これはOpenCVのリリースにデフォルトで同梱されています。この分類器の学習済みモデルデータは、OpenCV のインストールディレクトリのデータパスにある lbpcascades ディレクトリに格納されています。LBP カスケード分類器は、Haar よりも高速ですが、精度も低くなっています。以下の表で、両者を比較することができます。

![表](img/6f842a64-f44d-4238-9b7c-0693c8b9cb92.png)

これらのアルゴリズムと学習済みモデルデータを自由に試して、ご自分のケースに合った最適なものを見つけてください。

***

## 顔のランドマークの検出

前節では、OpenCVが提供するカスケード分類器を用いて顔を検出することで、画像中のどの領域が顔であるかを知ることができました。しかし、矩形領域だけでは、顔の目、眉、鼻がどこにあるのか、といった顔の詳細まではわかりません。顔認識技術では、このような詳細な情報を顔のランドマークと呼んでいます。本章では、このような顔のランドマークを検出する方法を探します。

残念ながら、OpenCVのコアモジュールは顔のランドマークを検出するアルゴリズムを提供していないので、OpenCVの追加モジュールであるfaceモジュールに頼ることになります。

faceモジュールを使用する前に、このモジュールがコンピュータにインストールされていることを確認する必要があります。第2章「プロ並みの画像編集」の「OpenCVをソースからビルドしてインストールする」では、OpenCV v4.0.0を追加モジュールのないソースからビルドしてインストールしました。ここで、追加モジュールを含めて再構築し、再インストールしてみましょう。

前回ビルドした際に、OpenCVのソースをダウンロードして解凍し、あるディレクトリに配置しました。では、OpenCVの追加モジュールのソースを、そのリリースページ（[https://github.com/opencv/opencv_contrib/releases](https://github.com/opencv/opencv_contrib/releases)）からダウンロードしてみましょう。ダウンロードして使用したコアモジュールのバージョンはv4.0.0なので、同じバージョンのエクストラモジュールを [https://github.com/opencv/opencv_contrib/archive/4.0.0.zip](https://github.com/opencv/opencv_contrib/archive/4.0.0.zip) からダウンロードします。ダウンロードしたソースを解凍し、解凍したコアモジュールのソースと同じディレクトリに置き、ターミナルでビルドします。

```sh
    $ ls ~
    opencv-4.0.0 opencv_contrib-4.0.0 # ... other files
    $ cd ~/opencv-4.0.0 # path to the unzipped source of core modules
    $ mkdir release # create the separate dir
    $ cd release
    $ cmake -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib-4.0.0/modules \
      -D CMAKE_BUILD_TYPE=RELEASE \
      -D CMAKE_INSTALL_PREFIX=$HOME/programs/opencv ..
    # ... output of cmake ...
    # rm ../CMakeCache.txt if it tells you are not in a separate dir
    $ make
    # ... output of make ...
    $ make install
```

このように、前回のインストールとは異なり、cmake コマンドに -D OPENCV_EXTRA_MODULES_PATH=../../opencv_contrib-4.0.0/modules オプションを追加し、追加モジュールのソースが存在する場所を指定しています。これらのコマンドが終了した後、faceモジュールが正しくインストールされているかどうかを確認します。

```sh
    $ ls ~/programs/opencv/include/opencv4/opencv2/face
    bif.hpp facemark.hpp facerec.hpp
    face_alignment.hpp facemarkLBF.hpp mace.hpp
    facemarkAAM.hpp facemark_train.hpp predict_collector.hpp
    $ ls ~/programs/opencv/lib/libopencv_face*
    /home/kdr2/programs/opencv/lib/libopencv_face.so
    /home/kdr2/programs/opencv/lib/libopencv_face.so.4.0
    /home/kdr2/programs/opencv/lib/libopencv_face.so.4.0.0
    $
```

先のシェルコマンドのように、ヘッダファイルや共有オブジェクトがOpenCVのインストールパスにあれば、OpenCVの追加モジュールが正常にインストールされたことになります。

Faceモジュールがインストールされたら、Webブラウザで[https://docs.opencv.org/4.0.0/d4/d48/namespacecv_1_1face.html](https://docs.opencv.org/4.0.0/d4/d48/namespacecv_1_1face.html)、そのドキュメントを開き、どのような機能を提供しているか見てみましょう。

FacemarkKazemi、FacemarkAAM、FacemarkLBFクラスは、顔のランドマークを検出するために利用されるアルゴリズムです。これらのアルゴリズムはすべて機械学習ベースのアプローチであるため、データセット操作やモデル学習のための設備も充実しています。機械学習モデルの学習は本章の範囲外ですので、本節では、やはり事前に学習させたモデルを使用することにします。

今回のアプリケーションでは、FacemarkLBFクラスで実装されているアルゴリズムを使用します。事前学習済みモデルのデータファイルは、[https://raw.githubusercontent.com/kurnianggoro/GSOC2017/master/data/lbfmodel.yaml](https://raw.githubusercontent.com/kurnianggoro/GSOC2017/master/data/lbfmodel.yaml) からダウンロードすることができます。ダウンロードして、プロジェクトのルートディレクトリにあるdataというサブディレクトリに配置しましょう。

```sh
    $ mkdir -p data
    $ cd data/
    $ pwd
    /home/kdr2/Work/Books/Qt-5-and-OpenCV-4-Computer-Vision-Projects/Chapter-04/Facetious/data
    $ curl -O https://raw.githubusercontent.com/kurnianggoro/GSOC2017/master/data/lbfmodel.yaml
      % Total % Received % Xferd Average Speed Time Time Time Current
                                     Dload Upload Total Spent Left Speed
      0 53.7M 0 53.7k 0 0 15893 0 0:59:07 0:59:07 0:00:00 0
    $ ls
    lbfmodel.yaml
```

これで準備作業はすべて終了です。では、顔のランドマーク検出機能の開発を完了させるために、プロジェクトのコードソースに戻りましょう。capture_thread.hに、CaptureThreadクラスのinclude指示文とprivateメンバ・フィールドを新たに追加します。

```cpp
    // ...
    #include "opencv2/face/facemark.hpp"
    // ...
    class CaptureThread : public QThread
    {
        // ...
    private:
        // ...
        cv::Ptr<cv::face::Facemark> mark_detector;
    };
```

cv::Ptr\<cv::face::Facemark\> 型のメンバフィールド mark_detector は、顔のランドマークを検出するために利用される検出器です。この検出器は、ソースファイルcapture_thread.cppのCaptureThread::runメソッドでインスタンス化されます。

```cpp
        classifier = new cv::CascadeClassifier(OPENCV_DATA_DIR "haarcascades/haarcascade_frontalface_default.xml");
        mark_detector = cv::face::createFacemarkLBF();
        QString model_data = QApplication::instance()->applicationDirPath() + "/data/lbfmodel.yaml";
        mark_detector->loadModel(model_data.toStdString());
```

以下のコードにあるように、runメソッドでは、顔検出に用いる分類器を作成した後、cv::face::createFacemarkLBF() を呼び出して FacemarkLBF のインスタンスを作成し、mark_detector メンバフィールドに代入しています。次に、ダウンロードしたプリトレーニングモデルのデータファイルのパスを文字列で作成します。最後に、モデルデータファイルのパスを指定して loadModel メソッドを呼び出し、mark_detector にデータをロードします。この時点で、検出器は使用可能な状態になりました。では、CaptureThread::detectFacesメソッドでどのように使うか見てみましょう。

```cpp
        vector< vector<cv::Point2f> > shapes;
        if (mark_detector->fit(frame, faces, shapes)) {
            // draw facial land marks
            for (unsigned long i=0; i<faces.size(); i++) {
                for(unsigned long k=0; k<shapes[i].size(); k++) {
                    cv::circle(frame, shapes[i][k], 2, color, cv::FILLED);
                }
            }
        }
```

CaptureThread::detectFaces メソッドの最後で、要素の型が cv::Point2f の vector である変数を宣言しています。1つの顔のランドマークはvector\<cv::Point2f\>型で表現される点の列で、1つのフレームに複数の顔が検出されることがあるので、このように複雑なデータ型を使って表現する必要があるのです。次に、顔のランドマークを検出するために、mark_detector メンバフィールドの fit メソッドを呼び出すことが肝心です。このメソッドには、入力フレーム、カスケード分類器で検出した顔の矩形、そして出力ランドマークを保存するための変数が渡されます。fitメソッドが0以外の値を返した場合、ランドマークの取得に成功したことになります。顔のランドマークが得られたら、検出された顔を繰り返し処理し、各顔のランドマークの点を繰り返し処理し、各点に2ピクセルの円を描いてランドマークを示すようにします。

*前述したように、macOSを使用している場合、コンパイルされたアプリケーションは実際にはFacetious.appというディレクトリであり、QApplication::instance()->applicationDirPath()の表現値はFacetious.app/Contents/MacOSとなります。そのため、macOSでは、lbfmodel.yamlのモデルデータをFacetious.app/Contents/MacOS/dataディレクトリの下に配置する必要があります。*

プロジェクトファイルを除いて、ほとんどすべてが完了しました。LIBS の設定に使用した追加の face モジュールをそのファイルに追加してみましょう。

```qmake
    unix: !mac {
        INCLUDEPATH += /home/kdr2/programs/opencv/include/opencv4
        LIBS += -L/home/kdr2/programs/opencv/lib -lopencv_core -lopencv_imgproc -lopencv_imgcodecs -lopencv_video -lopencv_videoio -lopencv_objdetect -lopencv_face
    }
```

さて、いよいよアプリケーションをコンパイルして実行します。以下は、アプリケーションの実行中にこれらのランドマークがどのように見えるかを示しています。

![実行結果](img/71d8484b-bcb6-45b3-9004-4340a7f5ba9a.png)

このように、目、眉、鼻、口、あごに多くのランドマークが得られます。しかし、どの点がどの顔の特徴を表しているのかは、まだわかりません。各顔のランドマークの点の順番が決まっていると考えると、ある点がある顔の特徴を表しているかどうかは、その点のインデックスを利用して判断することができます。そこで、2ピクセルの円ではなく、各点のインデックス番号を描き、その分布を見ることにする。

```cpp
    // cv::circle(frame, shapes[i][k], 2, color, cv::FILLED);
    QString index = QString("%1").arg(k);
    cv::putText(frame, index.toStdString(), shapes[i][k], cv::FONT_HERSHEY_SIMPLEX, 0.4, color, 2);
```

そして、人間の顔のランドマークを検出して描画すると、次のようになります。

![実行結果](img/23f2644c-ad09-4510-b402-1ae72d87d07f.png)

このように、インデックス番号で表される点は、顔上の位置が固定されているため、以下のような点のインデックスによって、これらの顔の特徴にアクセスすることができる。

* 口は点[48, 68]
* 右眉は点[17, 22]
* 左眉は点[22, 27]
* 右目は点[36, 42]
* 左目は点[42, 48]
* 鼻は点[27, 35]
* 顎は点[0, 17]

次章では、この位置情報を用いて、検出された顔にマスクを適用する。

***

## 顔へのマスクの適用

本章の前のセクションでは、ビデオフィードから顔と顔のランドマークを検出することに成功しました。このセクションでは、より興味深いことを行います。ここには3つのマスクまたは装飾があります。検出された顔に対して、リアルタイムでこれらを適用してみましょう。

![マスク](img/f26b39b7-4ea2-4de9-a5fb-73e71842d758.png)

これらの装飾品は、ディスク上の画像です。第1章「画像ビューアの構築」で見た画像、第2章「プロ並みの画像編集」で編集した画像、第3章「ホームセキュリティアプリケーション」で録画した動画など、ユーザーからのデータとは異なり、コードのソースファイルと同じように、アプリケーションの一部となります。特に、アプリケーションをユーザーに配布する際には、何らかの方法でアプリケーションと結び付けなければなりません。これらのリソースをコンパイル済みのバイナリに同梱して、ユーザーに解凍させ、これらのリソースを特定のパスに配置することも可能です。しかし、これでは、特にユーザがアプリケーションを異なるプラットフォームで実行する場合に、ユーザにとって困難が生じる可能性があります。幸い、Qtライブラリはこのような状況に対処するためのリソースシステムを提供しています。そのリソースシステムは、プラットフォームに依存しない仕組みになっています。使用したリソースファイルをアプリケーションの実行ファイルに格納することができるのです。もし、あなたのアプリケーションが特定の静的ファイル（例えば、アイコン、画像、翻訳ファイル、カスケードスタイルシートなど）を使用していて、アプリケーションの出荷に問題があったり、ファイルを紛失するリスクを負いたくないのであれば、Qtリソースシステムはあなたのためにあるのです。

それでは、このリソースシステムを使って、オーナメントの画像をどのように管理し、読み込むか見てみましょう。

***

### Qt リソースシステムによる画像の読み込み

Qt リソースシステムでは、アプリケーションで使用するリソースファイルは、アプリケーションのソースツリーの一部でなければならないことが要求されます。そこで、まず、私たちが言及する各オーナメントをJPEG画像として保存し、プロジェクトのルートディレクトリのサブディレクトリにimagesという名前で配置します。


```sh
   $ pwd
     /home/kdr2/Work/Books/Qt-5-and-OpenCV-4-Computer-Vision-Projects/Chapter-04/Facetious
     $ ls images
     glasses.jpg mouse-nose.jpg mustache.jpg
```

*これらの装飾を、ビデオフィードのフレームに重ね合わせます。この重ね合わせを簡単にするため、これらの画像は、描画色：黒、背景色：白、形状：正方形の3チャンネルJPEG画像として保存されます。なぜ、このようにすることでオーナメントアプリケーションを簡単にすることができるのかは、後ほど説明します。*

画像の準備ができたら、それを記述するためのQtリソース・コレクション・ファイルを作成します。Qtリソースファイルは、XMLベースのファイル形式で、拡張子はQtリソースコレクションの略称である.qrcです。リソースファイルの名前を images.qrc とし、プロジェクトのルートディレクトリに配置します。その内容を見てみましょう。

```xml
<!DOCTYPE RCC>
<RCC version="1.0">
  <qresource>
    <file>images/glasses.jpg</file>
    <file>images/mustache.jpg</file>
    <file>images/mouse-nose.jpg</file>
  </qresource>
</RCC>
```

とても簡単です。Qt リソースコレクションファイルのファイルノードとして、リソースイメージのすべてのパスをリストアップしています。指定されたパスは、.qrc ファイルを含むディレクトリからの相対パスで、ここではプロジェクトのルートディレクトリになります。リストアップされたリソースファイルは、.qrcファイルと同じディレクトリ、またはそのサブディレクトリのいずれかに配置されている必要があることに注意してください。

そして、このリソース集ファイルを Facetious.pro のプロジェクトファイルに追加して、qmake に処理を指示します。

```qmake
RESOURCES = images.qrc
```

これで、qmake -makefile と make でプロジェクトをコンパイルすると、rcc を起動するコマンドが Makefile に配置され、実行されます。その結果、qrc_images.cppという名前のC++ソースファイルが生成されます。そのファイルは、Qtリソースコンパイラであるrccによって生成されます。また、.qrcファイルに記載されている画像はすべて、この生成されたC++ソースファイルにバイト配列として埋め込まれ、プロジェクトをコンパイルする際に、アプリケーションの実行ファイルにコンパイルされることになります。

さて、オーナメント画像をアプリケーションの実行ファイルにうまく埋め込むことができました。でも、どうやってアクセスするのでしょうか？簡単です。これらのリソースは、ソースツリーにあるのと同じファイル名の下に :/ というプレフィックスを付けて、あるいは qrc スキームを持つ URL を使って、コードの中でアクセスすることができます。例えば、:/images/glasses.jpg というファイルパスや qrc:///images/glasses.jpg という URL は glasses.jpg ファイルにアクセスするためのもので、アプリケーションのソースツリー内の位置は images/glasses.jpg になります。

Qt のリソースシステムに関するこの知識を利用して、アプリケーションに装飾品を cv::Mat のインスタンスとしてロードしてみましょう。まず、capture_thread.h ヘッダファイルの変更です。

```cpp
     // ...
     private:
         // ...
         void loadOrnaments();
     // ...
     private:
         // ...
         // mask ornaments
         cv::Mat glasses;
         cv::Mat mustache;
         cv::Mat mouse_nose;
         // ...
```

ご覧のように、読み込んだ装飾品を保持するための cv::Mat 型の 3 つのプライベートメンバフィールドと、それらを読み込むためのプライベートメソッドを追加しています。新しく追加されたメソッドの実装は、ソースファイル capture_thread.cpp にあります。

```cpp
     void CaptureThread::loadOrnaments()
     {
         QImage image;
         image.load(":/images/glasses.jpg");
         image = image.convertToFormat(QImage::Format_RGB888);
         glasses = cv::Mat(
             image.height(), image.width(), CV_8UC3,
             image.bits(), image.bytesPerLine()).clone();

         image.load(":/images/mustache.jpg");
         image = image.convertToFormat(QImage::Format_RGB888);
         mustache = cv::Mat(
             image.height(), image.width(), CV_8UC3,
             image.bits(), image.bytesPerLine()).clone();

         image.load(":/images/mouse-nose.jpg");
         image = image.convertToFormat(QImage::Format_RGB888);
         mouse_nose = cv::Mat(
             image.height(), image.width(), CV_8UC3,
             image.bits(), image.bytesPerLine()).clone();
     }
```

このメソッドでは、まずQImageを型とするオブジェクトを定義し、そのloadメソッドを呼び出して画像を読み込んでいます。loadメソッドを呼び出す際、引数として :/images/glasses.jpg という文字列を使用します。 これは :/ で始まる文字列で、前述のようにQtのリソースシステムからリソースをロードするための方法です。ここでは、qrc:///images/glasses.jpg という文字列を使ってもかまいません。

画像が読み込まれた後、それを 3 チャンネル 8 深度 のフォーマットに変換し、データ型として CV_8UC3 を持つ cv::Mat インスタンスを作成します。変換後は、以前のプロジェクトで行ったように、 QImage から cv::Mat オブジェクトを作成します。ここで注目すべきは、 clone メソッドを呼び出して、構築されたばかりの Mat インスタンスのディープコピーを作成し、それをクラスのメンバフィールドに代入している点です。このため、構築されたばかりの Mat オブジェクトは、その下のデータバッファを QImage オブジェクトと共有しています。その QImage を再読み込みするとき、あるいはメソッドが返されてその QImage が破壊されるとき、そのデータバッファは削除されます。

同じように、次に口ひげとネズミの鼻の装飾をロードします。そして、CaptureThreadクラスのコンストラクターで、この新しく追加されたloadOrnamentsメソッドを呼び出します。

この時点で、Qtリソースシステムのおかげで、3つのオーナメント画像をすべてアプリケーションの実行ファイルにコンパイルして、コードに便利にロードすることができます。実は、Qtリソースシステムは、この章で使っているもの以外にもいろいろなことができるのです。例えば、アプリケーションが動作している環境に応じて、使用するリソースファイルを選択することができます。また、アプリケーションの実行ファイルに埋め込むのではなく、すべてのリソースファイルを1つのrccバイナリファイルにコンパイルし、QResource APIを使って登録・読み込みを行うこともできます。これらの詳細については、Qtリソースシステムの公式ドキュメント（[https://doc.qt.io/qt-5/resources.html](https://doc.qt.io/qt-5/resources.html)）を参照することができます。

***

### 顔へのマスクの描画

前のサブセクションで、用意した装飾品を cv::Mat のインスタンスとしてアプリケーションにロードしました。では、このサブセクションでカメラから検出された顔に、それらを描画してみましょう。

それでは、メガネの飾りを見てみましょう。メガネを顔につける前に、すべきことはたくさんあります。まず、オーナメント画像のメガネの幅は固定ですが、映像から検出した顔の幅は自由なので、幅に合わせてメガネのサイズを変更する必要があります。次に、メガネは水平に配置されていますが、映像中の顔は傾いていたり、90度回転していることもあるので、顔の傾きに合わせてメガネを回転させる必要があります。最後に、メガネを描くのに適切な位置を探します。

これらのことをコードでどのように行うか見てみましょう。capture_thread.h のヘッダーファイルで、クラス内のメソッド宣言を新たに追加する。

```cpp
     private:
         // ...
         void drawGlasses(cv::Mat &frame, vector<cv::Point2f> &marks);
```

このメソッドの第一引数はメガネを描きたいフレームで、第二引数はこのフレームで検出したある顔のランドマークである。そして、その実装をcapture_thread.cppのソースファイルに与えています。

```cpp
     void CaptureThread::drawGlasses(cv::Mat &frame, vector<cv::Point2f> &marks)
     {
         // resize
         cv::Mat ornament;
         double distance = cv::norm(marks[45] - marks[36]) * 1.5;
         cv::resize(glasses, ornament, cv::Size(0, 0), distance / glasses.cols, distance / glasses.cols, cv::INTER_NEAREST);

         // rotate
         double angle = -atan((marks[45].y - marks[36].y) / (marks[45].x - marks[36].x));
         cv::Point2f center = cv::Point(ornament.cols/2, ornament.rows/2);
         cv::Mat rotateMatrix = cv::getRotationMatrix2D(center, angle * 180 / 3.14, 1.0);

         cv::Mat rotated;
         cv::warpAffine(
             ornament, rotated, rotateMatrix, ornament.size(),
             cv::INTER_LINEAR, cv::BORDER_CONSTANT, cv::Scalar(255, 255, 255));

         // paint
         center = cv::Point((marks[45].x + marks[36].x) / 2, (marks[45].y + marks[36].y) / 2);
         cv::Rect rec(center.x - rotated.cols / 2, center.y - rotated.rows / 2, rotated.cols, rotated.rows);
         frame(rec) &= rotated;
     }
```

前のコードで見たように、このメソッドの最初の部分は、メガネのサイズを変更することです。cv::norm 関数を呼び出して、点 mark[45] と mark[36] の間の距離を計算します。この2つの点は何であり、なぜそれを選んだのでしょうか？この章の顔のランドマークの検出のセクションで、顔の69個のランドマークすべての位置を示すために写真を使ったのを覚えていますか？その写真では、marks[45]が左目の一番外側、marks[36]が右目の一番外側であることがわかりました。通常、メガネの幅はこの2点間の距離より少し大きいので、その距離に1.5を掛けてメガネの適正幅とし、メガネのオーナメント画像を適正サイズにリサイズしています。なお、画像サイズを変更した場合、横幅と縦幅は同じ比率で拡大縮小されます。

第二のパートは、オーナメントを回転させることである。選んだ2点の垂直距離を水平距離で割って、その結果をatan関数に渡して、顔の傾きの角度を計算します。計算結果の角度はラジアン単位で表現されているので、OpenCV を用いて回転させる際には度単位に変換する必要があることに注意してください。次に、第2章「プロ並みの画像編集」で、RotatePluginを使った画像の回転のところで使ったcv::warpAffine関数を使って、オーナメントを回転させるのです。画像を回転させる場合、入力画像のサイズを使うのではなく、回転させた画像の適切なサイズを計算しないと、画像が切り取られてしまうことがあります。しかし、入力画像が正方形で、その中のオブジェクトの最大幅と最大高さがともに正方形の辺の長さより小さい場合、入力画像のサイズを出力サイズとして使うことで、切り取られることなく安全に回転させることができるんです。前のサブセクションでオーナメント画像を正方形として用意したのはこのためです。このため、ここでの回転は実にシンプルで簡単です。

最後に、サイズを変更して回転させたメガネを顔の上に描きます。選択した2点の中心点をメガネ画像の中心点として、メガネ画像を配置する矩形を計算します。そして、frame(rec) &= rotated; というステートメントで飾りを描きます。このステートメントには少し説明が必要かもしれません。frame 変数は cv::Mat 型であり， frame(rec) は，その Mat cv::Mat::operator()(const Rect &roi) const operator メソッドを呼び出します。このメソッドは、引数 Rect によって決定される新しい cv::Mat 型を返し、元の cv::Mat 型とデータバッファを共有します。したがって、この行列に対する操作は、実際には元の行列の対応する領域に対して適用されます。この装飾画像は、背景色として白、前景色として黒を持っているので、単純にビット単位の AND 演算を用いて描画することができます。

これでメソッドの実装は完了です。顔のランドマークを取得した後に検出した各顔に描画するために、detectFaces でこのメソッドを呼び出してみましょう。

```cpp
             // ...
             for (unsigned long i=0; i<faces.size(); i++) {
                 for(unsigned long k=0; k<shapes[i].size(); k++) {
                     // cv::circle(frame, shapes[i][k], 2, color, cv::FILLED);
                 }
                 drawGlasses(frame, shapes[i]);
             }
             // ...
```

また、動画をわかりやすくするために、顔のランドマークを描画するためのステートメントをコメントアウトしています。では、アプリケーションをコンパイルして実行し、どのように動作するかを見てみましょう。

![実行結果](img/0b034f30-f1b4-4a28-92a3-8e807fa207bf.png)

はっ、悪くない！？次に進みましょう。 まだ2つの装飾品を描かなければなりませんが、それは眼鏡を描いたのとほぼ同じ方法で行えます。

口ひげは、左の口角を表す mark[54]と右の口角を表す mark[48]を使って、幅と傾きを計算します。中心点は、marks[33]とmarks[51]を使って決めます。
マウスの鼻は、marks[13]とmarks[3]で幅を、marks[16]とmarks[0]で傾きを、そしてmarks[30]と鼻の先端を中心点として計算しています。
以上の内容で、drawMustacheとdrawMouseNoseのメソッドを自分で実装することができますし、弊社のコードリポジトリにあるコードを参照することもできます。以下の写真は、私たちのアプリケーションがこれらの装飾をどのように行っているかを示しています。

![実行結果](img/f46dd526-e860-4d62-a22c-4ae6a5825904.png)

うちの子、かわいいですよね。

***

### UI上でのマスクの選択

本章の前節では、Webカメラからの映像からリアルタイムに顔や顔のランドマークを検出する作業を多く行ってきました。その際、顔の周囲に矩形を描き、顔のランドマークを2ピクセルの円として描き、3種類のマスクを顔に適用しました。この方法は、OpenCVを使った顔認識の勉強になりますが、ユーザーからすると、顔に描く量が多すぎます。そこで、ここでは、検出された顔に描くマークやマスクを、ユーザーインターフェースのチェックボックスで選択できるようにします。そのために、メインウィンドウのシャッターボタンの下に5つのチェックボックスを追加します。

ユーザー・インターフェースを変更する前に、CaptureThreadクラスにどのマークやマスクを最初に描画するかを選択する機能を持たせる必要があります。そこで、capture_thread.hファイルを開いて、このクラスに何かを追加してみましょう。

```cpp
     public:
         // ...
         enum MASK_TYPE{
                        RECTANGLE = 0,
                        LANDMARKS,
                        GLASSES,
                        MUSTACHE,
                        MOUSE_NOSE,
                        MASK_COUNT,
         };

         void updateMasksFlag(MASK_TYPE type, bool on_or_off) {
             uint8_t bit = 1 << type;
             if(on_or_off) {
                 masks_flag |= bit;
             } else {
                 masks_flag &= ~bit;
             }
         };

     // ...
     private:
         // ...
         bool isMaskOn(MASK_TYPE type) {return (masks_flag & (1 << type)) != 0; };
     private:
         // ...
         uint8_t masks_flag;
         // ...
```

まず、マークやマスクの種類を示す列挙型 MASK_TYPE を追加する。

* RECTANGLEの値は0であり、顔の周囲を囲む矩形である。
* LANDMARKSは値1で顔のランドマークを表す。
* GLASSESは、値2でメガネの飾り（またはマスク）です。
* MUSTACHE は口ひげの飾りで、値は 3 です。
* MOUSE_NOSE はネズミの鼻の飾りで、値は 4 です。
* MASK_COUNT は、どのマークやマスクにも対応するものではなく、値5（前の値に1を足した値）を持ち、便宜上、マークやマスクの総数を数えるために使用されます。

次に、これらのマークやマスクがオンになっているかオフになっているかの状態を保存するために、uint8_t masks_flag プライベートフィールドを新たに追加する。このフィールドは、マークやマスクがオンになっていれば対応するビットが1になり、そうでなければビットが0になるというビットマップとして使用されます。これは、新しく追加した updateMasksFlag public inlineメソッドで行います。 また、あるマークやマスクがオンになっているかどうかをテストするために、isMaskOnというプライベートインラインメソッドも用意しました。では、これらの機能をcapture_thread.cppのソースファイル内で使ってみましょう。

まず、CaptureThread::runメソッドの中で、detectFacesメソッドを呼び出すところで、条件チェックを追加しています。

```cpp
             // detectFaces(tmp_frame);
             if(masks_flag > 0)
                 detectFaces(tmp_frame);
```

masks_flag ビットマップがゼロであれば、マークやマスクがオンになっていないことがわかるので、detectFaces メソッドを呼び出す必要はない。しかし、そのビットマップがゼロより大きい場合、顔を検出するためにそのメソッドを呼び出して入力する必要があります。ここで、どのように条件チェックを行うか見てみましょう。

```cpp
    // ...
         if (isMaskOn(RECTANGLE)) {
             for(size_t i = 0; i < faces.size(); i++) {
                 cv::rectangle(frame, faces[i], color, 1);
             }
         }
     // ...
 
                 if (isMaskOn(LANDMARKS)) {
                     for(unsigned long k=0; k<shapes[i].size(); k++) {
                         cv::circle(frame, shapes[i][k], 2, color, cv::FILLED);
                     }
                 }
     // ...
                 if (isMaskOn(GLASSES))
                     drawGlasses(frame, shapes[i]);
                 if (isMaskOn(MUSTACHE))
                     drawMustache(frame, shapes[i]);
                 if (isMaskOn(MOUSE_NOSE))
                     drawMouseNose(frame, shapes[i]);
     // ...
```

それらはシンプルでわかりやすい。isMaskOnメソッドで各マスクタイプのフラグをチェックし、そのタイプのマークやマスクを描画するかどうかを判断しているのです。

CaptureThreadクラスの最後の注意点は、コンストラクタでmasks_flagを0に初期化することを忘れないことです。CaptureThreadクラスに関するこれらのことが終わったら、ユーザー・インターフェースを変更するために、メイン・ウィンドウに移りましょう。

mainwindow.hのヘッダーファイルで、新しいプライベートスロットとチェックボックスの配列を追加しています。

```cpp
     private slots:
         // ...
         void updateMasks(int status);

     // ...
     private:
         // ...
         QCheckBox *mask_checkboxes[CaptureThread::MASK_COUNT];
```

先の配列にはCaptureThread::MASK_COUNTの要素があり、先程のように5個です。updateMasksスロットは、これらのチェックボックスのためのものです。これらのチェックボックスをmainwindow.cppのソースファイルのinitUIメソッドで作成、配置してみましょう。

```cpp
         // masks
         QGridLayout *masks_layout = new QGridLayout();
         main_layout->addLayout(masks_layout, 13, 0, 1, 1);
         masks_layout->addWidget(new QLabel("Select Masks:", this));
         for (int i = 0; i < CaptureThread::MASK_COUNT; i++){
             mask_checkboxes[i] = new QCheckBox(this);
             masks_layout->addWidget(mask_checkboxes[i], 0, i + 1);
             connect(mask_checkboxes[i], SIGNAL(stateChanged(int)), this, SLOT(updateMasks(int)));
         }
         mask_checkboxes[0]->setText("Rectangle");
         mask_checkboxes[1]->setText("Landmarks");
         mask_checkboxes[2]->setText("Glasses");
         mask_checkboxes[3]->setText("Mustache");
         mask_checkboxes[4]->setText("Mouse Nose");
```

initUIメソッドでは、チェックボックス用の新しいグリッドレイアウトを作成するために、シャッターボタンを作成する次の行に前のコードを追加しています。そのグリッドレイアウトは、メインレイアウトの1行、14列目を占有しています。そして、新しく作成したグリッドレイアウトに新しいラベルを追加し、そのテキストをSelect Masks: に設定して、この領域の機能を紹介します。その後、forループで、マスクの種類ごとにチェックボックスを作成してグリッドレイアウトに追加し、そのstateChanged（int）シグナルを新しく追加したupdateMasks（int）スロットに接続します。forループの後、先ほど作成した各チェックボックスに適切なテキストを設定します。

新しく追加した領域がメインレイアウトの14行目を占めることを考えると、同じメソッドで保存された写真用のリストの領域を1行下に移動させる必要があります。

```cpp
         // main_layout->addWidget(saved_list, 13, 0, 4, 1);
         main_layout->addWidget(saved_list, 14, 0, 4, 1);
```

以下はupdateMasksスロットの実装ですが、これから見ていきましょう。

```cpp
    void MainWindow::updateMasks(int status)
     {
         if(capturer == nullptr) {
             return;
         }

         QCheckBox *box = qobject_cast<QCheckBox*>(sender());
         for (int i = 0; i < CaptureThread::MASK_COUNT; i++){
             if (mask_checkboxes[i] == box) {
                 capturer->updateMasksFlag(static_cast<CaptureThread::MASK_TYPE>(i), status != 0);
             }
         }
     }
```

このスロットでは、まず、キャプチャしているスレッドがNULLであるかどうかをチェックします。もしNULLでなければ、シグナルの送り手、つまりQtライブラリからsender関数を呼び出して、このスロットが呼ばれるようにどのチェックボックスがクリックされたかを見つけます。そして、チェックボックス配列 mask_checkboxes の中から sender のインデックスを見つけます。先ほど見つけたインデックスは、正しくMASK_TYPE列挙型の中の対応するマスクの型の値なので、次に、キャプチャするスレッドインスタンスのupdateMasksFlagメソッドを呼び出し、送り手のチェックボックスの状態に従って、マスクをオンまたはオフにします。

最後に、ユーザインタフェースの変更についてですが、MainWindow::openCameraメソッドでキャプチャスレッドの新しいインスタンスを生成して起動した後、すべてのチェックボックスのチェックをはずすように設定します。

```cpp
         for (int i = 0; i < CaptureThread::MASK_COUNT; i++){
             mask_checkboxes[i]->setCheckState(Qt::Unchecked);
         }
```

さて、ついに私たちの新しいアプリケーション、Facetious のすべての作業が完了しました! それでは、コンパイルと実行を行い、どのように見えるか見てみましょう。

![実行結果](img/efb1c2db-8891-499e-a6f2-1060d41b8c5f.png)

シャッターボタンの下にあるチェックボックスを利用して、表示するマークやマスクを選択できるようになりました。

***

## まとめ

本章では、前章で作成した Gazer アプリケーションを簡略化して、Facetious という新しいアプリケーションを作成しました。要約の過程で、ビデオ保存とモーション検出の機能が削除され、新たに写真撮影の機能が追加されました。その結果、顔認証に集中できるすっきりとしたアプリケーションを得ることができました。そして、このアプリケーションでは、事前に学習したカスケード分類器を用いて顔を検出する機能と、別の事前学習した機械学習モデルを用いて顔のランドマークを検出する機能を開発しました。最後に、面白いことに、検出されたランドマークを用いて、検出された顔に様々な装飾を施すことができました。

次章では、光学式文字認識（OCR）技術について説明し、この技術を使って画像やスキャンした文書からテキストを抽出します。

***

## 問題集

この章の知識を試すために、以下の問題に挑戦してください。

1. LBPカスケード分類器を用いて、自分で顔を検出してみてください。
2. OpenCVライブラリには、顔のランドマークを検出するために使用できる他のアルゴリズムがいくつかあり、それらのほとんどは[https://docs.opencv.org/4.0.0/db/dd8/classcv_1_1face_1_1Facemark.html](https://docs.opencv.org/4.0.0/db/dd8/classcv_1_1face_1_1Facemark.html)。ぜひ試してみてください。
3. 顔に色のついた装飾を施すには？

***

**[次へ](../5/OpticalCharacterRecognition.md)**
