# iOS用パッケージ

iOS用のQtアプリケーションのパッケージ化はXCodeに依存しています。Qt Creatorからgallery-mobileをビルドして実行すると、フードの下でXCodeが呼び出されます。最終的には、.xcodeproj ファイルが生成され、XCode に渡されます。

これを知っていると、パッケージングの部分はかなり制限されます: 自動化できる唯一のものは.xcodeprojの生成です。

まず、環境変数が正しく設定されていることを確認します。

![image](img/9.png)

scripts/package-ios.shを作成し、そこにこのスニペットを追加します。

```shell
#!/bin/bash

DIST_DIR=dist/mobile-ios
BUILD_DIR=build

mkdir -p $DIST_DIR && cd $DIST_DIR
mkdir -p $BIN_DIR $LIB_DIR $BUILD_DIR

pushd $BUILD_DIR
$QTDIR_IOS/bin/qmake \
    -spec macx-ios-clang \
    "CONFIG += release iphoneos device" \
    ../../../ch13-gallery-packaging.pro
make qmake_all
pushd gallery-core && make ; popd
pushd gallery-mobile && make ; popd

popd
```

スクリプトは以下の手順を実行します。

1. メインパスの変数を設定します。出力ディレクトリはdist_dirです。すべてのファイルは dist/mobile-ios/build フォルダに生成されます。
2. すべてのディレクトリを作成し、dist/mobile-ios/buildに移動します。
3. iPhoneデバイス（iPhoneシミュレータではなく）プラットフォームのリリースモードでqmakeを実行して、親プロジェクトのMakefileを生成します。
4. make qmake_all コマンドを実行して、サブプロジェクトの Makefile を生成します。
5. make コマンドを実行して、必要な各サブプロジェクトをビルドします。

このスクリプトを実行すると、dist/mobile-ios/build/gallerymobile/gallery-mobile.xcodeprojがXCodeで開く準備ができました。残りのステップは全てXCodeで行います。

1. XCodeでgallery-mobile.xcodeprojを開きます。
2. iOSデバイス用のアプリケーションをコンパイルします。
3. Appleの手順に従って、アプリケーションを配布します（App Storeで配布するか、スタンドアロンファイルとして配布します）。

その後、gallery-mobileはあなたのユーザーのために準備ができています!

***

## まとめ

アプリケーションがコンピュータ上で正常に動作していても、開発環境がこの動作に影響を与える可能性があります。ユーザーのハードウェア上でアプリケーションを実行するためには、そのパッケージが正しくなければなりません。アプリケーションをデプロイする前にパッケージ化するために必要な手順を学習しました。いくつかのプラットフォームでは、慎重に従わなければならない特定のタスクが必要でした。アプリケーションが独自のスクリプトを実行している場合、スタンドアロン パッケージをベイクできるようになりました。

次の章では、Qtを使ってアプリケーションを開発する際に役立つコツを紹介します。Qt Creatorに関するいくつかのヒントを学ぶことができます。

***

**[戻る](../index.html)**
