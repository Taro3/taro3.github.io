# Mac用のプロジェクトを設定する

プロジェクトをMac OS上で動作させるための最初のステップは，OpenCVをインストールすることです．幸いなことに、これは brew コマンドを使えば非常に簡単にできます。もしあなたが Mac OS で開発をしていて、まだ使っていないのであれば、今すぐダウンロードしてください。一言で言えば、brew は代替パッケージマネージャであり、Mac App Store にはない多くのパッケージ（開発者向け、非開発者向け）にアクセスできるようにしてくれます。

***

## Info

brewをダウンロードしてインストールするには、http://brew.sh/。

***

ターミナルでは、以下のコマンドを入力するだけです。

```shell
    brew install opencv
```

これで、OpenCVをダウンロードしてコンパイルし、マシンにインストールすることができます。この記事を書いている時点で、brewで利用可能なOpenCVの最新バージョンは2.4.13でした。これが終わったら、filter-plugin-designer.proを開き、以下のブロックを追加します。

```QMake
macx {
target.path = "$$(QTDIR)/../../QtCreator.app/Contents/PlugIns/designer/"
target_lib.files = $$OUT_PWD/lib$${TARGET}.dylib
target_lib.path =
"$$(QTDIR)/../../QtCreator.app/Contents/PlugIns/designer/"
    INSTALLS += target_lib
    INCLUDEPATH += /usr/local/Cellar/opencv/2.4.13/include/
    LIBS += -L/usr/local/lib \
    -lopencv_core \
    -lopencv_imgproc
}
```

OpenCVヘッダを追加し、パスをINCLUDEPATHとLIBS変数でリンクしています。target定義とinstallsは、出力された共有オブジェクトをQt Creatorアプリケーションのプラグインディレクトリに自動的にデプロイするために使用されます。

最後にしなければならないことは、最終的なアプリケーションにリンクするライブラリがどこにあるかをQt Creatorに知らせるための環境変数を追加することです。プロジェクトタブで、以下の手順を実行します。

1. **Build Environment**で**Details**ウィンドウを開きます。
2. **追加**ボタンをクリックします。
3. \<VARIABLE\> フィールドに DYLD_LIBRARY_PATH と入力します。
4. \<VALUE\> にビルドディレクトリのパスを入力します (**一般**→**ビルドディレクトリ**のセクションからコピーして貼り付けることができます)。

***

**[戻る](../index.html)**
