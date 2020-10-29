# OpenCV に関するメモ

## Ubuntu 20.04 に OpenCV 4.5 をインストールする

### インストール

* 依存するものをインストールする

```shell
sudo apt update && sudo apt install -y cmake g++ wget unzip
sudo apt install -y libavcodec-dev libavformat-dev libswscale-dev libv4l-dev
sudo apt install -y libxvidcore-dev libx264-devunzip opencv.zip
sudo apt install -y libgtk-3-dev
sudo apt install -y ninja-build
```

* OpenCVの最新版をダウンロードする

```shell
wget -O opencv.zip https://github.com/opencv/opencv/archive/master.zip
wget -O opencv_contrib.zip https://github.com/opencv/opencv_contrib/archive/master.zip
unzip opencv_contrib.zip
```

* ビルドしてインストールする

```shell
mv opencv-master opencv
mkdir -p build && cd build
cmake -GNinja ../opencv
ninja
sudo ninja install ※
```

※ sudo ninja install は、 /usr/local/ 以下にヘッダやライブラリをインストールするため、複数のバージョンが必要な場合は実行しないほうがいい！

***

## 動作テスト

* 画像の表示テスト

DisplayImage.cpp というファイルを作って試す( OpenCV のサイトにあるソースそのまま)

```c++
#include <stdio.h>
#include <opencv2/opencv.hpp>
using namespace cv;
int main(int argc, char** argv )
{
    if ( argc != 2 )
    {
        printf("usage: DisplayImage.out <Image_Path>\n");
        return -1;
    }
    Mat image;
    image = imread( argv[1], 1 );
    if ( !image.data )
    {
        printf("No image data \n");
        return -1;
    }
    namedWindow("Display Image", WINDOW_AUTOSIZE );
    imshow("Display Image", image);
    waitKey(0);
    return 0;
}
```

CMakeLists.txt を作る

```txt
cmake_minimum_required(VERSION 2.8)
project( DisplayImage )
find_package( OpenCV REQUIRED )
include_directories( ${OpenCV_INCLUDE_DIRS} )
add_executable( DisplayImage DisplayImage.cpp )
target_link_libraries( DisplayImage ${OpenCV_LIBS} )
```

ビルドして実行する

```shell
cmake .
make

./DisplayImage XXXXX.jpg
```

これで画像が表示されればOK。

* 動画テスト

※Ubuntu 内で動画を再生するには  
sudo apt install ubuntu-restricted-extras  
で、ubuntu-restricted-extras がインストールされていること

DisplayImage.cpp を下記のように変更

```c++
#include <stdio.h>
#include <opencv2/opencv.hpp>
using namespace cv;
int main(int argc, char** argv )
{
    auto cap = VideoCapture("XXXXX.mp4");   // 動画名を適当に入れる
    if (!cap.isOpened())
        return -1;
    Mat frame;
    while (true) {
        auto ret = cap.read(frame);
        if (!ret)
            break;
        imshow("img", frame);
        if (waitKey(30) == 27)
            break;
    }
    destroyAllWindows();
    return 0;
}
```

CMakeLists.txt は上記と同じ

ビルドして実行する。

```shell
cmake .
make

./DisplayImage
```

で動画が表示されることを確認する。

とりあえず、以上で画像と動画の動作確認ができる。

***

## Qt での OpenCV アプリ作成( Ubuntu )

Qt で OpenCV アプリを作成する際には、下記のように .pro ファイルに追記する必要がある。

```QMake
INCLUDEPATH += /usr/local/include/opencv4
LIBS += -L/usr/local/lib
LIBS += -lopencv_core -lopencv_highgui -lopencv_imgcodecs -lopencv_videoio
```

リンクエラーが出た場合は、適宜ライブラリを追加すること。( libopencv_world を作成していない場合)
libopencv_world を作成する場合は、-D BUILD_opencv_world=on を付けてビルドする。

**[戻る](../index.md)**
